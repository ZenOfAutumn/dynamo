---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Discovery Plane（服务发现平面）
---

Dynamo 的服务发现层让各组件在运行时互相找到对方。Worker 在启动时注册自己的 endpoint，frontend 则会自动发现它们。
discovery backend 会根据部署环境自适应。

![Discovery plane 架构示意：Kubernetes 与 etcd 两种 backend](../assets/img/discovery-plane.svg)

## Discovery Backend

| 部署形态 | Discovery Backend | 配置方式 |
|------------|-------------------|---------------|
| **Kubernetes**（搭配 Dynamo operator） | 原生 K8s（CRD、EndpointSlices） | operator 设置 `DYN_DISCOVERY_BACKEND=kubernetes` |
| **裸金属 / 本地**（默认） | etcd | `ETCD_ENDPOINTS`（默认为 `http://localhost:2379`） |

> **注意：** runtime 默认始终使用 etcd。Kubernetes discovery 必须显式启用 —— Dynamo operator 会自动完成这件事。

## Kubernetes 服务发现

在搭配 Dynamo operator 的 Kubernetes 上运行时，服务发现使用 Kubernetes 原生资源，而不是 etcd。

### 工作原理

1. Worker 通过创建 **DynamoWorkerMetadata** 自定义资源来注册自身的 endpoint。
2. **EndpointSlices** 向系统通报 pod 的 ready 状态。
3. 各组件监听 CRD 变更以发现可用的 worker。

### 优势

- 无需外部 etcd 集群。
- 与 Kubernetes pod 生命周期原生集成。
- pod 终止时自动清理。
- 与标准 Kubernetes RBAC 协作。

### 环境变量（由 operator 注入）

| 变量 | 说明 |
|----------|-------------|
| `DYN_DISCOVERY_BACKEND` | 固定为 `kubernetes` |
| `POD_NAME` | 当前 pod 名称 |
| `POD_NAMESPACE` | 当前 namespace |
| `POD_UID` | pod 唯一标识 |

## etcd 服务发现（默认）

当未设置 `DYN_DISCOVERY_BACKEND`（或设为 `etcd`）时，使用 etcd 进行服务发现。

### 连接配置

| 变量 | 说明 | 默认值 |
|----------|-------------|---------|
| `ETCD_ENDPOINTS` | 逗号分隔的 etcd URL 列表 | `http://localhost:2379` |
| `ETCD_AUTH_USERNAME` | Basic Auth 用户名 | 无 |
| `ETCD_AUTH_PASSWORD` | Basic Auth 密码 | 无 |
| `ETCD_AUTH_CA` | CA 证书路径（TLS） | 无 |
| `ETCD_AUTH_CLIENT_CERT` | 客户端证书路径 | 无 |
| `ETCD_AUTH_CLIENT_KEY` | 客户端私钥路径 | 无 |

示例：

```bash
export ETCD_ENDPOINTS=http://etcd-0:2379,http://etcd-1:2379,http://etcd-2:2379
```

### 服务注册

Worker 在 etcd 中按如下 key 层级注册 endpoint：

```
/services/{namespace}/{component}/{endpoint}/{instance_id}
```

例如：

```
/services/vllm-agg/backend/generate/694d98147d54be25
```

Frontend 与 router 通过监听相应的 key 前缀来发现可用 worker，并在 worker 加入或离开时实时收到更新。

### 基于 Lease 的清理

每个 runtime 都会与 etcd 维持一个 lease（默认 TTL：10 秒）。当 worker 崩溃或失联时：

![Lease 生命周期：DistributedRuntime 与 etcd 之间的 keep-alive 心跳](../assets/img/discovery-plane-lease.svg)

1. keep-alive 心跳停止。
2. lease 在 TTL 后过期。
3. 所有已注册的 endpoint 被自动删除。
4. 客户端收到移除事件，并把流量重新路由到健康的 worker。

这确保了过期的 endpoint 无需人工介入即可清理。

## KV Store

Dynamo 提供了一层 KV store 抽象用于存储元数据（endpoint 实例、model deployment card、event channel 等）。支持多种 backend：

| Backend | 适用场景 |
|---------|----------|
| etcd | 生产部署 |
| Memory | 测试与开发 |
| NATS | 仅依赖 NATS 的部署 |
| File | 本地持久化 |

## 运维建议

### 在 K8s 上使用 Kubernetes 服务发现

Dynamo operator 会自动为 pod 设置 `DYN_DISCOVERY_BACKEND=kubernetes`，无需任何额外配置。

### 裸金属环境部署 etcd 集群

对于裸金属生产部署，建议部署一个 3 节点的 etcd 集群以保证高可用。

### 调整 Lease TTL

在故障检测速度与开销之间做权衡：

- **较短 TTL（5s）** —— 故障检测更快，但 keep-alive 流量更多。
- **较长 TTL（30s）** —— 开销更小，但检测更慢。

默认值（10s）对大多数部署是一个合理的起点。

## 相关文档

- [Event Plane](event-plane.md) —— 用于 KV cache 事件与 worker 指标的发布订阅
- [Distributed Runtime](distributed-runtime.md) —— runtime 架构
- [Request Plane](request-plane.md) —— 请求传输配置
- [Fault Tolerance](../fault-tolerance/README.md) —— 故障处理

