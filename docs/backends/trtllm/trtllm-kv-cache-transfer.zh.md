---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: KV Cache Transfer
---

关于 TensorRT-LLM 的通用功能与配置，请参见[参考指南](trtllm-reference-guide.md)。

---

在解耦（disaggregated）服务架构中，KV 缓存（KV cache）必须在预填充（prefill）与解码（decode）worker 之间进行传输。TensorRT-LLM 支持两种传输方式：

## 使用 NIXL 进行 KV 缓存传输

启动解耦服务：参见[解耦服务](./trtllm-examples.md#disaggregated)以了解如何启动部署。

## 默认方式：NIXL
TensorRT-LLM 默认使用 **NIXL**（NVIDIA Inference Xfer Library），并以 UCX（Unified Communication X）为后端（backend）在 prefill 与 decode worker 之间传输 KV 缓存。[NIXL](https://github.com/ai-dynamo/nixl) 是 NVIDIA 设计的高性能通信库，用于在分布式 GPU 环境中高效地传输数据。

### 为 NIXL 指定后端

TensorRT-LLM 支持两种 NIXL 通信后端：UCX 与 LIBFABRIC。如果不显式指定后端，则默认使用 UCX。Dynamo 当前支持这两种后端。对于 AWS EFA 部署，已测试且推荐的后端是带 SRD 传输的 UCX（参见下文 [AWS EFA](#aws-efa)）。

## 替代方式：UCX

TensorRT-LLM 也可以直接利用 **UCX**（Unified Communication X）在 prefill 与 decode worker 之间传输 KV 缓存。要启用 UCX 作为 KV 缓存传输后端，请在引擎配置 YAML 中设置 `cache_transceiver_config.backend: UCX`。

> [!Note]
> 环境变量 `TRTLLM_USE_UCX_KVCACHE=1` 与 `cache_transceiver_config.backend: DEFAULT` 的组合**不会**启用 UCX。你必须在配置中显式设置 `backend: UCX`。

## AWS EFA

在 AWS 上，UCX 在 EFA 设备上使用 **SRD（Scalable Reliable Datagram）** 传输。NIXL 通过 UCX 自动发现 EFA 的 `rdmap*` 设备 —— 在 NIXL 层面无需任何配置改动。

**镜像选项：**

- **预构建 EFA 镜像（仅 AMD64）：** NGC 上提供了一个内置 EFA SDK 的专用 EFA 镜像。推荐用于 AMD64 实例（如 `p5.48xlarge`）：

```
nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime:1.1.1-efa-amd64
```

所有可用的 EFA 镜像见[发布产物](../../reference/release-artifacts.md)。

- **主机挂载方式（ARM64 / GB200）：** 未发布预构建的 EFA ARM64 镜像。请使用标准 `tensorrtllm-runtime` 镜像，并从主机节点（node）挂载 EFA SDK。这正是我们在 GB200 NVL72 上测试的方式：

```yaml
volumeMounts:
  - name: efa-sdk
    mountPath: /opt/amazon/efa
volumes:
  - name: efa-sdk
    hostPath:
      path: /opt/amazon/efa
```

**EFA 资源请求：**

```yaml
resources:
  requests:
    vpc.amazonaws.com/efa: "4"
  limits:
    vpc.amazonaws.com/efa: "4"
```

**EFA worker 必需的环境变量**（prefill 与 decode 都要设置）：

```yaml
env:
  - name: FI_PROVIDER
    value: "efa"
  - name: FI_EFA_USE_DEVICE_RDMA
    value: "1"
  - name: FI_EFA_ENABLE_SHM_TRANSFER
    value: "0"
  - name: LD_LIBRARY_PATH
    value: "/opt/amazon/efa/lib:/usr/local/lib:/usr/lib"
```

> [!IMPORTANT]
> `FI_EFA_ENABLE_SHM_TRANSFER` 必须为 `0`。SHM 传输会破坏 NIXL 的 GPU 缓冲注册。

**安全上下文：** AWS EFA 当前需要 privileged 模式：

```yaml
securityContext:
  privileged: true
```

### Decode 多节点上的 NIXL 插件 ABI 不匹配

在多节点 decode 时，decode leader 通过 `mpirun -> mgmn_worker_node` 启动 worker，加载的是 TRT-LLM 自带的 NIXL，而不是系统的 `nixl_cu13`。容器默认的 `NIXL_PLUGIN_DIR` 指向系统插件，与 TRT-LLM 自带的 NIXL ABI 不兼容。**仅在 decode 服务上**覆盖该变量：

```yaml
env:
  - name: NIXL_PLUGIN_DIR
    value: "/opt/dynamo/venv/lib/python3.12/site-packages/tensorrt_llm/libs/nixl/plugins"
```

不要在 prefill worker 上设置该变量 —— 它们使用的是 `nixl_cu13`，与系统插件兼容。

### GB200 NVL72 的 ComputeDomain

在 GB200 NVL72 机架上，NCCL 需要一个 `ComputeDomain` CR，以便正确初始化 cuMem/NVLS。否则 worker 在加载模型时会以 `NCCL error 'unhandled system error'` 失败。

```yaml
apiVersion: resource.nvidia.com/v1beta1
kind: ComputeDomain
metadata:
  name: my-compute-domain
spec:
  numNodes: 3    # prefill + decode 跨节点的总数
  channel:
    resourceClaimTemplate:
      name: my-compute-domain-channel
```

prefill 与 decode 服务都必须包含 ResourceClaim：

```yaml
resources:
  claims:
    - name: compute-domain-channel
extraPodSpec:
  resourceClaims:
    - name: compute-domain-channel
      resourceClaimTemplateName: my-compute-domain-channel
```

GB200 上 NCCL 必需的环境变量：

```yaml
env:
  - name: NCCL_MNNVL_ENABLE
    value: "1"
  - name: NCCL_CUMEM_ENABLE
    value: "1"
  - name: NCCL_NVLS_ENABLE
    value: "1"
  - name: NVIDIA_GDRCOPY
    value: "1"
```

### 验证 EFA 已启用

部署完成后，从 worker 日志确认 NIXL 正在通过 EFA 使用 SRD：

```bash
kubectl logs <prefill-pod> | grep -iE "NixlTransfer|srd|rdmap"
```

预期输出：

```
NixlTransferAgent using NIXL backend: UCX
ucp_context_2 self cfg#1 rma_am(srd/rdmap40s0:1) am(srd/rdmap40s0:1 srd/rdmap62s0:1 ...)
NixlTransferAgent mAddress: 100.x.x.x:32939
```

- `srd/rdmap*` 表示在 EFA 设备上使用 SRD 传输
- 多个 `rdmap` 条目对应每张 GPU 一个 EFA 设备
