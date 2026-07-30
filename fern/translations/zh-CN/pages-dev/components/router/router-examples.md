---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 路由器示例
subtitle: KvRouter Python API、Kubernetes 部署和自定义路由模式的完整示例。
---

有关快速入门说明，请参阅[路由器 README](README.md)。本文档进一步介绍 Dynamo 路由器的使用示例，包括 Python API 用法、Kubernetes 部署和自定义路由模式。

## 使用 KvRouter Python API

除了通过命令行启动 KV 路由器外，还可以直接在 Python 中创建 `KvRouter` 对象。这样可以为每个请求单独覆盖路由配置。

> [!WARNING]
> **同一运行时中的多个路由器**：不要从同一个 `DistributedRuntime` 创建多个独立管理的 `KvRouter` 实例。由同一运行时所拥有端点创建的路由器会共享该运行时的主取消令牌，因此销毁一个路由器可能会取消其他路由器使用的后台任务。对于单个进程内前端，请使用一个 `KvRouter`；如果需要相互独立的路由器生命周期，请使用不同的前端进程，或通过不同的 `DistributedRuntime` 创建各个路由器。

当事件循环可通过 `loop` 使用时，相互独立的进程内路由器生命周期需要使用不同的运行时：

```python
router_a = KvRouter(DistributedRuntime(loop, "etcd", "tcp").endpoint("dynamo.backend.generate"), 16, KvRouterConfig())
router_b = KvRouter(DistributedRuntime(loop, "etcd", "tcp").endpoint("dynamo.backend.generate"), 16, KvRouterConfig())
```

### 方法

`KvRouter` 提供以下方法：

- **`generate(token_ids, model, ...)`**：路由并执行请求，返回异步响应流。它会自动处理工作器选择、状态跟踪和生命周期管理。

- **`best_worker(token_ids, router_config_override=None, request_id=None, update_indexer=False)`**：查询对于给定 token 将选择哪个工作器。返回 `(worker_id, dp_rank, overlap_blocks)`。
  - 不传 `request_id`：仅查询，不更新路由器状态
  - 传入 `request_id`：更新路由器生命周期状态以跟踪请求。**注意**：使用 `request_id` 时，必须在适当的生命周期节点调用 `mark_prefill_complete()` 和 `free()`，以确保负载跟踪准确
  - 传入 `update_indexer=True`：在近似索引器中记录选定的工作器，用于后续预测重叠情况。仅当 `use_kv_events=False` 时才有意义

- **`get_potential_loads(token_ids)`**：获取所有工作器的详细负载信息，包括潜在的预填充 token 数和活跃解码块数。返回负载字典列表。

