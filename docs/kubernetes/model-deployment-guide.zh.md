---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Model Deployment Guide
---

# 模型部署指南

本指南介绍如何使用 `DynamoGraphDeploymentRequest`（DGDR）在 Dynamo 上部署模型。如果你已经完成了 [Quickstart](README.md) 并希望端到端地了解 DGDR 的工作方式、如何为模型选择正确设置以及如何避免常见陷阱，那么本指南正是你需要的。

DGDR 的 spec 参考、字段说明与生命周期阶段，参见 [DGDR Reference](dgdr.md)。

## 什么是 DGDR？

DGDR 是 Dynamo 的**意图驱动（deploy-by-intent）** Custom Resource。你不必手工编写包含并行设置、副本数与资源限制的 deployment spec，只需描述你希望运行**什么**（模型、后端、SLA 目标），DGDR 会自动完成其余事务：

1. **Spec** —— 你提交一个 DGDR，描述模型、负载预期与可选 SLA 目标。
2. **硬件发现** —— operator 通过 DCGM 或节点标签发现集群的 GPU 硬件（SKU、VRAM、每节点数量）。
3. **Profiling** —— profiler 在所发现的硬件上分析你的模型，使用快速模拟或真实 GPU 全面基准测试。
4. **DGD 生成** —— profiler 产出一个最优化的 `DynamoGraphDeployment`（DGD）spec，包含最佳并行策略、副本数与资源配置。
5. **审阅**（当 `autoApply: false`） —— 生成的 DGD 存入 `.status.profilingResults.selectedConfig`，便于你检查与可选地修改后再部署。
6. **部署** —— operator 创建 DGD，由其拉起推理 pod。
7. **Planner**（可选） —— 如启用，Planner 会监控线上流量并在运行时调整副本数以达到 SLA 目标。

```text
┌──────┐    ┌───────────┐    ┌──────────┐    ┌─────────────┐    ┌────────┐    ┌─────────┐
│ Spec │───▶│ Hardware  │───▶│ Profiler │───▶│ Generated   │───▶│ Deploy │───▶│ Planner │
│      │    │ Discovery │    │          │    │ DGD         │    │        │    │ (opt.)  │
└──────┘    └───────────┘    └──────────┘    └─────────────┘    └────────┘    └─────────┘
                                                   │
                                          autoApply: false?
                                             ▼ Review
```

## 选择搜索策略

`searchStrategy` 字段控制 profiler 如何探索配置。选择取决于你愿意投入多少时间，以及对最优解的接近程度有多严格。

### Rapid（默认）

```yaml
searchStrategy: rapid
```

使用 **AI Configurator（AIC）** 模拟 GPU 性能，无需进行真实推理。约 30 秒完成，profiling 期间不消耗 GPU 资源。

