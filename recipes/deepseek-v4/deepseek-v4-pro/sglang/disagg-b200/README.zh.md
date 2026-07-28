<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# DeepSeek-V4-Pro — B200 上的解耦预填充/解码（Disaggregated Prefill/Decode）

在搭载 InfiniBand RDMA 的 B200 节点上，通过 Dynamo 使用 SGLang 以解耦（disaggregated）预填充/解码（prefill/decode）的方式提供 [deepseek-ai/DeepSeek-V4-Pro](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro) 服务。

## 拓扑

| 角色    | 节点数 | 每节点 GPU | 总 GPU | 并行度 |
|---------|-------|-----------|------------|-------------|
| 解码（Decode） | 1     | 8         | 8          | TP8         |
| 预填充（Prefill） | 1     | 8         | 8          | TP8         |

## 前置条件

- 2× B200 节点，配备 InfiniBand 与 `rdma/ib` 设备插件
- 已安装 Dynamo operator 与 dynamo-platform HelmRelease
- 用于模型权重的共享 RWX PVC（`shared-model-cache`）
- Hugging Face token secret：
  ```bash
  kubectl create secret generic hf-token-secret \
    --from-literal=HF_TOKEN=<your-token> -n <namespace>
  ```

## 快速开始

```bash
# 0. 将 deploy.yaml 中的 <your-namespace> 替换为你的命名空间

# 1. 部署
kubectl apply -f deploy.yaml

# 2. 测试
kubectl port-forward svc/dsv4-pro-disagg-frontend 8000:8000 &
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek-ai/DeepSeek-V4-Pro","messages":[{"role":"user","content":"Hello!"}],"max_tokens":128}'
```

## InfiniBand RDMA 配置

`deploy.yaml` 中包含已在 nscale B200（ConnectX-8）上验证的 InfiniBand 专用 UCX 配置：

| 变量 | 值 | 用途 |
|----------|-------|---------|
| `UCX_TLS` | `rc_x,rc,cuda_copy,cuda_ipc` | 用于 IB RDMA 的加速 RC 传输 |
| `UCX_NET_DEVICES` | `mlx5_0:1` | 单一 IB 设备（避免绑定设备问题） |
| `UCX_IB_ADDR_TYPE` | `eth` | 以太网寻址 — Kubernetes 上跨 Pod IB 通信所必需 |
| `UCX_RNDV_SCHEME` | `get_zcopy` | KV 传输使用零拷贝 RDMA GET |
| `UCX_RNDV_THRESH` | `0` | 对所有消息大小启用 rendezvous |
| `rdma/ib` | `8`（每 GPU 1 个） | RDMA 设备插件注入 IB 设备 |

### 为什么使用 `UCX_IB_ADDR_TYPE=eth`？

未设置该项时，UCX 默认使用基于 LID 的 InfiniBand 寻址方式，在 Kubernetes 跨节点 Pod 间无法正确路由。设为 `eth` 会切换到基于 GID/以太网的寻址方式，可在 Pod 网络中正常工作。这是在 InfiniBand 集群上启用 NIXL 解耦（disagg）部署时最常见的问题。

### 绑定 IB 设备的坑

某些集群会暴露 LID=0 的 `mlx5_bond_0`。设置 `UCX_NET_DEVICES=mlx5_0:1` 可避开该设备。如果你的集群有不同的 IB 设备命名方案，请使用 `ibv_devinfo` 查看。

## 关键配置说明

- **EAGLE 已禁用** — 在 TP8 上会引发 OOM（启用 EAGLE 时仅剩 4.23 GB 空闲，未启用时为 7.75 GB）
- **`mem-fraction-static=0.82`** — 高于 GB200 食谱的 0.75，因为 B200 单节点 TP 不存在多节点权重打洗（weight shuffle）开销
- **安全** — RDMA 内存 pin 操作需要 `IPC_LOCK` + `SYS_RESOURCE` 能力
- **冷启动约 20 分钟** — FP4 权重打洗 + CUDA graph 捕获

## 相关链接

- [解耦（Disagg）通信指南](../../../../../docs/kubernetes/disagg-communication-guide.md) — 完整的 RDMA 传输参考
- [GB200 解耦食谱](../disagg-gb200/) — GB200 上基于 ComputeDomain 的多节点解耦
