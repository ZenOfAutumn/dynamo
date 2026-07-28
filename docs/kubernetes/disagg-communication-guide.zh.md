---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
sidebar-title: Disagg Communication
subtitle: Best practices for prefill/decode worker communication on Kubernetes
---

# 解耦推理通信指南

本指南解释 Dynamo 在 Kubernetes 上解耦（disaggregated）推理架构中 prefill 与 decode worker 之间如何通信。它回答常见问题：**为什么同一节点上的 prefill 与 decode worker 不能用 NVLink 通信？**

## 摘要

- **NVLink 不能在 Kubernetes pod 之间使用**，因为存在进程隔离与 GPU 切分
- **生产解耦部署需要 RDMA**（InfiniBand、RoCE 或 AWS EFA）
- **没有 RDMA 时，TTFT（首 token 时延）会有 200-500 倍劣化** —— 实测 TCP 下 TTFT 约 98s，对比 RDMA 下约 200-500ms
- **UCX 或 libfabric** 是 NIXL 用于在 worker 间传输 KV cache 的通信层

---

## 架构概览

### 通信栈

<Frame>
  <img src="../assets/img/disagg-comm-stack.svg" alt="Disaggregated inference communication stack showing NIXL, UCX/libfabric, and transport layers" />
</Frame>

### 各组件职责

| 组件 | 角色 | 位置 |
|-----------|------|----------|
| **NIXL** | 高层 KV cache 传输 API | Dynamo runtime 库 |
| **UCX 或 libfabric** | 底层通信框架 | 系统库 |
| **传输（Transports）** | 物理数据搬运 | 硬件 / 内核驱动 |

---

## 为什么 NVLink 不能在 pod 之间使用

### 根本约束

NVLink 是工作在硬件层面的 **GPU 与 GPU 直连**。它要求：

1. **同一进程** —— 两个 GPU 必须对单个进程都可见，才能调用 `cudaDeviceEnablePeerAccess()`
2. **直接内存访问** —— 进程必须有权限访问两块 GPU 的内存区域
3. **点对点（peer-to-peer）映射** —— CUDA runtime 必须在两个 GPU 之间建立内存映射

**Kubernetes pod 同时违反了这三点：**

<Frame>
  <img src="../assets/img/disagg-nvlink-limitation.svg" alt="Why NVLink cannot work between Kubernetes pods due to process isolation" />
</Frame>

### 技术解释

1. **进程隔离**：Kubernetes pod 运行在不同的 Linux namespace 中。即便在同一节点上，Pod A 也无法直接访问 Pod B 的内存。

2. **GPU 切分**：Kubernetes device plugin 通过 `CUDA_VISIBLE_DEVICES` 把特定 GPU 分配给各 pod。Pod A 的 GPU 0 与 Pod B 的 GPU 0 是物理上不同的设备。

3. **进程 / Namespace 隔离**：每个 pod 在不同的进程 namespace 中运行。NVLink 点对点传输需要两个 GPU 同时在一个进程内才能调用 `cudaDeviceEnablePeerAccess()`。

4. **内存注册**：NVLink 传输使用启用了对端访问的 `cudaMemcpy`。这需要调用 `cudaDeviceEnablePeerAccess()`——跨进程边界不可行。

### NVLink 起作用的地方

NVLink 在 **同一 pod 内部** 用于并行策略（TP、EP），即所有 GPU 都属于同一进程：

```yaml
# Decode worker with TP=4 uses NVLink between its 4 GPUs
VLLMDecodeWorker:
  resources:
    limits:
      gpu: "4"   # All 4 GPUs visible to single process
  args:
    - --tensor-parallel-size
    - "4"        # NVLink used for TP/EP communication within pod
```

---

## 受支持的通信选项

### 传输对比

| 传输 | 带宽 | 时延 | 同节点 | 跨节点 | GPU Direct |
|-----------|-----------|---------|-----------|------------|------------|
| **NVLink** | 450-900 GB/s | ~µs | ✅（仅 pod 内） | ❌ | ✅ |
| **InfiniBand RDMA** | 20-50 GB/s | ~1 µs | ✅ | ✅ | ✅（含 GPUDirect） |
| **RoCE RDMA** | 10-25 GB/s | ~2 µs | ✅ | ✅ | ✅（含 GPUDirect） |
| **TCP** | 1-3 GB/s | ~50 µs | ✅ | ✅ | ❌（host staging） |

