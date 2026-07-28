---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 事件平面（Event Plane）
---

事件平面（Event Plane）为 Dynamo 提供组件之间近实时事件交换的发布/订阅层。它负责投递 KV 缓存更新、Worker 负载指标和序列追踪事件，从而支撑 KV 感知路由（KV-aware routing）和分离式服务（disaggregated serving）等能力。

## 什么时候用到事件平面？

主要使用场景：

- **KV 缓存事件** —— Worker 发布缓存状态，Router 据此做缓存感知（cache-aware）调度。
- **Worker 负载指标** —— Worker 上报使用率，Router 据此做负载均衡。
- **序列追踪** —— 在多个 Router 副本之间协调活跃序列，实现容错路由。

![事件平面架构：NATS 与 ZMQ 两种传输方式连接 Frontend、Planner 和 Worker](../assets/img/event-plane-transport.svg)

## 选择传输方式

事件平面支持两种传输方式：

|  | NATS（默认） | ZMQ |
|---|---|---|
| **外部基础设施** | 需要一个 NATS Server | 不需要（点对点） |
| **部署复杂度** | 简单 —— 指向 NATS Server 即可 | 自动 —— Worker 绑定 socket 并通过服务发现注册 |
| **适用场景** | 大规模部署 | 运维成本低的场景 |

## 配置

### 选择传输方式

通过环境变量 `DYN_EVENT_PLANE` 选择传输方式：

```bash
# 使用 NATS（默认值，可以不显式设置）
export DYN_EVENT_PLANE=nats

# 使用 ZMQ
export DYN_EVENT_PLANE=zmq
```

Python 组件也支持通过 CLI 参数指定：

```bash
# SGLang 后端
python3 -m dynamo.sglang --event-plane zmq --model Qwen/Qwen3-0.6B

# vLLM 后端
python3 -m dynamo.vllm --event-plane zmq --model Qwen/Qwen3-0.6B
```

### 环境变量

| 变量 | 说明 | 默认值 |
|----------|-------------|---------|
| `DYN_EVENT_PLANE` | 传输方式：`nats` 或 `zmq` | 依赖上下文（见下文） |
| `NATS_SERVER` | NATS Server 地址（仅 NATS 传输需要） | `nats://localhost:4222` |

当未显式设置 `DYN_EVENT_PLANE` 时，默认值根据 discovery backend 自动决定：

- `--discovery-backend file` 或 `mem`（本地后端）：默认 **zmq**，无需外部服务。
- `--discovery-backend etcd` 或 `kubernetes`（分布式后端）：默认 **nats**。

显式设置 `DYN_EVENT_PLANE` 可覆盖此自动选择。

## NATS 传输

使用 NATS 时（`DYN_EVENT_PLANE=nats`，或在分布式后端下未显式设置）：

- 需要一个运行中的 NATS Server。如果不在 `localhost:4222`，需设置 `NATS_SERVER`。
- 事件按 namespace 和 component 维度发布到 NATS subject。
- 内置短暂断连期间的重连和消息缓冲机制。

示例：

```bash
export NATS_SERVER=nats://nats-server:4222
export DYN_EVENT_PLANE=nats

# 启动 Worker —— 显式开启 KV 事件发布
python3 -m dynamo.vllm --model Qwen/Qwen3-0.6B \
    --kv-events-config '{"publisher":"nats","topic":"kv-events","enable_kv_cache_events":true}'

# 启动 Frontend —— 自动从 NATS 订阅事件
python3 -m dynamo.frontend --router-mode kv
```

## ZMQ 传输

使用 ZMQ 时（`DYN_EVENT_PLANE=zmq`）：

- 不需要外部 server。每个 Worker 绑定一个 ZMQ PUB socket，并通过服务发现广播自己的地址。
- 订阅方自动发现并连接所有活跃的发布方。
- 发布方动态增减时（如 Worker 扩缩容），订阅方会动态调整连接。

示例：

```bash
export DYN_EVENT_PLANE=zmq

# 启动 Worker —— 各自绑定 ZMQ socket，并注册到服务发现
python3 -m dynamo.vllm --model Qwen/Qwen3-0.6B \
  --kv-events-config '{"publisher":"zmq","endpoint":"tcp://*:20080","enable_kv_cache_events":true}'

# 启动 Frontend —— 自动发现 Worker 并直连
python3 -m dynamo.frontend --router-mode kv
```

## 禁用事件平面

如果不需要 KV 感知路由，可以完全禁用事件平面：

```bash
python3 -m dynamo.frontend --router-mode kv --no-router-kv-events
```

启用 `--no-router-kv-events` 后：

- Router 退回到基于预测的缓存感知路由（通过路由决策估算缓存状态）。
- 不需要 NATS Server 或 ZMQ socket。
- 通过 TTL 过期机制防止预测状态长期失真。

## 部署模式

### 裸机 / 本地

两种传输方式开箱即用：

```bash
# NATS（需先启动 nats-server）
export NATS_SERVER=nats://localhost:4222

# 或者 ZMQ（无需额外基础设施）
export DYN_EVENT_PLANE=zmq
```

### Kubernetes（搭配 Dynamo Operator）

Operator 可向 Pod 注入 `DYN_EVENT_PLANE`。两种传输方式都可用。若使用 NATS，需在集群内部署一个 NATS Server，并相应设置 `NATS_SERVER`。

## 相关文档

- [发现平面（Discovery Plane）](discovery-plane.md) —— 服务发现与协调（etcd、Kubernetes）
- [分布式运行时（Distributed Runtime）](distributed-runtime.md) —— 运行时架构
- [请求平面（Request Plane）](request-plane.md) —— 请求传输配置
- [容错（Fault Tolerance）](../fault-tolerance/README.md) —— 故障处理