**适合何时使用：**
- 起步或快速迭代
- 在 CI/CD 流水线中运行
- 你的 GPU SKU 在 [AIC 支持矩阵](#aic-support-matrix) 中

**限制：**
- 如果 AIC 不支持你的 模型/硬件/后端 组合，profiler 会回退到一个简单的内存可放置配置（基础 TP 计算），可能并非最优。
- 在不寻常配置下，模拟结果可能与真实硬件性能存在差异。

### Thorough

```yaml
searchStrategy: thorough
backend: vllm  # must specify a concrete backend
```

枚举候选并行配置，将每一个部署到真实 GPU 上，并使用 AIPerf 基准测试。耗时 2–4 小时。

**适合何时使用：**
- 为生产调优、需要最优配置时
- 你的硬件不被 AIC 支持（例如 PCIe GPU）
- 你希望使用实测而非模拟性能数据

**约束：**
- **仅解耦模式** —— thorough 不会运行聚合配置。
- **不支持 `backend: auto`** —— 必须指定 `vllm`、`sglang` 或 `trtllm`。在 `thorough` 中使用 `auto` 的 DGDR 会被拒绝。
- **需要 GPU 资源** —— profiler 会在 profiling 期间在你的集群上部署真实推理引擎。

## AIC 支持矩阵

rapid 策略依赖 AIC 性能模型。AIC 当前支持：

### GPU SKU

| 已支持（rapid） | 暂不支持（请使用 thorough） |
|---|---|
| H100 SXM | V100（SXM/PCIe） |
| H100 PCIe | T4 |
| H200 SXM | MI200、MI300 |
| A100 SXM |  |
| A100 PCIe |  |
| A30 |  |
| B200 SXM |  |
| GB200 SXM |  |
| L40S |  |
| L4 |  |

> [!NOTE]
> 部分 rapid 模式下的 SKU 在尚未提供实测 profile 之前会使用 AIC 的估计数据。当你需要对仅有估计或不被支持的 SKU 做实测 profiling 时，请使用 `searchStrategy: thorough`。

手动指定 GPU SKU 时使用小写下划线格式（例如 `h100_sxm`，而非 `H100-SXM5-80GB`）。完整列表参见 [DGDR Reference — SKU Format](dgdr.md#sku-format)。

### 后端

三种后端都同时支持 rapid 与 thorough：

| 后端 | Dense 模型 | MoE 模型 |
|---------|-------------|------------|
| vLLM | ✅ | 🚧 进行中 |
| SGLang | ✅ | ✅ |
| TensorRT-LLM | ✅ | 🚧 进行中 |

**如果你正在部署 Mixture-of-Experts（MoE）模型**（例如 DeepSeek-R1、Qwen3-MoE），请使用 **SGLang** 作为后端以获得完整支持。vLLM 与 TRT-LLM 的 MoE 支持仍在开发中。

### 并行策略

profiler 会根据模型架构选择不同的并行策略：

| 模型架构 | Prefill | Decode |
|---|---|---|
| MLA+MoE（DeepSeek-V3、DeepSeek-R1） | TEP、DEP | TEP、DEP |
| GQA+MoE（Qwen3-MoE） | TP、TEP、DEP | TP、TEP、DEP |
| Dense 模型（Llama、Qwen 等） | TP | TP |

## 模型缓存（Model Caching）

**如有以下任一情况，请在部署前先搭建模型缓存：**

- 模型很大（>70B 参数） —— 每个 pod 都下载几百 GB 需数小时
- 你将扩到很多副本 —— 每个 pod 独立下载完整模型，HuggingFace 会对并发下载限速
- 你希望扩缩事件下 pod 启动迅速

### 与 DGDR 的协作方式

在 DGDR spec 中加入 `modelCache` 段，指向预填充好的 PVC：

```yaml
spec:
  model: meta-llama/Llama-3.1-70B-Instruct
  modelCache:
    pvcName: model-cache
    pvcMountPath: /home/dynamo/.cache/huggingface
    pvcModelPath: hub/models--meta-llama--Llama-3.1-70B-Instruct/snapshots/<commit-hash>
```

operator 会以只读方式将该 PVC 挂载到 `pvcMountPath`，进入 profiling job，并透传到生成的 DGD，从而 profiling 与服务都能复用缓存权重。

`pvcModelPath` 必须是 PVC 内部的 HuggingFace snapshot 路径 —— `hub/models--<org>--<model>/snapshots/<commit-hash>`。这与将 `HF_HOME` 设置为挂载点后 `huggingface-cli download` 创建的目录布局一致。把模型 ID 中的 `/` 替换为 `--` 得到 `<org>--<model>`，并以实际的 snapshot revision 替换 `<commit-hash>`。下载后如何查找哈希值参见 [Model Caching](model-caching.md#find-the-snapshot-path)。

### 设置

1. 创建一个 `ReadWriteMany` PVC —— 不同提供商的具体选项（EFS、Azure Lustre、GKE Filestore）参见 [Installation Guide — Shared Storage](installation-guide.md#shared-storage-for-model-caching)。
2. 运行一次性下载 Job 填充该 PVC。
3. 在 DGDR 的 `modelCache` 字段中引用该 PVC。

完整 YAML 示例与流程参见 [Model Caching](model-caching.md)。

### 私有与受限模型

对于需要鉴权的模型（如受限的 HuggingFace 模型），创建名为 `hf-token-secret`、含 `HF_TOKEN` 键的 Kubernetes Secret：

```bash
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=<your-token> \
  -n $NAMESPACE
```

profiler 与已部署的 pod 会自动使用该 token。

## 启用 Planner

Planner 为解耦部署提供**运行时自动扩缩**。它会随流量波动调整 prefill 与 decode 副本数以满足 SLA 目标。

```yaml
spec:
  features:
    planner:
      enabled: true
  sla:
    ttft: 500    # Target time to first token (ms)
    itl: 50      # Target inter-token latency (ms)
```

### Planner 扩缩模式

| 模式 | 描述 | 是否需要 Prometheus？ |
|---|---|---|
| `throughput`（默认） | 静态的队列深度与 KV 缓存阈值；按饱和度扩缩 | 否 |
| `latency` | 与 throughput 相同但阈值更激进 | 否 |
| `sla` | 基于回归模型，以特定 TTFT/ITL 为目标；同时使用 profiling 数据与线上指标 | 是 |

### Prometheus 要求

`sla` 优化目标会从 Prometheus 读取线上 TTFT/ITL 指标。如果你希望 SLA 驱动的自动扩缩，请在创建 DGDR 之前安装 Prometheus。详见 [Installation Guide — Prometheus](installation-guide.md#kube-prometheus-stack)。

`throughput` 与 `latency` 模式使用内部队列深度信号，**无需 Prometheus** 即可工作。

进一步的配置与扩缩行为细节，参见 [Planner Guide](../components/planner/planner-guide.md)。

## 多节点部署

需要的 GPU 数超过单节点能提供（例如 DeepSeek-R1 在 8-GPU 节点上）的模型，需要多节点编排。

### Grove 与 KAI Scheduler

**Grove** 是多节点 DGDR 部署的必需组件。它提供 gang scheduling（一组 pod 要么同时启动要么都不启动）、协调扩缩与拓扑感知的网络放置。如果你试图在没有 Grove 或 LeaderWorkerSet（LWS）的情况下进行多节点部署，operator 会返回错误。

**KAI Scheduler** 是可选的，但建议与 Grove 同时使用，以获得 GPU 感知的调度与拓扑优化。

设置说明与兼容性矩阵参见 [Installation Guide — Grove + KAI Scheduler](installation-guide.md#grove--kai-scheduler)。

### 高速网络（RDMA）

解耦服务会在 prefill 与 decode worker 之间传输 KV 缓存数据。理解网络栈有助于诊断性能问题：

| 层级 | 描述 |
|---|---|
| **NIXL** | Dynamo 的 KV 缓存传输库。在 prefill 与 decode pod 之间搬运数据。 |
| **UCX / libfabric** | NIXL 底层使用的低级通信框架。 |
| **RDMA** | Remote Direct Memory Access —— 在不经过 CPU 的情况下在机器间搬运数据的通用技术。 |
| **InfiniBand** | 高速 RDMA 网络标准。在本地机房与 Azure（AKS）上常见。 |
| **RoCE** | RDMA over Converged Ethernet —— 在标准以太网硬件上的 RDMA。 |
| **EFA** | AWS Elastic Fabric Adapter —— AWS 在 EKS 上的 RDMA 网络。 |
| **GPUDirect RDMA** | 让数据在 GPU 与网卡之间直接传输，完全绕过 CPU 内存。 |
| **NCCL** | NVIDIA Collective Communications Library —— 处理 pod _内部_ 模型并行（TP/PP）通信。与 NIXL 是分开的。 |

如果没有 RDMA，NIXL 会回退到 TCP，导致 TTFT **约 40 倍恶化**（从约 355ms 增至 10 秒以上）。

**在以下情况下启用 RDMA：**
- 进行多节点解耦部署
- 需要 worker 间低延迟的 KV 缓存传输

不同提供商的设置说明参见 [Installation Guide — Network Operator / RDMA](installation-guide.md#network-operator--rdma)；传输细节与性能预期参见 [Disaggregated Communication Guide](disagg-communication-guide.md)。

### MoE 模型与多节点扫描上限

profiler 对 MoE 模型最多扫描 **4 个节点**（dense 模型扫描时每引擎最多 1 个节点）。如果你的 MoE 模型需要超过 4 节点的 GPU，profiler 会在该范围内选择最佳配置，你可能需要手动调整副本数。

## 后端选择

`backend` 字段控制使用哪个推理引擎。默认值（`auto`）让 profiler 选择最佳后端，但在以下情况下应显式指定后端：

| 场景 | 推荐后端 |
|---|---|
| MoE 模型（DeepSeek-R1、Qwen3-MoE） | `sglang`（完整 MoE 支持） |
| 使用 `searchStrategy: thorough` | 除 `auto` 外的任意（必填） |
| TensorRT-LLM 编译缓存 | `trtllm`（添加编译缓存 PVC） |
| 需要负载式 planner 扩缩（FPM） | `vllm`（唯一带 ForwardPassMetrics 的后端） |

> [!WARNING]
> TensorRT-LLM 不支持 Python 3.11。如果你的环境使用 Python 3.11，请改用 `vllm` 或 `sglang`。

### 多节点后端行为

每种后端处理多节点推理的方式不同：

- **vLLM**：使用 Ray 进行多节点 TP/PP。Ray head 在 leader 上运行，agent 在 worker 上运行。
- **SGLang**：使用 `--dist-init-addr`、`--nnodes`、`--node-rank` 标志做分布式初始化。
- **TRT-LLM**：基于 MPI。operator 会自动生成 SSH 密钥对；leader 运行 `mpirun`。

## 常见陷阱

### Profiling 或服务期间 OOM

- **原因**：所选 TP 大小下模型不匹配 GPU 内存。
- **解决方法**：确保 `hardware.totalGpus` 足够。profiler 会按模型大小与 VRAM 计算最小 TP，但在边角场景（长上下文长度、KV 缓存开销）下可能需要超过最小值的更多 GPU。

### GPU 自动检测上限

operator 将 GPU 自动检测数量上限设为 **32**。如果你的集群有更多 GPU 并希望 profiler 使用，请显式设置 `hardware.totalGpus`：

```yaml
spec:
  hardware:
    totalGpus: 64
```

### Profiling Job 无法被调度

GPU 节点常常带有 taint。可通过 `overrides` 字段添加 toleration：

```yaml
spec:
  overrides:
    profilingJob:
      template:
        spec:
          containers: []    # required placeholder
          tolerations:
            - key: nvidia.com/gpu
              operator: Exists
              effect: NoSchedule
```

### DGDR Spec 不可变

一旦 DGDR 进入 `Profiling` 阶段，spec 就不能更改。如需调整设置，请删除并重建 DGDR：

```bash
kubectl delete dgdr my-model -n $NAMESPACE
kubectl apply -f updated-dgdr.yaml -n $NAMESPACE
```

### DGD 在 DGDR 删除后仍然存在

删除 DGDR **不会** 删除它创建的 DGD。这是有意为之 —— DGD 会继续独立提供服务。要彻底清理：

```bash
kubectl delete dgdr my-model -n $NAMESPACE
kubectl delete dgd my-model-dgd -n $NAMESPACE
```

## 整合：示例工作流

### 小型 Dense 模型（快速上手）

单节点上小模型 + rapid profiling —— 最简单的场景：

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: qwen-small
spec:
  model: Qwen/Qwen3-0.6B
```

### 大 Dense 模型 + SLA 目标

70B 模型，启用模型缓存、SLA 目标与 planner：

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: llama-70b
spec:
  model: meta-llama/Llama-3.1-70B-Instruct
  backend: vllm
  searchStrategy: rapid
  autoApply: false
  modelCache:
    pvcName: model-cache
    pvcMountPath: /home/dynamo/.cache/huggingface
    pvcModelPath: hub/models--meta-llama--Llama-3.1-70B-Instruct/snapshots/<commit-hash>
  sla:
    ttft: 500
    itl: 50
  workload:
    isl: 4000
    osl: 1000
    requestRate: 10
  features:
    planner:
      enabled: true
```

### MoE 模型（DeepSeek-R1）

需要多节点、SGLang 后端与 thorough profiling 的大型 MoE 模型：

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: deepseek-r1
spec:
  model: deepseek-ai/DeepSeek-R1
  backend: sglang
  searchStrategy: thorough
  autoApply: false
  modelCache:
    pvcName: model-cache
    pvcMountPath: /home/dynamo/.cache/huggingface
    pvcModelPath: hub/models--deepseek-ai--DeepSeek-R1/snapshots/<commit-hash>
  sla:
    ttft: 2000
    itl: 100
  hardware:
    totalGpus: 32
  features:
    planner:
      enabled: true
  overrides:
    profilingJob:
      template:
        spec:
          containers: []
          tolerations:
            - key: nvidia.com/gpu
              operator: Exists
              effect: NoSchedule
```

**该部署的前置条件：**
- 已安装 [Grove 与 KAI Scheduler](installation-guide.md#grove--kai-scheduler)
- 已为高效 KV 缓存传输配置 [RDMA](installation-guide.md#network-operator--rdma)
- 模型已[缓存到共享 PVC](installation-guide.md#shared-storage-for-model-caching)
- 已安装 [Prometheus](installation-guide.md#kube-prometheus-stack)（用于 SLA 驱动的 planner 扩缩）

## 进一步阅读

- [DGDR Reference](dgdr.md) —— Spec 参考、生命周期阶段、监控命令
- [DGDR Examples](../components/profiler/profiler-examples.md) —— 适配各种场景的 YAML 示例
- [Profiler Guide](../components/profiler/profiler-guide.md) —— Profiling 算法、模式选择、关卡校验
- [Planner Guide](../components/planner/planner-guide.md) —— 扩缩模式、PlannerConfig 参考
- [Model Caching](model-caching.md) —— PVC 设置与 Model Express
- [Creating Deployments](deployment/create-deployment.md) —— 用于手工编写配置的 DGD spec
- [Multinode Deployments](deployment/multinode-deployment.md) —— Grove、LWS 与多节点细节
- [Disaggregated Communication](disagg-communication-guide.md) —— NIXL、RDMA 与网络