### 同节点通信

当 prefill 与 decode worker 在 **同一物理节点** 上时：

<Frame>
  <img src="../assets/img/disagg-same-node.svg" alt="Same-node RDMA communication between prefill and decode pods" />
</Frame>

**选项（从最佳到最差）：**
1. 带 GPUDirect 的 InfiniBand RDMA → GPU 到 GPU，绕过 CPU
2. 带 GPUDirect 的 RoCE RDMA → GPU 到 GPU，绕过 CPU
3. host 中转的 RDMA → GPU→CPU→RDMA→CPU→GPU
4. TCP（回退）→ GPU→CPU→TCP→CPU→GPU

**最佳实践**：即便在同节点也建议使用 RDMA。其开销很小，且无论 pod 落在同节点还是不同节点都行为一致。

### 跨节点通信

当 prefill 与 decode worker 在 **不同节点** 上时：

<Frame>
  <img src="../assets/img/disagg-cross-node.svg" alt="Cross-node RDMA communication between prefill and decode pods on separate nodes" />
</Frame>

**获得最佳跨节点性能的要求：**
- RDMA 网络（InfiniBand、RoCE 或 AWS EFA）
- 启用 GPUDirect RDMA（GPU 内存与 NIC 注册）
- 正确的 UCX 或 libfabric 配置

---

## UCX 配置参考

### 环境变量

UCX 行为通过环境变量控制。在 prefill 与 decode worker pod 上都要设置。

#### 核心传输选择

```yaml
env:
  - name: UCX_TLS
    value: "rc_x,rc,dc_x,dc,cuda_copy,cuda_ipc"
```

| 传输 | 说明 | 何时使用 |
|-----------|-------------|-------------|
| `rc_x` | Reliable Connection（accelerated） | 主 RDMA 传输 |
| `rc` | Reliable Connection（standard） | RDMA 回退 |
| `dc_x` | Dynamically Connected（accelerated） | 可扩展 RDMA（多端点） |
| `dc` | Dynamically Connected（standard） | 可扩展 RDMA 回退 |
| `cuda_copy` | GPU↔Host 内存中转 | GPU 缓冲区必需 |
| `cuda_ipc` | CUDA IPC（同节点、同 pod） | pod 内 GPU 传输 |
| `tcp` | TCP socket | RDMA 不可用时的回退 |
| `srd` | Scalable Reliable Datagram（AWS EFA） | AWS 专用（由 EFA 提供，非核心 UCX） |

**排除某些传输**：使用 `^` 前缀（例如 `UCX_TLS=^mm` 排除 memory mapping）。

**注意**：在 GPU 内存场景下显式指定 `UCX_TLS` 时，必须包含 `cuda_copy` 或 `cuda_ipc`，UCX 才能识别 GPU 缓冲区。

#### Rendezvous 协议设置

```yaml
env:
  - name: UCX_RNDV_SCHEME
    value: "get_zcopy"
  - name: UCX_RNDV_THRESH
    value: "0"
```

| 变量 | 取值 | 说明 |
|----------|-------|-------------|
| `UCX_RNDV_SCHEME` | `get_zcopy` | 零拷贝 RDMA GET（接收方拉取） |
| `UCX_RNDV_SCHEME` | `put_zcopy` | 零拷贝 RDMA PUT（发送方推送） |
| `UCX_RNDV_SCHEME` | `auto` | 由 UCX 按消息大小自动选择 |
| `UCX_RNDV_THRESH` | `0` | 所有大小的消息都用 rendezvous |
| `UCX_RNDV_THRESH` | `8192` | 消息 ≥8KB 用 rendezvous |
| `UCX_RNDV_THRESH` | `auto` | 由 UCX 计算最优阈值 |

**建议**：KV cache 传输（始终很大）使用 `get_zcopy` + 阈值 `0`。

