---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 分布式运行时（Distributed Runtime）
---

## 概述

Dynamo 的 `DistributedRuntime` 是框架中实现各组件分布式通信与协调的核心基础设施。它使用 Rust 实现（`/lib/runtime`），并通过绑定暴露给其他编程语言（例如 Python 绑定位于 `/lib/bindings/python`）。该运行时支持多种发现后端（Kubernetes 原生或 etcd）和请求平面（TCP、HTTP 或 NATS）。`DistributedRuntime` 采用层级化结构：

- `DistributedRuntime`：暴露分布式运行时接口的最顶层对象。它管理与发现后端（K8s API 或 etcd）以及可选消息队列（用于 KV 事件的 NATS）的连接，并通过 cancellation token 处理生命周期。
- `Namespace`：`Namespace` 是组件的逻辑分组，用于在不同模型部署之间隔离。
- `Component`：`Component` 是 `Namespace` 内可发现的对象，代表逻辑上的一组 worker。
- `Endpoint`：`Endpoint` 是一个网络可访问的服务，提供具体功能。

理论上每个 `DistributedRuntime` 可以包含多个 `Namespace`（只要名称唯一；同样的逻辑也适用于 `Component/Namespace` 与 `Endpoint/Component` 的关系），但实践中，每个 dynamo 组件通常以独立的进程部署，因此各自拥有自己的 `DistributedRuntime` 对象。然而，它们共享同一 namespace 以相互发现。

例如，一个典型部署配置（如 `examples/backends/vllm/deploy/agg.yaml` 或 `examples/backends/sglang/deploy/agg.yaml`）包含多个组件：

- `Frontend`：启动 HTTP 服务器（在端口 8000 上提供 OpenAI 兼容 API），处理入站请求、应用 chat template、执行 tokenization，并将请求路由到 worker。`make_engine` 函数封装了这些功能。
- `Worker` 组件（例如 `VllmDecodeWorker`、`VllmPrefillWorker`、`SGLangDecodeWorker`、`TRTLLMWorker`）：使用各自的引擎（SGLang、TensorRT-LLM、vLLM）执行实际推理计算。

由于这些组件部署在不同进程中，每个都有自己的 `DistributedRuntime`。在各自的 `DistributedRuntime` 中，它们共享相同的 `Namespace`（如 `vllm-agg`、`sglang-disagg`）。在它们的 namespace 下，每个组件都拥有自己的 `Component`：

- `Frontend` 使用 `make_engine` 函数，自动处理 HTTP 服务、请求预处理与 worker 发现
- Worker 组件按角色注册名称，例如 `backend`、`prefill`、`decode` 或 `encoder`
- Worker 注册诸如 `generate`、`clear_kv_blocks` 或 `load_metrics` 等端点

它们的 `DistributedRuntime` 在各自的 main 函数中初始化，`Namespace` 在部署 YAML 中配置，`Endpoint` 通过路径获取。在 Python 中，使用 `runtime.endpoint("namespace.component.endpoint")`（例如 `runtime.endpoint("dynamo.backend.generate")`）。

## 初始化

本节解释当 `DistributedRuntime/Namespace/Component/Endpoint` 对象创建时底层发生了什么。`DistributedRuntime` 的初始化基于部署环境有多种模式。

```{caution}
该层级与命名可能随时间变化，本文档可能未必反映最新更改。无论如何变化，主要概念将保持不变。
```

### 服务发现后端

`DistributedRuntime` 支持两种服务发现后端，通过 `DYN_DISCOVERY_BACKEND` 配置：

- **KV Store 发现**（`DYN_DISCOVERY_BACKEND=etcd`）：使用 etcd 进行服务发现。**这是默认值**，除非显式覆盖。其他 KV store 后端（`file`、`mem`）也可用。

