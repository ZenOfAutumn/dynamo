---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Rolling Updates
---

本指南介绍 `DynamoGraphDeployment`（DGD）资源的滚动更新（rolling update）工作机制。滚动更新允许你以最小停机时间更新 worker 配置（镜像、资源、环境变量等），通过逐步以新 pod 替换旧 pod 来完成更新。

滚动更新的行为取决于部署的底层资源类型。由 Kubernetes Deployment 支撑的 DGD 受益于 **托管滚动更新**（带命名空间隔离），而由 Grove 与 LWS 支撑的部署使用其原生更新机制。

## 示例

考虑一个解耦（disaggregated）部署，其中 prefill 与 decode worker 分离。你希望将 decode worker 的 tensor parallelism 改为 2。

**变更前** —— 原始部署：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-disagg
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
    VllmDecodeWorker:
      componentType: worker
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
          command:
          - python3
          - -m
          - dynamo.vllm
          args:
            - --model
            - Qwen/Qwen3-0.6B
            - --disaggregation-mode
            - decode
    VllmPrefillWorker:
      componentType: worker
      subComponentType: prefill
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
          command:
          - python3
          - -m
          - dynamo.vllm
          args:
            - --model
            - Qwen/Qwen3-0.6B
            - --disaggregation-mode
            - prefill
```

**变更后** —— 调整并行度后的更新：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-disagg
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
    VllmDecodeWorker:
      componentType: worker
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
          command:
          - python3
          - -m
          - dynamo.vllm
          args:
            - --model
            - Qwen/Qwen3-0.6B
            - --disaggregation-mode
            - decode
            - --tensor-parallelism
            - "2"
    VllmPrefillWorker:
      componentType: worker
      subComponentType: prefill
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
          command:
          - python3
          - -m
          - dynamo.vllm
          args:
            - --model
            - Qwen/Qwen3-0.6B
            - --disaggregation-mode
            - prefill
```

应用更新：

```bash
kubectl apply -f vllm-disagg.yaml
```

监控滚动更新进度：

```bash
kubectl get dgd vllm-disagg -n dynamo -o jsonpath='{.status.rollingUpdate}'
```

## 默认行为（Grove 与 LWS）

对于由 **Grove**（PodCliques、PodCliqueSets）或 **LWS**（LeaderWorkerSets）支撑的 DGD，operator 不直接管理滚动更新。这类部署依赖其底层资源原生的滚动更新机制。

### 发生了什么

- 对某个服务的 pod spec 修改会触发底层资源的滚动更新。在上面的示例中，对 decode worker pod spec 的修改只会触发该 decode worker 的滚动更新。
- 对于 Grove，PodCliques（PCLQ）与 PodCliqueScalingGroups 使用静态滚动更新策略 `maxUnavailable: 1` 与 `maxSurge: 0`。LWS 同样使用 `maxUnavailable: 1` 与 `maxSurge: 0`。
- **新旧 worker 在同一个 Dynamo 命名空间内运行。** 这意味着新旧 worker 可通过服务发现互相发现。

下图展示了 Grove PodCliqueSet（PCS）中 decode worker 的滚动更新。仅 decode 这个 PodClique 被更新——前端与 prefill PodClique 不受影响：

