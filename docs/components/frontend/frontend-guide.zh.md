---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Frontend 指南
---

本指南介绍 Dynamo Frontend 的 KServe gRPC frontend 配置与集成。

## KServe gRPC Frontend

### 背景

[KServe v2 API](https://github.com/kserve/kserve/tree/master/docs/predict-api/v2) 是机器学习模型推理的工业级标准协议之一。Triton Inference Server 是兼容 KServe v2 API 的推理方案之一，并已被广泛采用。为了让 Triton 用户能快速尝试 Dynamo 带来的收益，Dynamo 提供了 KServe gRPC frontend。

本文档假设读者已熟悉 KServe v2 API 的用法，重点介绍在 Dynamo 中支撑 KServe API 的相关组件，以及如何把现有 KServe 部署迁移到 Dynamo。

## 支持的 endpoint

* `ModelInfer` endpoint：KServe 标准 endpoint，详见[此处](https://github.com/kserve/kserve/blob/master/docs/predict-api/v2/required_api.md#inference-1)
* `ModelStreamInfer` endpoint：Triton 扩展 endpoint，提供推理 RPC 的双向流式版本，可在一个 gRPC stream 上发送一系列推理请求/响应，详见[此处](https://github.com/triton-inference-server/common/blob/main/protobuf/grpc_service.proto#L84-L92)
* `ModelMetadata` endpoint：KServe 标准 endpoint，详见[此处](https://github.com/kserve/kserve/blob/master/docs/predict-api/v2/required_api.md#model-metadata-1)
* `ModelConfig` endpoint：Triton 扩展 endpoint，详见[此处](https://github.com/triton-inference-server/server/blob/main/docs/protocol/extension_model_configuration.md)

## 启动 frontend

启动 KServe frontend：

```bash
python -m dynamo.frontend --kserve-grpc-server
```

## gRPC 性能调优

gRPC server 通过环境变量支持可选的 HTTP/2 流控调优。可以在启动 server 之前设置，以适配高吞吐流式负载。

| 环境变量 | 说明 | 默认值 |
|---------------------|-------------|---------|
| `DYN_GRPC_INITIAL_CONNECTION_WINDOW_SIZE` | HTTP/2 连接级流控窗口大小（字节） | tonic 默认（64KB） |
| `DYN_GRPC_INITIAL_STREAM_WINDOW_SIZE` | HTTP/2 单 stream 流控窗口大小（字节） | tonic 默认（64KB） |

### 示例：面向流式负载的高 ISL/OSL 配置

```bash
# 用于 128 个并发的 15k token 请求
export DYN_GRPC_INITIAL_CONNECTION_WINDOW_SIZE=16777216  # 16MB
export DYN_GRPC_INITIAL_STREAM_WINDOW_SIZE=1048576      # 1MB
python -m dynamo.frontend --kserve-grpc-server
```

未设置这些变量时，server 使用 tonic 的默认值。

<Note>
请基于你的负载调优。连接窗口应能容纳 `concurrent_requests x request_size`。内存开销等于连接窗口大小（在所有 stream 间共享）。更多细节参见 [gRPC performance best practices](https://grpc.io/docs/guides/performance/) 与 [gRPC channel arguments](https://grpc.github.io/grpc/core/group__grpc__arg__keys.html)。
</Note>

## 注册一个 backend

与 HTTP frontend 类似，注册的 backend 会被自动发现并加入 frontend 的 serving 模型列表。注册时使用同样的 `register_model()` API。当前 frontend 支持以下 model type 与 model input 组合：

* `ModelType::Completions` + `ModelInput::Text`：使用自定义 preprocessor 的 LLM backend
* `ModelType::Completions` + `ModelInput::Token`：使用 Dynamo preprocessor 的 LLM backend（即 Dynamo SGLang / TRTLLM / vLLM backend）
* `ModelType::TensorBased` + `ModelInput::Tensor`：通用基于 tensor 的推理 backend

前两种组合走 OpenAI Completions API，详见 [OpenAI Completions](#openai-completions) 一节。最后一种则与 KServe API 最契合——只要 backend 实现了 `NvCreateTensorRequest/NvCreateTensorResponse` 的适配，用户就可以把已有部署替换为 Dynamo，详见 [Tensor](#tensor) 一节。

### OpenAI Completions

Dynamo 的大多数特性都是为 LLM 推理量身定制的，走 OpenAI API 的组合能启用这些特性，最适合用来探索 Dynamo 的能力。但这也意味着需要在通用 tensor 消息与 OpenAI 消息之间做特定转换，并对 KServe 请求消息的结构有特定要求。

#### Model Metadata / Config

metadata 与 config endpoint 会按下述结构上报已注册 backend——注意这并不是确切的响应：

```json
{
    "name": "$MODEL_NAME",
    "version": 1,
    "platform": "dynamo",
    "backend": "dynamo",
    "inputs": [
        {
            "name": "text_input",
            "datatype": "BYTES",
            "shape": [1]
        },
        {
            "name": "streaming",
            "datatype": "BOOL",
            "shape": [1],
            "optional": true
        }
    ],
    "outputs": [
        {
            "name": "text_output",
            "datatype": "BYTES",
            "shape": [-1]
        },
        {
            "name": "finish_reason",
            "datatype": "BYTES",
            "shape": [-1],
            "optional": true
        }
    ]
}
```

#### 推理

收到推理请求时会做如下转换：

* `text_input`：元素应为用户 prompt 字符串，会被转换为 OpenAI Completion 请求中的 `prompt` 字段
* `streaming`：会被转换为 OpenAI Completion 请求中的 `stream` 字段

收到模型响应时会做如下转换：

* `text_output`：每个元素对应 OpenAI Completion 响应里的一个 choice，内容写入该 choice 的 `text`
* `finish_reason`：每个元素对应 OpenAI Completion 响应里的一个 choice，内容写入该 choice 的 `finish_reason`

### Tensor

此组合用于把已有的 KServe-based backend 迁移到 Dynamo 生态。

#### Model Metadata / Config

注册 backend 时必须提供模型元数据，因为基于 tensor 的部署是通用的，frontend 无法像 OpenAI Completions 模型那样作出假设。提供模型元数据有两种方式：

* [TensorModelConfig](https://github.com/ai-dynamo/dynamo/tree/main/lib/llm/src/protocols/tensor.rs)：Dynamo 定义的元数据结构，backend 可以按此 [示例](https://github.com/ai-dynamo/dynamo/tree/main/lib/bindings/python/tests/test_tensor.py) 提供。通过这种方式提供时，下列字段会被设置为固定值：`version: 1`、`platform: "dynamo"`、`backend: "dynamo"`。注意：对于 model config endpoint，其余字段将取默认值。
* [triton_model_config](https://github.com/ai-dynamo/dynamo/tree/main/lib/llm/src/protocols/tensor.rs)：如果用户已有 Triton model config，且客户端逻辑需要原样返回完整 config，可以把它放在 `TensorModelConfig::triton_model_config` 中——它会覆盖 `TensorModelConfig` 中其他字段，并直接用于 endpoint 响应。`triton_model_config` 应是 `ModelConfig` protobuf 消息序列化后的字符串，参见 [echo_tensor_worker.py](https://github.com/ai-dynamo/dynamo/blob/main/tests/frontend/grpc/echo_tensor_worker.py)。

#### 推理

收到推理请求时，backend 会拿到 [NvCreateTensorRequest](https://github.com/ai-dynamo/dynamo/tree/main/lib/llm/src/protocols/tensor.rs)，并需要返回 [NvCreateTensorResponse](https://github.com/ai-dynamo/dynamo/tree/main/lib/llm/src/protocols/tensor.rs)。两者分别是 Dynamo 中 ModelInferRequest / ModelInferResponse protobuf 消息的映射。

## Python 绑定

frontend 也可以通过 Python binding 启动——当需要把 Dynamo 集成进既有系统、并希望 frontend 与其他组件跑在同一进程中时很有用。示例参见 [server.py](https://github.com/ai-dynamo/dynamo/tree/main/lib/bindings/python/examples/kserve_grpc_service/server.py)。

## 集成

### 与 Router

frontend 内置了一个 router 用于请求分发。配置路由模式：

```bash
python -m dynamo.frontend --router-mode kv --http-port 8000
```

路由配置详情参见 [Router 文档](../router/README.md)。

### 与 Backend

backend 调用 `register_model()` 后会自动向 frontend 注册。受支持的 backend：

- [vLLM Backend](../../backends/vllm/README.md)
- [SGLang Backend](../../backends/sglang/README.md)
- [TensorRT-LLM Backend](../../backends/trtllm/README.md)

## 另请参阅

| 文档 | 说明 |
|----------|-------------|
| [Frontend Overview](README.md) | 快速上手与特性矩阵 |
| [NVIDIA Request Extensions (`nvext`)](nvext.md) | 路由、预处理、响应元数据与 engine 优先级扩展 |
| [Router 文档](../router/README.md) | KV 感知路由的配置 |

