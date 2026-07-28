---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Shadow Engine Failover
---

> ⚠️ **实验性特性**：Shadow Engine Failover（影子引擎故障转移）是一项可选启用的
> 预览特性。它依赖 GPU Memory Service（GMS）、Dynamic Resource Allocation（DRA）
> 以及后端（backend）专门的支持。其 API 形态与行为可能发生变化，故障转移状态机
> 也仍在持续演进中。除非你已经针对集群中具体的后端、拓扑与故障模式做过验证，
> 否则只用于非生产环境的评估。

## 概述

当你希望在 GPU 与节点保持健康的前提下、由备用引擎接管未知后端引擎或软件进程
故障时，使用 Shadow Engine Failover。其目标是在同节点进程故障后，避免再次为
模型权重做一次完整的重新加载。

Shadow Engine Failover 是 Kubernetes 上的工作流；GPU Memory Service 则是其
底层的支撑机制：GMS 持有驻留在 GPU 上的模型权重，主用与备用引擎都通过 DRA
attach 到这份权重上。

它与 [Dynamo Snapshot](snapshot.md) 是不同的功能。Snapshot 通过 CRIU 与
`cuda-checkpoint` 捕获并恢复进程映像。Shadow Engine Failover 是把模型权重
保持驻留在 GPU 显存中，从而在某些进程级别故障后，让一个备用或替换的引擎
attach 上来。两者都关注恢复延迟，但解决的问题不同，且不可互相替代。

## 故障恢复流程

下图展示了同节点进程级别的恢复流程：

```text
┌──────────────────────── Same healthy node + GPU ───────────────────────┐
│                                                                        │
│  Before failure                                                        │
│  ┌──────────────┐      attach/use      ┌───────────────────────────┐   │
│  │ Engine A     │ ───────────────────▶ │ GMS-owned model weights   │   │
│  │ active       │                      │ resident in GPU memory    │   │
│  └──────┬───────┘                      └────────────┬──────────────┘   │
│         │                                           ▲                  │
│         │                                           │ attach/use       │
│         │ unknown software/engine failure           │                  │
│         ▼                                           │                  │
│  ┌──────────────┐                            ┌──────┴───────┐          │
│  │ Engine A     │ exits                      │ Engine B     │          │
│  └──────────────┘                            │ shadow       │          │
│                                              └──────┬───────┘          │
│                                                     │ takeover         │
│                                                     ▼                  │
│                                              ┌──────────────┐          │
│                                              │ Engine B     │          │
│                                              │ active       │          │
│                                              └──────────────┘          │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

**工作原理：**

1. 根据所选的故障转移模式，operator 会为该 worker 创建主用与备用的引擎容器
   或 Pod。
2. 引擎之间通过 DRA 共享 GPU 访问，并 attach 到由 GMS 持有的模型权重上。
3. 当未知的软件 / 引擎故障导致主用引擎退出时，GMS 进程、GPU 与节点仍然保持
   健康。
4. 备用或替换的引擎会接管，并 attach 到由 GMS 持有的、仍然驻留在 GPU 上的
   权重，而不再做一次完整的权重重新加载。
5. 在途请求（in-flight requests）与 KV 缓存（KV cache）状态不会被保留。如果
   GPU、节点或 GMS 进程丢失，则替换 worker 必须走常规的重新调度与模型加载
   流程。

## 当前何时使用

- 用它来评估在同节点上从未知 vLLM 引擎或软件进程故障中的恢复。
- 在你想要避免的成本是把模型权重再加载一份独立副本到 GPU 显存中时使用。
- 仅 GMS 的示例可用于验证后端通过 GMS 的权重加载，不要把它当作完整的故障
  转移工作流。
- 不要将其用于硬件故障、GPU 丢失、节点丢失、跨节点恢复、在途请求恢复或
  KV-cache 恢复等场景。
- 不要把它与 Snapshot restore 组合使用。Snapshot + GMS 目前还不可用。

## GPU Memory Service

GMS 把驻留在 GPU 上的模型权重的所有权从引擎进程移到一个独立的 GPU 内存服务
中。在故障转移工作流里，这使得主用与备用引擎共享同一份权重内存边界，而不是
各自加载独立的副本。

直接启用 GMS 适合做后端集成测试以及 sleep / wake 风格的生命周期实验。仅启用
GMS 本身并不会配置主备故障转移；如果需要影子引擎流程，请使用 `failover` 字段。

## 前置条件

- Kubernetes 1.32 或更新版本，且启用了 DRA。
- 已安装 NVIDIA GPU DRA 驱动。
- 一个匹配的 DRA `DeviceClass`，默认值为 `gpu.nvidia.com`。
- 一个受支持的后端镜像。当前的故障转移示例聚焦于 vLLM。
- 后端命令行支持从 GMS 加载，例如 `--load-format gms`。
- 节点上有足够的 GPU 显存可同时容纳 GMS 进程以及共享该设备的主用 / 备用引擎。

## 限制

- 这不是一个通用的 checkpoint / restore 系统。
- 这不是针对 GPU、节点或机柜级别丢失的硬件容错机制。
- 它不会诊断或修复后端故障本身。
- 它不会保留在途请求、网络套接字或 KV-cache 状态。
- 它不会让 GPU 内存负载支持 Snapshot restore。
- 由于已知的 GPU driver restore 问题，Snapshot + GMS 在准入（admission）层暂时
  被阻止。
- 由于其位于 `experimental` 之下，因此并不受常规 v1beta1 兼容性保证的覆盖。

## API 位置

对于 `v1alpha1` 的 `DynamoGraphDeployment`，GMS 与 failover 是 service 级别的
字段：

```yaml
gpuMemoryService:
  enabled: true