```
┌─ PodCliqueSet: vllm-disagg ───────────────────────────────────────────────────────┐
│                                                                                    │
│  ┌─ PCLQ: Frontend ──────┐  ┌─ PCLQ: VllmPrefillWorker ─┐                        │
│  │                        │  │                            │                        │
│  │  ┌──────────────────┐  │  │  ┌──────────────────────┐  │                        │
│  │  │ Pod (v1) ✓       │  │  │  │ Pod (v1) ✓           │  │   No changes —        │
│  │  └──────────────────┘  │  │  └──────────────────────┘  │   not rolling          │
│  │                        │  │                            │                        │
│  └────────────────────────┘  └────────────────────────────┘                        │
│                                                                                    │
│  ┌─ PCLQ: VllmDecodeWorker ──────────────────────────────────────────────────────┐ │
│  │                                                                                │ │
│  │  maxUnavailable: 1, maxSurge: 0                                                │ │
│  │                                                                                │ │
│  │  ┌──────────────────────┐  ┌──────────────────────┐                            │ │
│  │  │ Pod (v2) ✓ NEW       │  │ Pod (v1) Terminating │  ← rolling one at a time   │ │
│  │  └──────────────────────┘  └──────────────────────┘                            │ │
│  │                                                                                │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                    │
│                        ┌──────────────────────────────────┐                        │
│                        │  Dynamo Namespace: vllm-disagg   │                        │
│                        │                                  │                        │
│                        │  All v1 and v2 pods registered   │                        │
│                        │  and discoverable by each other  │                        │
│                        └──────────────────────────────────┘                        │
│                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### 对解耦部署的影响

由于新旧 worker 共享同一 Dynamo 命名空间，它们会被路由器（router）归入一组。在解耦设置中，这可能导致跨代际通信——例如，路由器可能将一个新部署的 prefill worker 的请求发送到旧的 decode worker（或反过来）。如果新旧版本不兼容，可能会出错。

> [!WARNING]
> 对于使用 Grove 和 LWS 且包含解耦 prefill/decode worker 的部署，请注意在滚动更新期间，新 worker 可能与旧 worker 通信。请确保你的 worker 版本向后兼容，或考虑使用由 Deployment 支撑的 DGD，它在更新期间提供命名空间隔离。

> [!NOTE]
> 计划在未来版本中为 Grove 和 LWS 支撑的部署提供带命名空间隔离的托管滚动更新。详见 [未来工作](#future-work)。

## 托管滚动更新（Deployments）

对于由 Kubernetes **Deployment** 支撑的 DGD（单节点、非多节点服务），Dynamo operator 实现了带命名空间隔离的托管滚动更新。该过程会在 DGD 状态中跟踪，为解耦部署提供更强保证。

### 工作原理

1. **Spec 变更检测** —— operator 计算所有 worker 服务（prefill、decode 与 worker 组件类型）spec 的哈希。当该哈希变化时，触发滚动更新。

2. **命名空间隔离** —— 创建新的 worker `DynamoComponentDeployment`（DCD）时，会将 spec 哈希追加到其 Dynamo 命名空间中。这意味着新 worker 注册到一个不同于旧 worker 的 Dynamo 命名空间，阻止跨代际发现。新 prefill worker 只会发现并路由到新的 decode worker，避免兼容性问题。

3. **逐步替换** —— operator 在遵守 `maxSurge` 与 `maxUnavailable` 约束下，逐步扩容新 worker DCD 并缩容旧 DCD。当某个 worker 服务完成更新（所有新副本就绪、所有旧副本终止）时，将其标记为已完成。

4. **清理** —— 当所有 worker 服务都完成切换后，旧 worker DCD 被删除，滚动更新被标记为已完成。

```
┌─ DynamoGraphDeployment: vllm-disagg ──────────────────────────────────────────────┐
│                                                                                    │
│  ┌─ DCD: Frontend ──────────┐                                                      │
│  │                          │                                                      │
│  │  ┌────────────────────┐  │   No changes —                                       │
│  │  │ Pod (v1) ✓         │  │   not a worker component                             │
│  │  └────────────────────┘  │                                                      │
│  │                          │                                                      │
│  └──────────────────────────┘                                                      │
│                                                                                    │
│  ┌─ OLD DCDs (hash: a1b2c3d4) ──────────────────────────────────────────────────┐  │
│  │                                                                               │  │
│  │  ┌─ DCD: VllmDecodeWorker-a1b2c3d4 ──┐  ┌─ DCD: VllmPrefillWorker-a1b2c3d4 ┐│  │
│  │  │                                    │  │                                   ││  │
│  │  │  ┌──────────────────────┐          │  │  ┌─────────────────────┐          ││  │
│  │  │  │ Pod (v1) Terminating │          │  │  │ Pod (v1) Terminating│          ││  │
│  │  │  └──────────────────────┘          │  │  └─────────────────────┘          ││  │
│  │  │                                    │  │                                   ││  │
│  │  │  Dynamo Namespace: vllm-disagg     │  │  Dynamo Namespace: vllm-disagg    ││  │
│  │  │                  -a1b2c3d4         │  │                  -a1b2c3d4        ││  │
│  │  └────────────────────────────────────┘  └───────────────────────────────────┘│  │
│  │                                                                               │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  ┌─ NEW DCDs (hash: f5e6d7c8) ──────────────────────────────────────────────────┐  │
│  │                                                                               │  │
│  │  ┌─ DCD: VllmDecodeWorker-f5e6d7c8 ──┐  ┌─ DCD: VllmPrefillWorker-f5e6d7c8 ┐│  │
│  │  │                                    │  │                                   ││  │
│  │  │  ┌──────────────────────┐          │  │  ┌─────────────────────┐          ││  │
│  │  │  │ Pod (v2) ✓ NEW      │          │  │  │ Pod (v2) ✓ NEW     │          ││  │
│  │  │  └──────────────────────┘          │  │  └─────────────────────┘          ││  │
│  │  │                                    │  │                                   ││  │
│  │  │  Dynamo Namespace: vllm-disagg     │  │  Dynamo Namespace: vllm-disagg    ││  │
│  │  │                  -f5e6d7c8         │  │                  -f5e6d7c8        ││  │
│  │  └────────────────────────────────────┘  └───────────────────────────────────┘│  │
│  │                                                                               │  │
│  └───────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                    │
│  Old and new workers are in different Dynamo namespaces —                           │
│  new prefill only discovers new decode, preventing cross-generation routing.        │
│                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────┘
```

> [!NOTE]
> 只有 worker 组件类型（`worker`、`prefill`、`decode`）参与托管滚动更新。诸如 `frontend` 这类非 worker 组件会就地更新，不带命名空间隔离。

### 滚动更新阶段

滚动更新进度被跟踪在 `.status.rollingUpdate` 中，包含以下阶段：

| 阶段 | 说明 |
|-------|-------------|
| `Pending` | 检测到 spec 变更，滚动更新已初始化。 |
| `InProgress` | 新 worker DCD 正在扩容，旧的正在缩容。 |
| `Completed` | 所有 worker 服务已切换到新副本。旧 DCD 已被清理。 |

状态中还会跟踪：
- `startTime` —— 滚动更新开始时间。
- `endTime` —— 滚动更新完成时间。
- `updatedServices` —— 已完成切换的 worker 服务列表。

### 配置 maxSurge 与 maxUnavailable

可以通过注解（annotation）按服务配置滚动更新策略：

| 注解 | 说明 | 默认 |
|------------|-------------|---------|
| `nvidia.com/deployment-rolling-update-max-surge` | 更新期间在期望副本数之上可创建的最大额外 pod 数。 | `25%` |
| `nvidia.com/deployment-rolling-update-max-unavailable` | 更新期间允许不可用的最大 pod 数。 | `25%` |

取值可以是绝对整数（如 `"1"`、`"2"`）或百分比（如 `"25%"`、`"50%"`）。百分比基于期望副本数解析——`maxSurge` 向上取整，`maxUnavailable` 向下取整。operator 会确保 `maxSurge` 或 `maxUnavailable` 至少有一个大于零，以保证向前推进。

**示例** —— 带 surge 容量的零停机更新：

```yaml
VllmPrefillWorker:
  componentType: worker
  subComponentType: prefill
  replicas: 4
  annotations:
    nvidia.com/deployment-rolling-update-max-surge: "1"
    nvidia.com/deployment-rolling-update-max-unavailable: "0"