> **⚠️ AWS EFA 例外**：在 AWS 上 Ubuntu 24.04 + 内核 ≥6.8 时，**不要** 使用 `get_zcopy`。所需设置见 [AWS EFA Configuration](#aws-efa-configuration)。

#### 内存注册

```yaml
env:
  - name: UCX_IB_REG_METHODS
    value: "odp,rcache"
```

| 方式 | 说明 |
|--------|-------------|
| `odp` | 按需分页（动态注册） |
| `rcache` | 注册缓存（复用注册） |
| `direct` | 直接注册（每次传输） |

#### 调试与诊断

```yaml
env:
  - name: UCX_LOG_LEVEL
    value: "info"        # Options: fatal, error, warn, info, debug, trace, data, func
  - name: UCX_LOG_FILE
    value: "/tmp/ucx.log" # Optional: log to file instead of stdout
```

**注意**：UCX 统计信息（`UCX_STATS_DEST`、`UCX_STATS_TRIGGER`）需要在编译 UCX 时加 `--enable-stats`，默认构建未启用。

### 完整生产配置

```yaml
env:
  # Transport selection - RDMA with GPU support
  - name: UCX_TLS
    value: "rc_x,rc,dc_x,dc,cuda_copy,cuda_ipc"

  # Rendezvous for large transfers
  - name: UCX_RNDV_SCHEME
    value: "get_zcopy"
  - name: UCX_RNDV_THRESH
    value: "0"

  # Memory registration optimization
  - name: UCX_IB_REG_METHODS
    value: "odp,rcache"

  # RDMA settings
  - name: UCX_IB_GID_INDEX
    value: "3"           # RoCE v2 GID index (cluster-specific)
```

### InfiniBand 配置

对于带 InfiniBand RDMA 的集群（例如 ConnectX 网卡），使用 UCX 的 `rc`（Reliable Connection）传输。这是本地与裸金属 Kubernetes 集群的标准路径。

**RDMA 资源：**

每张 GPU 申请一个 `rdma/ib` 设备。RDMA device plugin 会自动注入 `/dev/infiniband/*`：

```yaml
resources:
  limits:
    gpu: "4"
    custom:
      rdma/ib: "4"
```

无需 pod 注解。InfiniBand 设备由 device plugin 注入。

**Security Context：**

添加 `IPC_LOCK` 与 `SYS_RESOURCE` 能力。`IPC_LOCK` 允许 RDMA pin 内存，`SYS_RESOURCE` 允许提升 memlock 上限：

```yaml
securityContext:
  runAsUser: 0
  capabilities:
    add:
      - IPC_LOCK
      - SYS_RESOURCE
```

**环境变量（worker 容器）：**

```yaml
env:
  # --- UCX (RDMA transport) ---
  - name: UCX_TLS
    value: "rc_x,rc,cuda_copy,cuda_ipc"
  - name: UCX_NET_DEVICES
    value: "<ib-device>:1"       # e.g. "mlx5_0:1" — run `ibv_devinfo` to find your device
  - name: UCX_IB_ADDR_TYPE
    value: "eth"                 # required for cross-pod IB on Kubernetes
  - name: UCX_RNDV_SCHEME
    value: "get_zcopy"
  - name: UCX_RNDV_THRESH
    value: "0"
  - name: UCX_RC_TIMEOUT
    value: "600s"
  - name: UCX_KEEPALIVE_INTERVAL
    value: "300s"
```

| 变量 | 说明 |
|----------|-------------|
| `UCX_TLS` | 把 `rc_x`（accelerated RC）放在最前以获得最佳 RDMA 性能 |
| `UCX_NET_DEVICES` | 绑定到具体的 IB 设备。在 pod 内运行 `ibv_devinfo` 列出可用设备。选择具有有效 LID 的非 bond 设备。 |
| `UCX_IB_ADDR_TYPE` | 在 Kubernetes 上跨 pod IB 通信必须设为 `eth`。否则 UCX 使用基于 LID 的寻址，无法在 pod 间路由。 |
| `UCX_RNDV_SCHEME` | `get_zcopy` 启用零拷贝 RDMA GET，最适合大块 KV cache 传输 |

> **注意**：在 InfiniBand 集群上启用 NIXL 解耦时最常被遗漏的设置就是 `UCX_IB_ADDR_TYPE=eth`。如果 NIXL init 成功但传输报 `NIXL_ERR_REMOTE_DISCONNECT`，多半是这个原因。

**已知问题 —— bond 的 IB 设备：**

部分集群暴露的是 bond 的 InfiniBand 设备（例如 `mlx5_bond_0`），LID=0。如果 UCX 选中 bond 设备，传输可能失败。请检查设备 LID 并选用非 bond 设备：

```bash
# Inside a pod with rdma/ib resources:
ibv_devinfo | grep -E "hca_id|lid"
# Use a device with a non-zero LID in UCX_NET_DEVICES
```

### AWS EFA 配置

NIXL 在 AWS EFA 部署中支持 **libfabric** 作为后端。这是在 AWS 上做解耦推理的 **推荐方式**，可达到约 9.6 GB/s 的 KV 传输带宽。完整搭建说明参见 [AWS EFA with NIXL documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa-start-nixl.html)。

**要求：**
- EFA installer 版本 **1.47.0** 或更高
- libfabric（由 EFA installer 安装在 `/opt/amazon/efa`）
- 用于 GPU Direct RDMA 操作的 GDRCopy（GPU Operator v26.x 会自动安装）
- 启用 EFA 的容器镜像（例如 `nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1-efa-amd64`）

**内核兼容性：**

GDRCopy v2.5.1 在内核 6.15+ 上由于 `vm_flags_set` 重新定义而构建失败。在 GPU Operator 中提供 GDRCopy v2.5.2 之前，请将 Ubuntu EKS AMI 固定到内核 6.14 或更早。

| 内核版本 | GDRCopy v2.5.1 | GDRCopy v2.5.2 |
|----------------|----------------|----------------|
| 6.14 及以下 | ✅ 可用 | ✅ 可用 |
| 6.15+ | ❌ 构建失败 | ✅ 可用 |

**Pod Anti-Affinity（必需）：**

EFA 是为 **跨节点** 通信设计的。prefill 与 decode worker 必须调度到 **不同节点**，否则 KV 传输期间会出现 EAGAIN 错误。

```yaml
VllmDecodeWorker:
  extraPodSpec:
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
                - key: nvidia.com/dynamo-component
                  operator: In
                  values:
                    - VllmPrefillWorker
            topologyKey: kubernetes.io/hostname
```

> **注意**：anti-affinity 只需在一侧配置（这里是 decode worker）。Kubernetes 调度器会对称地强制约束——如果 decode 不能与 prefill 调度在一起，无论规则配置在哪一侧，它们都会被分到不同节点。

**EFA 资源请求：**

在 pod spec 中申请 EFA 接口。p5.48xlarge 实例有 **32 个 EFA 接口**（32 张网卡 × 每张 1 个接口），总带宽 3200 Gbps。每个 worker 分配多少接口取决于你的部署：

| 部署 | 每 worker EFA | 理由 |
|------------|----------------|-----------|
| 每对节点 1P + 1D | 4 | 实测约 9.6 GB/s；剩余 24 个接口可供其他 pod |
| 每节点多 worker | 2-4 | 在共享节点的 worker 之间均衡 |
| 最大带宽 | 8-16 | 用于非常大的 KV cache 传输或 TP>1 |

带 4 个 EFA 接口的示例（已验证配置）：

```yaml
extraPodSpec:
  mainContainer:
    securityContext:
      capabilities:
        add: ["IPC_LOCK"]
    resources:
      limits:
        vpc.amazonaws.com/efa: "4"
      requests:
        vpc.amazonaws.com/efa: "4"
```

> **注意**：NIXL/libfabric 会自动在所有已分配 EFA 接口上做 stripe。4 接口配置在测试中达到了约 9.6 GB/s，足以支撑 ISL=8000 下 Llama-3.1-8B 的 KV cache 传输。如果你的工作负载需要更高带宽（例如更大模型或更高 TP），请增加数量。

**环境变量：**

```yaml
env:
  - name: NIXL_LOG_LEVEL
    value: "INFO"
  - name: LD_LIBRARY_PATH
    value: "/usr/local/nixl/lib/x86_64-linux-gnu:/opt/amazon/efa/lib64:$(LD_LIBRARY_PATH)"
```

**vLLM 配置：**

```bash
vllm serve <your-model> \
    --kv-transfer-config '{"kv_connector":"NixlConnector","kv_role":"kv_both","kv_buffer_device":"cuda","kv_connector_extra_config":{"backends":["LIBFABRIC"]}}'
```

| 参数 | 取值 | 用途 |
|-----------|-------|---------|
| `kv_connector` | `NixlConnector` | 启用 NIXL 进行 KV cache 传输 |
| `kv_role` | `kv_both` | 对称功能（既是生产者也是消费者） |
| `kv_buffer_device` | `cuda` | 用 GPU 内存作为 KV cache 缓冲 |
| `backends` | `["LIBFABRIC"]` | 把 NIXL 流量路由到 EFA |

**验证：**

```bash
# Confirm EFA/libfabric installation
fi_info -p efa -t FI_EP_RDM

# Verify GDRCopy device
ls -la /dev/gdrdrv

# Check NIXL initialization in pod logs (should show 32 EFA devices on p5.48xlarge)
kubectl logs <worker-pod> | grep -i "NIXL\|libfabric\|efa"
```

**期望日志输出：**

```text
NIXL  INFO Loaded backend plugin: LIBFABRIC
NIXL  INFO Found 32 fabric devices
```

---

## 部署配置

### Kubernetes 资源要求

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
spec:
  services:
    VLLMPrefillWorker:
      resources:
        limits:
          gpu: "2"
      extraPodSpec:
        mainContainer:
          securityContext:
            capabilities:
              add: ["IPC_LOCK"]      # Required for RDMA memory pinning
          resources:
            limits:
              rdma/ib: "2"           # RDMA resources (match TP size)
            requests:
              rdma/ib: "2"
```

### 必需能力与资源

| 设置 | 用途 | 备注 |
|---------|---------|-------|
| `IPC_LOCK` 能力 | pin 内存供 RDMA 使用 | 绕过 RLIMIT_MEMLOCK；`ibv_reg_mr()` pin GPU/host 缓冲所必需 |
| `rdma/ib` 资源 | RDMA 网卡访问 | 由 RDMA device plugin 提供 |
| `sharedMemory.size` | 进程间 IPC | vLLM 16Gi，TRT-LLM 80Gi |

### 基础设施前置要求

1. **RDMA Device Plugin**：把 `rdma/ib` 或 `vpc.amazonaws.com/efa` 资源暴露给 Kubernetes
   ```bash
   # InfiniBand/RoCE
   kubectl get nodes -o jsonpath='{.items[*].status.allocatable.rdma/ib}'
   # AWS EFA
   kubectl get nodes -o jsonpath='{.items[*].status.allocatable.vpc\.amazonaws\.com/efa}'
   ```

2. **RDMA 网络**：以下之一：
   - InfiniBand 或 RoCE
   - AWS EFA（Elastic Fabric Adapter）

3. **GPUDirect RDMA**（可选但推荐）：
   - 启用了 GPUDirect 的 NVIDIA 驱动
   - 已加载 `nvidia-peermem` 内核模块（InfiniBand/RoCE）
   - 已安装 GDRCopy（AWS EFA + libfabric）

---

## 诊断与性能验证

### 部署前验证

#### 1. 检查 RDMA 是否可用

```bash
# Check RDMA devices on node
kubectl debug node/<node-name> -it --image=ubuntu:22.04 -- bash
ibv_devinfo
```

期望输出会显示 InfiniBand 或 RoCE 设备：
```text
hca_id: mlx5_0
        transport:                      InfiniBand (0)
        fw_ver:                         28.35.2000
        ...
```

#### 2. 检查 UCX 传输能力

```bash
# Inside a Dynamo worker pod
ucx_info -d
```

确认是否有 GPU 内存支持：
```text
# Memory domain: mlx5_0
#     Component: ib
#     memory types: host (access,reg,cache), cuda (access,reg,cache)
#                                            ^^^^ GPU memory supported
```

**如果只看到 `host`**：GPUDirect RDMA 没有生效。KV 传输会走 host 中转。

#### 3. 测试 UCX 性能

```bash
# Server (on decode worker pod)
ucx_perftest -t tag_bw -n 100 -s 134217728

# Client (on prefill worker pod)
ucx_perftest <server-ip> -t tag_bw -n 100 -s 134217728
```

**期望带宽**：
- InfiniBand HDR：每端口 20-25 GB/s
- RoCE 100GbE：10-12 GB/s
- TCP 回退：1-2 GB/s

### NIXL 基准工具

部署 NIXL 基准测试以验证端到端 KV 传输性能：

```bash
cd deploy/pre-deployment/nixl
./build_and_deploy.sh
```

它会部署一个测量经由 NIXL 实际 GPU 到 GPU 传输速率的基准。

### 运行时诊断

#### 验证 NIXL 后端初始化

```bash
kubectl logs <worker-pod> | grep -i "NIXL\|UCX"
```

**好的输出**：
```text
NIXL INFO Backend UCX was instantiated
```

**坏的输出**（RDMA 未生效）：
```text
UCX WARN no RDMA transports available
NIXL INFO falling back to TCP transport
```

#### 监控传输性能

在 Grafana dashboard 中查看：
- **NIXL 传输带宽**：应是 GB/s 级别，而非 MB/s
- **KV cache 传输延迟**：典型负载下应 < 500ms

**提示 RDMA 出问题的红线**：
- 传输带宽低于 1 GB/s
- TTFT > 10 秒
- 日志中出现 `Unsupported operation` 错误

### 常用诊断命令

```bash
# Check UCX transport selection
kubectl exec <pod> -- env | grep UCX

# Verify RDMA device visibility
kubectl exec <pod> -- ls /dev/infiniband/

# Check GPUDirect RDMA status (on node)
kubectl debug node/<node> -it --image=ubuntu:22.04 -- \
  nsenter -t 1 -m -u -n -p -- dmesg | grep -i "nvidia\|peermem\|gdr"

# Test basic connectivity between pods
kubectl exec <prefill-pod> -- ping -c 3 <decode-pod-ip>
```

---

## 性能预期

### KV cache 传输开销

| 配置 | TTFT 增量（平均） | KV 传输带宽 | 来源 |
|---------------|---------------------|----------------|--------|
| 聚合（基线） | 0 | 不适用 | 无需 KV 传输 |
| 解耦 + 带 GPUDirect 的 InfiniBand RDMA | +200-500ms | 20-50 GB/s | *基于硬件规格预期* |
| 解耦 + 带 GPUDirect 的 RoCE RDMA | +300-800ms | 10-25 GB/s | *基于硬件规格预期* |
| 解耦 + AWS EFA + libfabric + GDRCopy | **+37ms** | **~9.6 GB/s** | *实测* AWS p5.48xlarge（Llama-3.1-8B，ISL=8000，OSL=50） |
| 解耦 + Host-staged（无 GPUDirect） | +1-3s | 1-3 GB/s | *预期* —— CPU 瓶颈 |
| 解耦 + AWS EFA + UCX（无 GPUDirect） | 比聚合慢约 3 倍 | ~1 GB/s | *实测* AWS p5.48xlarge |
| 解耦 + TCP 回退 | **+90-100s** | ~100 MB/s | *实测* AWS p5.48xlarge 上 TTFT 约 98s |

> **注意**：在 AWS EFA 部署中，请使用 libfabric + GDRCopy 启用 GPUDirect RDMA。AWS EFA 上的 UCX 在内核 ≥6.8 时不支持 GPUDirect，性能严重下降。配置说明见 [AWS EFA Configuration](#aws-efa-configuration)。

### 何时该用解耦架构

**在以下情况使用解耦：**
- 输入序列长度（ISL）≥ 4000 token（吞吐提升 14-22%）
- 你需要独立扩缩 prefill 与 decode 的容量
- prefill 与 decode 有不同的硬件要求

**在以下情况使用聚合：**
- 低 TTFT 至关重要
- 输入序列短于 2000 token（解耦收益很小）
- 没有 RDMA 可用

### 盈亏平衡分析

KV 传输开销会被分摊到输出 token 上。**Llama-3.1-8B-Instruct 实测数据**（AWS p5.48xlarge + NIXL+libfabric）：

```text
KV Transfer Overhead (TTFT min, unqueued):
- Aggregated:    ~173ms
- Disaggregated: ~210ms
- KV transfer cost: ~37ms

Performance at ISL=8000, OSL=50, concurrency=10:
- ITL improvement: 41% faster per-token generation
- Throughput gain: 22% higher output throughput
```

**关键结论**：libfabric+EFA 的 KV 传输开销仅 **约 37ms**。结合快 41% 的 decode（ITL），解耦推理在 prefill bound 工作负载下带来 **22% 更高的吞吐**。

| 指标 | 聚合 | 解耦 | 差异 |
|--------|------------|---------------|------------|
| TTFT（min，未排队） | 173 ms | 210 ms | +37ms |
| TTFT（p95） | 2097 ms | 1752 ms | **-16%** |
| ITL（平均） | 28.5 ms | 16.9 ms | **-41%** |
| 输出吞吐（ISL=8000，OSL=50） | 204 tok/s | 248 tok/s | **+22%** |

**解耦优势随输入长度（ISL）增长**（均为 OSL=50，并发=10）：

| ISL | 吞吐 Δ | ITL Δ | 建议 |
|-----|--------------|-------|----------------|
| 1000 | ~0% | -7% | 用聚合 |
| 2000 | +3% | -11% | 二者皆可 |
| 4000 | +14% | -18% | 优先解耦 |
| 8000 | **+22%** | **-41%** | **强烈推荐解耦** |

---

## 故障排查指南

### 问题：TTFT 达 10+ 秒

**症状**：TTFT 从期望的 200-500ms 退化到 10+ 秒

**根因**：RDMA 未生效，回退到 TCP

**诊断**：
```bash
kubectl logs <worker-pod> | grep -i "transport\|UCX\|TCP"
```

**解决方案**：
1. 验证已安装 RDMA device plugin
2. 在 pod spec 中加入 `rdma/ib` 资源请求
3. 添加 `IPC_LOCK` 能力
4. 设置 UCX 环境变量

### 问题："Unsupported operation" 错误

**症状**：日志出现 `Unexpected UCX error: Unsupported operation`

**根因**：UCX 在不支持的硬件上尝试 GPU RDMA

**解决方案**：
1. 检查 GPUDirect RDMA 是否启用：`ucx_info -d | grep cuda`
2. 若不支持，设置 `UCX_RNDV_THRESH=inf` 关闭 GPU RDMA
3. 验证已加载 `nvidia-peermem` 模块

### 问题：AWS EFA 没有走 GPU Direct

**症状**：在 AWS 上 EFA 已配置，但性能仍下降 3 倍

**根因**：在内核 ≥6.8 + EFA + UCX 组合下 GPU Direct RDMA 不工作

**解决方案**：在 AWS EFA 部署中改用 libfabric 而非 UCX。libfabric + GDRCopy 在 AWS 上能高效进行 GPU Direct RDMA 操作。配置说明见 [AWS EFA Configuration](#aws-efa-configuration)。

**替代方案**（若 libfabric 不可用）：
1. 使用 6.8 之前的内核（Ubuntu 22.04 + 5.15 内核）
2. 接受 host staging 的性能损失

### 问题：EFA EAGAIN 错误（fi_read 仍在重试）

**症状**：decode worker 日志反复出现 EAGAIN：

```text
fi_read still retrying EAGAIN on rail 0
fi_read still retrying EAGAIN on rail 1
...
```

**根因**：prefill 与 decode worker 被调度到了 **同一节点**。AWS EFA 是为跨节点通信设计的，对节点内传输不能正确工作。

**诊断**：

```bash
# Check if workers are on the same node
kubectl get pods -o wide | grep vllm
```

如果两个 worker 显示同一 NODE，就是这个问题。

**解决方案**：加 pod anti-affinity 规则把 worker 调度到不同节点：

```yaml
VllmDecodeWorker:
  extraPodSpec:
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
                - key: nvidia.com/dynamo-component
                  operator: In
                  values:
                    - VllmPrefillWorker
            topologyKey: kubernetes.io/hostname
```

> **注意**：使用的 label key 是 `nvidia.com/dynamo-component`，不是 `app.kubernetes.io/component`。Dynamo operator 用此 label 标识组件类型。

### 问题：InfiniBand 上 create_backend 时 NIXL_ERR_BACKEND

**症状**：NIXL 后端创建立刻失败并报 `NIXL_ERR_BACKEND`。UCX 日志显示：

```text
mlx5dv_devx_obj_destroy(SRQ) failed: Invalid argument
mlx5dv_devx_obj_destroy(CQ) failed: Invalid argument
```

或：

```text
select.c: no active messages transport: Unsupported operation
```

**根因**：

1. **bond 的 IB 设备 LID=0**：UCX 默认选中 `mlx5_bond_0`，但 bond 设备的 LID 可能为 0（对 UD 传输无效）。修复：把 `UCX_NET_DEVICES` 设到具有有效 LID 的非 bond 设备。

2. **UCX/OFED 版本不一致**：容器中的 UCX mlx5 库可能针对与宿主内核驱动不同的 devx ABI 编译。任何使用 IB 的传输（rc、配合 IB 的 cuda_ipc）都会触发 devx 崩溃。

3. **RDMA 设备未注入**：如果 pod spec 没请求 `rdma/ib`，IB 设备就不会注入容器。

**诊断**：

```bash
# Check which IB devices are visible and their LIDs
ibv_devinfo | grep -E "hca_id|lid"

# Verify rdma/ib was requested
kubectl get pod <pod> -o jsonpath='{.spec.containers[0].resources}'

# Check /dev/infiniband exists
ls -la /dev/infiniband/
```

**解决方案**：
1. 在 pod spec 中请求 `rdma/ib` 资源（每张 GPU 1 个）
2. 如果 `mlx5_bond_0` 的 LID=0，把 `UCX_NET_DEVICES` 设到非 bond 设备
3. 确保容器镜像的 UCX 构建与宿主 OFED 版本匹配

### 问题：传输偶发失败

**症状**：偶发 `getXferStatus: backend 'UCX' returned error status`

**诊断**：
```bash
# Enable UCX debug logging
kubectl set env deployment/<worker> UCX_LOG_LEVEL=debug
kubectl logs <worker-pod> | grep -i error
```

**常见原因**：
- 网络拥塞或丢包
- pod 间 UCX 版本不一致
- RDMA 资源耗尽

---

## 速查参考

### 最小可行 RDMA 配置

```yaml
env:
  - name: UCX_TLS
    value: "rc_x,rc,dc_x,dc,cuda_copy,cuda_ipc"
  - name: UCX_RNDV_SCHEME
    value: "get_zcopy"
  - name: UCX_RNDV_THRESH
    value: "0"

securityContext:
  capabilities:
    add: ["IPC_LOCK"]

resources:
  limits:
    rdma/ib: "2"
  requests:
    rdma/ib: "2"
```

### 诊断清单

- [ ] `rdma/ib` 资源可见：`kubectl get nodes -o jsonpath='{..allocatable.rdma/ib}'`
- [ ] NIXL 已初始化：`kubectl logs <pod> | grep "Backend"`
- [ ] 传输带宽 > 1 GB/s（查看 Grafana 指标）

**UCX 部署：**
- [ ] UCX 看到 RDMA 设备：`ucx_info -d | grep "Transport: rc"`
- [ ] UCX 看到 GPU 内存：`ucx_info -d | grep "memory types.*cuda"`

**libfabric 部署（AWS EFA）：**
- [ ] EFA 设备可用：`fi_info -p efa`
- [ ] 已安装 GDRCopy：`ls /dev/gdrdrv`

---

## 相关文档

- [Disaggregated Serving Architecture](../design-docs/disagg-serving.md)
- [AIConfigurator Deployment Guide](../features/disaggregated-serving/README.md)
- [NIXL Benchmark Deployment](../../deploy/pre-deployment/nixl/README.md)
- [KV Cache Transfer Methods](../backends/trtllm/trtllm-kv-cache-transfer.md)