- **Kubernetes 发现**（`DYN_DISCOVERY_BACKEND=kubernetes`）：使用原生 Kubernetes 资源（DynamoWorkerMetadata CRD、EndpointSlices）进行服务发现。**必须显式设置。** Dynamo operator 自动将该环境变量注入 Kubernetes 部署的 Pod。**无需 etcd。**

> **注意：** 不会自动检测部署环境。运行时默认使用 `etcd`。在 Kubernetes 部署中，operator 会向 Pod 环境注入 `DYN_DISCOVERY_BACKEND=kubernetes`。

使用 Kubernetes 发现时，KV store 后端会自动切换到内存存储，因为不再需要 etcd。

### 运行时初始化

- `DistributedRuntime`：当一个 `DistributedRuntime` 对象创建时，根据发现后端建立连接：
    - **Kubernetes 模式**：通过 DynamoWorkerMetadata CRD 使用 K8s API 进行服务注册。无需外部依赖。
    - **KV Store 模式**：连接到 etcd 进行服务发现。创建一个带有后台 keep-alive 任务的主 lease。在该 `DistributedRuntime` 下注册的所有对象均使用此 lease_id 维持其生命周期。
    - **NATS**（可选）：在使用 KV 感知路由时用于 KV 事件消息。可通过 `--no-router-kv-events` 关闭，从而启用基于预测的路由而不持久化事件。
    - **请求平面**：默认 TCP。可通过 `DYN_REQUEST_PLANE` 环境变量配置为 HTTP 或 NATS。
- `Namespace`：`Namespace` 主要是逻辑分组机制。它为该 `Namespace` 下的所有组件提供根路径。
- `Component`：当 `Component` 对象创建时，会在 `DistributedRuntime` 的内部 registry 中注册一个服务，registry 跟踪所有服务和端点。
- `Endpoint`：当 Endpoint 对象创建并启动时，根据发现后端进行注册：
  - **Kubernetes 模式**：端点信息存储在 DynamoWorkerMetadata CRD 资源中，由其他组件 watch 以发现。
  - **KV Store 模式**：端点信息按以下命名规则存储在 etcd 中：`/services/{namespace}/{component}/{endpoint}-{lease_id}`。请注意，同类型不同 worker 的端点（例如同一部署中的两个 `VllmPrefillWorker`）共享同一 `Namespace`、`Component` 与 `Endpoint` 名称，由各自不同的主 `lease_id` 区分。

## 调用端点

Dynamo 使用 `Client` 对象调用端点。创建 `Client` 时，需要给定 `Namespace`、`Component` 与 `Endpoint` 的名称。它会监听端点变更：

- **Kubernetes 模式**：watch DynamoWorkerMetadata CRD 资源以获取端点更新。
- **KV Store 模式**：设置一个 etcd watcher，监控前缀 `/services/{namespace}/{component}/{endpoint}`。

watcher 持续向 `Client` 更新可用 `Endpoint` 的信息。

用户可决定从 `Client` 调用 `Endpoint` 时所采用的负载均衡策略，相关代码位于 [push_router.rs](https://github.com/ai-dynamo/dynamo/tree/main/lib/runtime/src/pipeline/network/egress/push_router.rs)。Dynamo 支持三种负载均衡策略：

- `random`：随机选择一个端点
- `round_robin`：按轮询顺序选择端点
- `direct`：通过指定 instance ID 将请求直接发送到特定端点

选定要访问的端点后，`Client` 使用所配置的请求平面（默认 TCP）发送请求。请求平面处理实际传输：

- **TCP**（默认）：直接 TCP 连接，带连接池
- **HTTP**：基于 HTTP/2 的传输
- **NATS**：基于消息中间件的传输（遗留）

## 示例

我们提供了 `DistributedRuntime` 基本用法的原生 rust 与 python（通过绑定）示例：

- Rust：`/lib/runtime/examples/`
- Python：我们也提供了完整的 `DistributedRuntime` 使用示例。完整实现细节请参考 `components/src/dynamo` 中的引擎。

