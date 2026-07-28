---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 服务发现
---

Dynamo 组件（前端、worker、planner）需要在运行时相互发现并感知彼此的能力。我们将其称为服务发现。Kubernetes 上支持两种服务发现后端。

## 发现后端

| 后端 | 默认 | 依赖 | 用途 |
|---------|---------|--------------|----------|
| **Kubernetes** | ✅ 是 | 无（原生 K8s） | 推荐用于所有 Kubernetes 部署 |
| **KV Store（etcd）** | 否 | etcd 集群 | 遗留部署 |

## Kubernetes 发现（默认）

在 Kubernetes 上运行时，Kubernetes 发现是默认且推荐的后端。它使用原生 Kubernetes 原语来辅助组件发现：

- **DynamoWorkerMetadata CRD**：每个 worker 将自身已注册的端点和模型卡保存在自定义资源中
- **EndpointSlices**：EndpointSlices 表示每个组件的就绪状态

### 实现细节

每个 Pod 运行一个**发现守护进程**，监视 EndpointSlices 与 DynamoWorkerMetadata CR 两者。只有当 Pod 在 EndpointSlice 中显示为 "ready" 且具有对应的 `DynamoWorkerMetadata` CR 时，它才可被发现。这一关联关系确保了 Pod 在就绪之前不会被发现，元数据立即可用，且 Pod 终止时陈旧条目能够被清理。

#### DynamoWorkerMetadata CRD

每个 worker Pod 创建一个 `DynamoWorkerMetadata` CR，存储其发现元数据：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoWorkerMetadata
metadata:
  name: my-worker-pod-abc123
  namespace: dynamo-system
  ownerReferences:
    - apiVersion: v1
      kind: Pod
      name: my-worker-pod-abc123
      uid: <pod-uid>
      controller: true
spec:
  data:
    endpoints:
      "dynamo/backend/generate":
        type: Endpoint
        namespace: dynamo
        component: backend
        endpoint: generate
        instance_id: 12345678901234567890
        transport:
          nats_tcp: "dynamo_backend.generate-abc123"
    model_cards: {}
```

CR 与 Pod 同名，并包含一个 owner reference，以便在 Pod 被删除时自动垃圾回收。

#### EndpointSlices

DynamoWorkerMetadata 资源提供组件能力的最新快照，而 EndpointSlices 给出 Dynamo 各组件健康状态的快照。

operator 会创建一个面向 Dynamo 组件的 Kubernetes Service。Kubernetes 控制器随后创建并维护 EndpointSlice 资源，跟踪该 Service 所选 Pod 的就绪状态。监听这些 slice 让我们获得哪些 Dynamo 组件已就绪可服务流量的最新快照。

##### 就绪探针
当就绪探针成功时 Pod 被标记为 ready。在 Dynamo worker 上，这意味着 `generate` 端点可用且健康。这些探针由 Dynamo operator 为每个 Pod / 组件配置。

#### RBAC

每个 Dynamo 组件 Pod 都会自动获得一个 ServiceAccount，允许其在所属 namespace 中监听 `EndpointSlice` 与 `DynamoWorkerMetadata` 资源。

#### 环境变量

operator 会自动将以下环境变量注入 Pod 以辅助服务发现：

| 变量 | 描述 |
|----------|-------------|
| `DYN_DISCOVERY_BACKEND` | 设为 `kubernetes` |
| `POD_NAME` | Pod 名称（通过 downward API） |
| `POD_NAMESPACE` | Pod 命名空间（通过 downward API） |
| `POD_UID` | Pod UID（通过 downward API） |

Pod 的 instance ID 通过对 Pod 名称做哈希确定性生成，确保身份一致以及 EndpointSlices 与 CR 之间的关联。

## KV Store 发现（etcd）

要使用基于 etcd 的发现而不是 Kubernetes 原生发现，请向 DynamoGraphDeployment 添加注解：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-deployment
  annotations:
    nvidia.com/dynamo-discovery-backend: etcd
spec:
  services:
    # ...
```

这需要可用的 etcd 集群。etcd 连接通过平台 Helm chart 配置。