```

这样可保证现有 4 个 prefill 副本一直可用，每次新增 1 个新副本。

**示例** —— 允许临时容量下降的快速更新：

```yaml
VllmDecodeWorker:
  componentType: worker
  subComponentType: decode
  replicas: 8
  annotations:
    nvidia.com/deployment-rolling-update-max-surge: "0"
    nvidia.com/deployment-rolling-update-max-unavailable: "2"
```

这避免了创建额外的 pod，但允许同时有最多 2 个 decode 副本不可用，从而加快切换速度。

### Worker 哈希与 DCD 命名

worker DCD 名称中始终包含从 worker spec 派生的哈希后缀：`{dgd-name}-{service-name}-{hash}`（如 `vllm-disagg-vllmdecodeworker-a1b2c3d4`）。滚动更新期间，新 worker DCD 使用新的 spec 哈希创建，而旧 DCD 保留其原哈希，使两代共存：

- **旧 worker DCD：** `vllm-disagg-vllmdecodeworker-a1b2c3d4`（旧哈希）
- **新 worker DCD：** `vllm-disagg-vllmdecodeworker-f5e6d7c8`（新哈希）

哈希通过对所有 worker 服务 spec（不包括 `replicas`、`autoscaling`、`ingress` 等非 pod template 字段）的 SHA-256 摘要计算。这意味着：

- 副本数变更（scaling）**不会**触发滚动更新。
- pod template 变更（镜像、资源、env vars、volumes 等）**会**触发滚动更新。
- 哈希覆盖**所有**worker 服务整体——任何单个 worker 的 spec 变更都会触发所有 worker 的滚动更新。

当前 worker 哈希作为注解 `nvidia.com/current-worker-hash` 存储于 DGD 资源上，单个 worker DCD 通过 label `nvidia.com/dynamo-worker-hash` 标记，便于过滤。

### 滚动更新期间的状态

滚动更新期间，DGD 状态会聚合新旧 worker DCD 的信息：

- **Replicas** —— 新旧合计的副本总数。
- **ReadyReplicas** —— 新旧合计的就绪副本数。
- **UpdatedReplicas** —— 仅新 worker 的副本数。

这为切换期间的部署健康状况提供了整体视图。

## 对比

| 方面 | Grove / LWS | Deployments（托管） |
|--------|-------------|----------------------|
| 更新机制 | 原生资源滚动更新 | operator 托管，配合 DCD 生命周期 |
| 命名空间隔离 | 无——新旧共用同一命名空间 | 有——基于哈希的命名空间分离 |
| 跨代际发现 | 可能——新旧 worker 互相可见 | 已阻止——新 worker 只发现新 worker |
| maxSurge / maxUnavailable | 固定（Grove 为 `maxUnavailable: 1`、`maxSurge: 0`） | 通过注解按服务可配置 |
| 状态跟踪 | 原生资源状态 | DGD `.status.rollingUpdate`，含阶段与按服务跟踪 |
| 多节点支持 | 是 | 否（仅单节点） |

## 未来工作

未来版本计划包括以下增强：

- **Grove 与 LWS 的托管滚动更新** —— 将带命名空间隔离的托管滚动更新扩展到 Grove 与 LWS 支撑的部署，提供与当前 Deployment 支撑的 DGD 一致的跨代际发现保护。
- **协调式 worker 更新** —— 当前 prefill 与 decode worker 各自独立更新，切换期间可能造成新旧集合不平衡。未来版本将跨 worker 类型协调发布。
- **分批发布** —— 能够先对一定比例（如 30%）的 worker 进行更新，暂停、观察指标后再继续。这支持金丝雀风格的发布以提升安全性。
- **DGD 级滚动更新配置** —— 不论底层资源类型如何，都能在 DGD API 层面配置 `maxSurge` 与 `maxUnavailable`。
