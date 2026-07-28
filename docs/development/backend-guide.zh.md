---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 在 Dynamo 中编写 Python Worker
sidebar-title: 编写 Python Worker
subtitle: 为 Dynamo 创建自定义 Python worker 与 engine
---

# 在 Dynamo 中编写 Python Worker

本指南介绍如何在 Dynamo 中创建你自己的 Python worker。

[dynamo](https://pypi.org/project/ai-dynamo/) Python 库允许你构建自己的 engine 并将其接入 Dynamo。

Python 文件需要做三件事：
1. 用装饰器获取 runtime
2. 在网络中注册自身
3. 挂载请求处理器（request handler）

```
from dynamo.llm import ModelInput, ModelType, register_model
from dynamo.runtime import DistributedRuntime, dynamo_worker

   # 1. 用装饰器获取 runtime
   #
   @dynamo_worker()
   async def worker(runtime: DistributedRuntime):

    # 2. 在网络中注册自身
    #
    endpoint = runtime.endpoint("namespace.component.endpoint")
    model_path = "Qwen/Qwen3-0.6B" # 或 "/data/models/Qwen3-0.6B"
    model_input = ModelInput.Tokens # 若 engine 自行处理预处理，则使用 ModelInput.Text
    model_type = ModelType.Chat # 或 ModelType.Chat | ModelType.Completions（若模型可同时部署在 chat 与 completions endpoint 上）
    # register_model 的最后一个可选参数是 model_name；若未提供，则从 model_path 推导得出
    await register_model(model_input, model_type, endpoint, model_path)

    # 在此初始化你的 engine
    # engine = ...

    # 3. 挂载请求处理器
    #
    await endpoint.serve_endpoint(RequestHandler(engine).generate)

class RequestHandler:

    def __init__(self, engine):
        ...

    async def generate(self, request):
        # 调用 engine
        # yield 结果 dict
        ...

if __name__ == "__main__":
    uvloop.install()
    asyncio.run(worker())
```


`model_path` 可以是：
- 一个 HuggingFace repo ID，可以前缀 `hf://`，会被下载并缓存到本地。
- 已检出的 HuggingFace repo 路径——任意包含 safetensor 文件以及 `config.json`、`tokenizer.json`、`tokenizer_config.json` 的目录。

`model_input` 可以是：
- ModelInput.Tokens：你的 engine 期望接收已预处理的输入（token ID）。Dynamo 负责 tokenization 与预处理。
- ModelInput.Text：你的 engine 期望接收原始文本输入，并自行处理 tokenization 与预处理。

`model_type` 可以是：
- ModelType.Chat：你的 `generate` 方法接收一个 `request`，并必须返回 [OpenAI Chat Completion](https://platform.openai.com/docs/api-reference/chat/completions) 类型的响应 dict。
- ModelType.Completions：你的 `generate` 方法接收一个 `request`，并必须返回更早的 [Completions](https://platform.openai.com/docs/api-reference/completions) 响应 dict。

`register_model` 还可以接收以下 kwargs：
- `model_name`：模型对外的名称。HTTP 请求中的 model 名必须与之匹配。默认取 HuggingFace repo 名或文件夹名。
- `context_length`：模型最大长度（token 数）。默认采用模型自身设定的最大值。仅在需要减少 KV cache 分配以适配 VRAM 时才设置。
- `kv_cache_block_size`：engine 的 KV block 大小（token 数）。默认 16。
- `user_data`：可选的 dict，用于配置 worker 行为的自定义元数据（例如 LoRA 配置）。默认 None。

完整代码示例请参见 `examples/backends`。

## 组件命名

worker 注册自身时需要三个名字：namespace.component.endpoint

* *Namespace*：一条 pipeline，通常对应一个模型。例如 "llama_8b"。仅是一个名字。
* *Component*：运行该 pipeline 所需的、负载均衡的服务，如 "backend"、"prefill"、"decode"、"preprocessor"、"draft" 等。它通常带有一些配置（例如使用哪个模型）。
* *Endpoint*：类似 URL，例如 "generate"、"load_metrics"。
* *Instance*：一个进程。唯一的。Dynamo 会为每个 instance 分配唯一的 instance_id。真正在运行的总是 instance；而 namespace/component/endpoint 可能对应多个 instance。

如果你运行两个模型，那就是两条 pipeline。投机解码（speculative decoding）是个例外——draft 模型属于更大模型 pipeline 的一部分。

如果你运行同一模型的两个实例（"data parallel"），它们的 namespace+component+endpoint 相同，但 instance 不同。Router 会把流量分散到一个 namespace+component+endpoint 下的所有 instance。如果一条 pipeline 中有四个 prefill worker，它们的 namespace+component+endpoint 完全相同，并会自动分配到不同的 instance_id。

示例 1：数据并行的负载均衡，一个模型一条 pipeline 两个 instance。
```
Node 1: namespace: qwen3-32b, component: backend, endpoint: generate, model: /data/Qwen3-32B --tensor-parallel-size 2 --base-gpu-id 0
Node 2: namespace: qwen3-32b, component: backend, endpoint: generate model: /data/Qwen3-32B --tensor-parallel-size 2 --base-gpu-id 2
```

示例 2：两个模型，两条 pipeline。
```
Node 1: namespace: qwen3-32b, component: backend, endpoint: generate, model: /data/Qwen3-32B
Node 2: namespace: llama3-1-8b, component: backend, endpoint: generat, model: /data/Llama-3.1-8B-Instruct/
```

示例 3：不同的 endpoint。

VLLM 中的 KV metrics publisher 会向当前 component 添加一个 `load_metrics` endpoint。如果上文的 `llama3-1-8b.backend` 使用了打过补丁的 vllm，那它也会暴露 `llama3-1-8b.backend.load_metrics`。

示例 4：一条 pipeline 中的多个 component。

在 P/D 解耦部署中，你会同时拥有 `deepseek-distill-llama8b.prefill.generate`（可能有多个实例）和 `deepseek-distill-llama8b.decode.generate`。

## 迁移进行中的请求

Python worker 可能需要被迅速关闭，例如运行该 worker 的节点要被回收，且没有足够的时间在关闭截止时间前完成所有进行中的请求。

这种情况下，你可以在 generate 循环中抛出 `EngineShutdown` 异常，以发出"响应不完整"的信号。这会立刻关闭响应流，告知 frontend 该流不完整。开启请求迁移后（参见 [`migration_limit`](../fault-tolerance/request-migration.md) 参数），frontend 会自动把已部分完成的请求迁移到另一个可用的 worker 实例继续完成。

下面是在 `RequestHandler` 中实现该机制的示例：

```python
from dynamo.llm.exceptions import EngineShutdown

class RequestHandler:

    async def generate(self, request):
        """生成响应，支持请求迁移"""
        for result in self.engine.generate_streaming(request):
            # 在 yield 每个 token 前检查是否需要迁移
            if is_shutting_down():
                # 抛出 EngineShutdown 会关闭流并触发迁移
                raise EngineShutdown("Worker shutting down, migrating request")

            yield result
```

抛出 `EngineShutdown` 后，frontend 会收到不完整的响应，并能在另一个可用的 worker 实例上无缝续写，从而即使在 worker 关闭过程中也能保持用户体验。

更多关于请求迁移工作原理的信息，请参阅 [Request Migration Architecture](../fault-tolerance/request-migration.md) 文档。

## 请求取消

你的 Python worker 的 request handler 可以在 `request` 参数后再接收一个 `context` 参数，用于支持请求取消。该 context 对象允许你检查取消信号并做出相应处理：

```python
class RequestHandler:

    async def generate(self, request, context):
        """生成响应，支持取消"""
        for result in self.engine.generate_streaming(request):
            # 检查请求是否已被取消
            if context.is_stopped():
                # 停止处理并清理
                break

            yield result
```

context 参数是可选的——如果你的 generate 方法签名中没有它，Dynamo 调用时就不会传递 context 参数。

关于请求取消的详细信息（包括异步取消监听与 context 传播模式），请参阅 [Request Cancellation Architecture](../fault-tolerance/request-cancellation.md) 文档。