failover:
  enabled: true
```

对于 `v1beta1`，预览字段被分组到 `experimental` 之下，以明确标识其稳定性
约定：

```yaml
experimental:
  gpuMemoryService:
    mode: IntraPod
  failover:
    mode: IntraPod
```

具体到你 CRD 版本所支持的精确 schema，请参见 [API 参考](api-reference.md)。

## 基础 Shadow Engine Failover 示例

failover 在 GMS 的基础上构建。在 intra-pod 模式下，operator 会把 worker 的
主容器克隆为主用与备用两个引擎容器，它们通过 DRA 与 GMS sidecar 共享 GPU。
当主用引擎故障时，备用引擎会接管。

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-agg-failover
  annotations:
    nvidia.com/dynamo-kube-discovery-mode: container
spec:
  services:
    VllmWorker:
      componentType: worker
      replicas: 1
      resources:
        limits:
          gpu: "2"
      gpuMemoryService:
        enabled: true
      failover:
        enabled: true
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:my-tag
          args:
            - --model
            - Qwen/Qwen3-0.6B
            - --tensor-parallel-size
            - "2"
            - --load-format
            - gms
```

完整的 manifest 参见
[vLLM failover example](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/deploy/agg_failover.yaml)。

## 基础 GMS 示例

worker 必须通过常规的 Dynamo service 资源申请 GPU，启用 `gpuMemoryService`，
并运行能够从 GMS 加载的后端命令。

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-agg-gms
spec:
  services:
    VllmWorker:
      componentType: worker
      replicas: 1
      resources:
        limits:
          gpu: "1"
      gpuMemoryService:
        enabled: true
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:my-tag
          workingDir: /workspace/examples/backends/vllm
          command:
            - python3
            - -m
            - dynamo.vllm
          args:
            - --model
            - Qwen/Qwen3-0.6B
            - --load-format
            - gms
```

可工作的仅 GMS 示例：

- [vLLM GMS example](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/deploy/agg_gms.yaml)
- [SGLang GMS example](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/sglang/deploy/agg_gms.yaml)

## 相关文档

- [Snapshot](snapshot.md)
- [API 参考](api-reference.md)
- [vLLM failover example](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/deploy/agg_failover.yaml)
- [vLLM GMS example](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/deploy/agg_gms.yaml)
- [SGLang GMS example](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/sglang/deploy/agg_gms.yaml)