- **`mark_prefill_complete(request_id)`**：表示请求已完成预填充阶段。仅在使用 `best_worker()` 代替 `generate()` 进行手动路由时，用于[手动管理生命周期](#2-手动管理状态高级)。

- **`free(request_id)`**：表示请求已完成，应释放其资源。仅在使用 `best_worker()` 代替 `generate()` 进行手动路由时，用于[手动管理生命周期](#2-手动管理状态高级)。

- **`dump_events()`**：将路由器索引器中的所有 KV 缓存事件转储为 JSON 字符串，可用于调试和分析。

### 准备工作

首先，启动后端引擎：

```bash
python -m dynamo.vllm --model meta-llama/Llama-2-7b-hf
```

### 示例脚本

```python
import asyncio
from dynamo.runtime import DistributedRuntime
from dynamo.llm import KvRouter, KvRouterConfig

async def main():
    # Get runtime and create endpoint
    loop = asyncio.get_running_loop()
    runtime = DistributedRuntime(loop, "etcd", "nats")
    endpoint = runtime.endpoint("dynamo.backend.generate")

    # Create KV router
    kv_router_config = KvRouterConfig()
    router = KvRouter(
        endpoint=endpoint,
        block_size=16,
        kv_router_config=kv_router_config
    )

    # Optional startup gate shared with the frontend and standalone indexer:
    # os.environ["DYN_ROUTER_MIN_INITIAL_WORKERS"] = "2"

    # Your input tokens
    token_ids = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

    # Generate with per-request routing override
    stream = await router.generate(
        token_ids=token_ids,
        model="meta-llama/Llama-2-7b-hf",
        stop_conditions={
            "max_tokens": 20,        # Generate exactly 20 tokens
            "ignore_eos": True,      # Don't stop at EOS token
        },
        sampling_options={
            "temperature": 0.7,
            "top_p": 0.9,
        },
        router_config_override={
            "overlap_score_credit": 1.0,    # Prioritize cache hits for this request
            "router_temperature": 0.5,       # Add routing randomness
        }
    )

    # Collect generated tokens
    generated_tokens = []
    async for response in stream:
        if isinstance(response, dict) and "token_ids" in response:
            generated_tokens.extend(response["token_ids"])

    print(f"Generated {len(generated_tokens)} tokens: {generated_tokens}")

if __name__ == "__main__":
    asyncio.run(main())
```

## Kubernetes 示例

有关使用 KV 路由器进行基本 Kubernetes 部署的说明，请参阅路由器指南中的 [Kubernetes 部署](../../../../../../docs/components/router/router-guide.md#kubernetes-deployment)部分。

### 完整的 Kubernetes 示例

- [TensorRT-LLM 聚合式路由器示例](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/trtllm/deploy/agg_router.yaml)
- [vLLM 聚合式路由器示例](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/deploy/agg_router.yaml)
- [SGLang 聚合式路由器示例](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/sglang/deploy/agg_router.yaml)
- [Kubernetes 部署指南](../../../../../../docs/kubernetes/README.md)

**有关 A/B 测试和高级 Kubernetes 设置：**
请参阅完整的 [KV 路由器 A/B 基准测试指南](../../../../../../docs/benchmarks/kv-router-ab-testing.md)，了解在 Kubernetes 中部署、配置 KV 路由器并进行基准测试的分步说明。

### 高级配置示例

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-deployment
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      envs:
        - name: DYN_ROUTER_MODE
          value: kv
        - name: DYN_ROUTER_TEMPERATURE
          value: "0.5"  # Add some randomness to prevent worker saturation
        - name: DYN_ROUTER_KV_OVERLAP_SCORE_CREDIT
          value: "1.0"  # Prefer device-local KV cache reuse
        - name: DYN_ROUTER_PREFILL_LOAD_SCALE
          value: "1.5"  # Prioritize TTFT over ITL
        - name: DYN_KV_CACHE_BLOCK_SIZE
          value: "16"
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.2.1
```

### 替代方案：在 Kubernetes 中使用命令行参数

还可以直接在容器命令中传递 CLI 参数：

```yaml
extraPodSpec:
  mainContainer:
    image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.2.1
    command:
      - /bin/sh
      - -c
    args:
      - "python3 -m dynamo.frontend --router-mode kv --router-temperature 0.5 --http-port 8000"
```

**建议：** 使用环境变量可以简化配置管理，并与 Dynamo 的 Kubernetes 模式保持一致。

## 路由模式

`KvRouter` 根据控制需求支持多种使用模式：

### 1. 自动路由（推荐）

直接调用 `generate()`，由路由器处理所有工作：

```python
stream = await router.generate(token_ids=tokens, model="model-name")
```

- **适用场景**：大多数使用场景
- **路由器自动执行**：选择最佳工作器、更新状态、路由请求并跟踪生命周期

### 2. 手动管理状态（高级）

使用 `best_worker(request_id=...)` 选择并跟踪工作器，然后自行管理请求：

```python
worker_id, _dp_rank, overlap = await router.best_worker(
    tokens,
    request_id="req-123",
    update_indexer=True,  # needed for approximate mode (use_kv_events=False)
)
response = await client.generate(tokens, request_id="req-123")
# await anext(response)  # Get first token
await router.mark_prefill_complete("req-123")  # After first token
# async for _ in response:  # Continue generating
#     ...
await router.free("req-123")  # After completion
```

- **适用场景**：需要跟踪路由器状态的自定义请求处理
- **要求**：在正确的生命周期节点调用 `mark_prefill_complete()` 和 `free()`
- **近似模式**：当 `use_kv_events=False` 时传入 `update_indexer=True`，使路由器能够从手动选择工作器的结果中学习
- **注意**：生命周期管理不正确会降低负载均衡的准确性

### 3. 分层路由器探测

在不更新状态的情况下进行查询，然后通过选定的路由器进行路由：

```python
# Probe multiple routers without updating state
worker_id_1, dp_rank, overlap_1 = await router_1.best_worker(tokens)  # No request_id
worker_id_2, dp_rank, overlap_2 = await router_2.best_worker(tokens)

# Pick the best router and corresponding worker based on results
if overlap_1 > overlap_2:
    chosen_router, chosen_worker = router_1, worker_id_1
else:
    chosen_router, chosen_worker = router_2, worker_id_2
stream = await chosen_router.generate(tokens, model="model-name", worker_id=chosen_worker)
```

- **适用场景**：多层部署（例如，由 Envoy Gateway 将请求路由到多个路由器组）
- **优势**：在选定路由器之前查询多个路由器

### 4. 自定义基于负载的路由

使用 `get_potential_loads()` 实现自定义路由逻辑：

```python
loads = await router.get_potential_loads(tokens)
# Apply custom logic (e.g., weighted scoring, constraints)
best_worker = min(loads, key=lambda x: custom_cost_fn(x))
stream = await router.generate(tokens, model="model-name", worker_id=best_worker['worker_id'])
```

- **适用场景**：超出内置成本函数能力的自定义优化策略
- **优势**：完全控制工作器选择逻辑
- **另请参阅**：下文的“自定义路由示例：最小化 TTFT”

所有模式都支持 `router_config_override`，无需重新创建路由器即可调整每个请求的路由行为。

## 自定义路由示例：最小化 TTFT

以下示例使用 `get_potential_loads()` 实现自定义路由。它选择预填充工作量最少的工作器，从而缩短首个 Token 生成时间（Time To First Token，TTFT）：

```python
import asyncio
from dynamo.runtime import DistributedRuntime
from dynamo.llm import KvRouter, KvRouterConfig

async def minimize_ttft_routing():
    # Setup router
    loop = asyncio.get_running_loop()
    runtime = DistributedRuntime(loop, "etcd", "nats")
    endpoint = runtime.endpoint("dynamo.backend.generate")

    router = KvRouter(
        endpoint=endpoint,
        block_size=16,
        kv_router_config=KvRouterConfig()
    )

    # Your input tokens
    token_ids = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

    # Get potential loads for all workers
    potential_loads = await router.get_potential_loads(token_ids)

    # Find worker with minimum prefill tokens (best for TTFT)
    best_worker = min(potential_loads, key=lambda x: x['potential_prefill_tokens'])

    print(f"Worker loads: {potential_loads}")
    print(f"Selected worker {best_worker['worker_id']} with {best_worker['potential_prefill_tokens']} prefill tokens")

    # Route directly to the selected worker
    stream = await router.generate(
        token_ids=token_ids,
        model="meta-llama/Llama-2-7b-hf",
        worker_id=best_worker['worker_id'],  # Force routing to optimal worker
        stop_conditions={"max_tokens": 20}
    )

    # Process response
    async for response in stream:
        if isinstance(response, dict) and "token_ids" in response:
            print(f"Generated tokens: {response['token_ids']}")

if __name__ == "__main__":
    asyncio.run(minimize_ttft_routing())
```

这种方法可以完全控制路由决策，并根据具体需求优化不同指标。例如：

- **最小化 TTFT**：选择 `potential_prefill_tokens` 最小的工作器
- **最大化缓存复用**：使用同时考虑预填充和解码负载的 `best_worker()`
- **均衡负载**：同时考虑 `potential_prefill_tokens` 和 `potential_decode_blocks`

有关架构详情和成本函数算法，请参阅[路由器设计](../../../../../../docs/design-docs/router-design.md)。

## 为自定义引擎发布 KV 事件

有关为自定义推理引擎实现 KV 事件发布的完整说明，请参阅专门的[为自定义引擎发布 KV 事件](../../../../../../docs/integrations/kv-events-custom-engines.md)指南。其中包括：

- **直接发布**：调用 `publish_stored()` / `publish_removed()`，通过 Dynamo 事件平面推送事件
- **ZMQ 中继**：对于通过 ZMQ 发出原始 KV 事件的引擎（例如 SGLang 和 vLLM），同一个 `KvEventPublisher` 会订阅 ZMQ 套接字并自动中继事件
- API 参考、事件结构、ZMQ 传输格式和最佳实践

## 全局路由器（分层路由）

对于包含多个工作器池的部署，**全局路由器**位于前端与本地路由器之间，以提供分层路由。它根据可配置的策略为每个请求选择合适的工作器池，并支持为不同工作负载特征调优工作器池的分离式拓扑。

- **组件详情**：[`components/src/dynamo/global_router/`](https://github.com/ai-dynamo/dynamo/tree/main/components/src/dynamo/global_router/)
- **示例**：[`examples/global_planner/`](https://github.com/ai-dynamo/dynamo/tree/main/examples/global_planner/)

## 另请参阅

- **[路由器 README](README.md)**：KV 路由器快速入门指南
- **[配置和调优](../../../../../../docs/components/router/router-configuration.md)**：路由器参数和生产环境设置
- **[路由器设计](../../../../../../docs/design-docs/router-design.md)**：架构详情和事件传输模式

