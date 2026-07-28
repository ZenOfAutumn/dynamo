<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# DeepSeek-V4-Pro —— 在 GB200 上的解耦预填充/解码（Disaggregated Prefill/Decode）

在 GB200 节点上，通过 Dynamo 使用 SGLang 以解耦的预填充/解码模式提供 [deepseek-ai/DeepSeek-V4-Pro](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro) 服务。

## 拓扑

| 角色    | 节点数 | GPUs/节点 | 总 GPU | 并行方式 |
|---------|-------|-----------|------------|-------------|
| Decode  | 2     | 4         | 8          | TP8         |
| Prefill | 2     | 4         | 8          | TP8         |

## 前置条件

- 4× GB200 节点，具备 RDMA 网络
- 已安装 Dynamo operator + dynamo-platform HelmRelease
- ComputeDomain DRA 驱动（设备类 `compute-domain-default-channel.nvidia.com`）
- 用于模型权重的共享 RWX PVC

## 快速开始

```bash
# 1. Deploy
kubectl apply -f deploy.yaml

# 2. Test
kubectl port-forward svc/dsv4-pro-disagg-frontend 8000:8000 &
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek-ai/DeepSeek-V4-Pro","messages":[{"role":"user","content":"Hello!"}],"max_tokens":128}'
```

## 集群相关配置

基础的 `deploy.yaml` 是通用的。每种集群类型都需要额外配置 RDMA 网络（用于 NIXL 在预填充与解码之间传输 KV 缓存）。

### GKE（GCP GB200）

**RDMA 注解** —— 同时为 decode 和 prefill 服务添加：

```yaml
# In spec.services.decode and spec.services.prefill:
annotations:
  networking.gke.io/default-interface: eth0
  networking.gke.io/interfaces: |
    [
      {"interfaceName":"eth0","network":"default"},
      {"interfaceName":"rdma0","network":"rdma-0"},
      {"interfaceName":"rdma1","network":"rdma-1"},
      {"interfaceName":"rdma2","network":"rdma-2"},
      {"interfaceName":"rdma3","network":"rdma-3"}
    ]
resources:
  limits:
    custom:
      networking.gke.io.networks/rdma-0: "1"
      networking.gke.io.networks/rdma-1: "1"
      networking.gke.io.networks/rdma-2: "1"
      networking.gke.io.networks/rdma-3: "1"
```

**NATS 变通方法** —— GCP 集群可能将 `system-cpu` 和 `customer-gpu` 的 pod 网络隔离开。如果 dynamo-system 命名空间下的 NATS 不可达，请在 `customer-cpu` 节点上部署一个命名空间内的 NATS，并在 `spec.envs` 中覆盖 `NATS_SERVER`。

**Tolerations** —— GCP GPU 节点带有 taint。在所有服务中加入：

```yaml
tolerations:
  - key: dedicated
    operator: Equal
    value: user-workload
    effect: NoExecute
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
  - key: kubernetes.io/arch
    operator: Equal
    value: arm64
    effect: NoSchedule
```

### AWS EFA

EFA 专用、基于 libfabric 的 NIXL 配置请参阅 [Disagg Communication Guide](../../../../../docs/kubernetes/disagg-communication-guide.md#aws-efa-configuration)。

### InfiniBand

对于使用原生 IB 的集群（例如 DGX），请添加 `rdma/ib` 资源限制，并确认你的 ConnectX 固件支持 UCX UD 传输。

## 关键配置说明

- **EAGLE 已禁用** —— 在 TP8 下会 OOM（启用 EAGLE 时仅剩 4.23 GB，不启用时为 7.75 GB）
- **`mem-fraction-static=0.75`** —— 比默认值更低，避免在 FP4 MxFP4 权重 shuffle 时 OOM
- **`cuda-graph-max-bs=128`** —— 限制 CUDA graph 捕获的内存
- **冷启动约 45 分钟** —— 启动时间主要由 FP4 权重 shuffle 占据
