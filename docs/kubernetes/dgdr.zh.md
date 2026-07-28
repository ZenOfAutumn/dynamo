---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: DGDR Reference
---

`DynamoGraphDeploymentRequest`（DGDR）是 Dynamo 的 **按意图部署（deploy-by-intent）** API。
你只需描述要运行的内容以及性能目标，由 profiler
确定最优配置并创建实际部署。

完整的模型部署流程分步指南——包括策略选择、模型缓存、planner 配置以及常见陷阱——
请参见 [Model Deployment Guide](model-deployment-guide.md)。

## DGDR vs DGD

Dynamo 提供两种用于部署推理图的自定义资源（CR）：

| | DGDR（推荐） | DGD（手动） |
|---|---|---|
| **你需要提供** | 模型 + 可选的 SLA 目标 | 完整的部署规约（并行度、副本数、资源限制等） |
| **profiling** | 自动化——遍历多种配置寻找最优组合 | 无——使用你自带的配置 |
| **硬件可移植性** | 适配集群中现有的任意 GPU | 与你配置时的硬件绑定 |
| **最适用于** | 多数部署场景、SLA 驱动的优化 | 已知良好配置、固定 recipe |

**何时改用 DGD**：当你针对特定 模型/硬件 组合手工编写了配置（例如来自 `recipes/` 目录）时使用 DGD。
这些配置在已知环境下可能更优，但要求理解何种并行参数（TP、PP、EP）合适，且无法跨硬件通用。

DGD 部署详情请参见 [Creating Deployments](deployment/create-deployment.md)。

## Spec 参考

### 最小示例

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: my-model
spec:
  model: Qwen/Qwen3-0.6B
  image: "nvcr.io/nvidia/ai-dynamo/dynamo-planner:1.0.0"
```

### 字段参考

| 字段 | 必填 | 默认值 | 用途 |
|---|---|---|---|
| `model` | 是 | — | HuggingFace 模型 ID（例如 `Qwen/Qwen3-0.6B`） |
| `image` | 否 | — | profiling 任务的容器镜像。Dynamo >= 1.1.0：使用 `dynamo-planner`；更早版本：使用 `dynamo-frontend`。 |
| `backend` | 否 | `auto` | 推理引擎：`auto`、`vllm`、`sglang`、`trtllm` |
| `searchStrategy` | 否 | `rapid` | profiling 深度：`rapid`（AIC 仿真，约 30s）或 `thorough`（真实 GPU，2–4 小时） |
| `autoApply` | 否 | `true` | 自动部署 profiler 推荐的配置 |
| `sla.ttft` | 否 | — | 首 token 延迟（time to first token，ms）目标 |
| `sla.itl` | 否 | — | token 间延迟（inter-token latency，ms）目标 |
| `sla.e2eLatency` | 否 | — | 端到端延迟目标（ms）。不能与显式 `ttft`/`itl` 同时使用。 |
| `workload.isl` | 否 | `4000` | 期望的平均输入序列长度 |
| `workload.osl` | 否 | `1000` | 期望的平均输出序列长度 |
| `workload.requestRate` | 否 | — | 目标每秒请求数 |
| `workload.concurrency` | 否 | — | 目标并发请求数 |
| `hardware.gpuSku` | 否 | 自动检测 | GPU SKU（参见 [SKU 格式](#sku-format)） |
| `hardware.vramMb` | 否 | 自动检测 | GPU 显存（MB） |
| `hardware.totalGpus` | 否 | 自动检测（上限 32） | 部署可用的 GPU 总数 |
| `hardware.numGpusPerNode` | 否 | 自动检测 | 每个节点的 GPU 数 |
| `hardware.interconnect` | 否 | 自动检测 | 互联类型 |
| `hardware.rdma` | 否 | 自动检测 | 是否可用 RDMA |
| `modelCache.pvcName` | 否 | — | 包含已缓存模型权重的 `ReadWriteMany` PVC 名称 |
| `modelCache.pvcModelPath` | 否 | — | PVC 内模型目录的路径 |
| `modelCache.pvcMountPath` | 否 | `/opt/model-cache` | 容器内挂载路径 |
| `features.planner` | 否 | 禁用 | 启用 SLA 感知 Planner（原始 JSON 配置） |
| `features.mocker` | 否 | 禁用 | 启用 mocker 模式以便测试 |
| `overrides.profilingJob` | 否 | — | profiling 任务的 `batchv1.JobSpec` 覆盖（例如 tolerations） |
| `overrides.dgd` | 否 | — | 应用于生成部署的原始 DGD 覆盖基底 |

完整的 CRD spec 请参见 [API Reference](api-reference.md)。

### SKU 格式

手动提供硬件配置时，请使用小写下划线格式：

| 正确 | 错误 |
|---|---|
| `h100_sxm` | `H100-SXM5-80GB` |
| `h200_sxm` | `H200-SXM-141GB` |
| `a100_sxm` | `A100-SXM4-80GB` |
| `a30` | `A30` |
| `l40s` | `L40S` |

支持的全部取值：`gb200_sxm`、`b200_sxm`、`h200_sxm`、`h100_sxm`、
`h100_pcie`、`a100_sxm`、`a100_pcie`、`a30`、`l40s`、`l40`、`l4`、
`v100_sxm`、`v100_pcie`、`t4`、`mi200`、`mi300`。

> [!NOTE]
> 并非所有 SKU 都受 AIC profiler 的 `rapid` 模式支持。详见
> [AIC Support Matrix](model-deployment-guide.md#aic-support-matrix)。

## 生命周期

创建 DGDR 后，它将经历以下阶段：

| 阶段 | 含义 |
|---|---|
| `Pending` | spec 校验通过；operator 正在发现 GPU 硬件并准备 profiling 任务 |
| `Profiling` | profiling 任务运行中——子阶段：`Initializing`、`SweepingPrefill`、`SweepingDecode`、`SelectingConfig`、`BuildingCurves`、`GeneratingDGD`、`Done` |
| `Ready` | profiling 完成；最优配置存放于 `.status.profilingResults.selectedConfig`。当 `autoApply: false` 时为终态。 |
| `Deploying` | 正在创建 `DynamoGraphDeployment`（仅当 `autoApply: true`） |
| `Deployed` | DGD 已运行且健康 |
| `Failed` | 不可恢复的错误——profiling 失败不会重试（`backoffLimit: 0`）；可查看事件和 conditions 获取详情 |

### Conditions

operator 在 DGDR 状态中维护以下 conditions：

| Condition | 含义 |
|---|---|
| `Validation` | spec 校验通过或失败 |
| `Profiling` | profiling 任务运行中、成功或失败 |
| `SpecGenerated` | 生成的 DGD spec 可用 |
| `DeploymentReady` | DGD 已部署且健康 |
| `Succeeded` | 聚合 condition——当 DGDR 达到目标状态时为 true |

### 监控

```bash
# Watch phase transitions
kubectl get dgdr my-model -n $NAMESPACE -w

# Detailed status, conditions, and events
kubectl describe dgdr my-model -n $NAMESPACE

# Profiling sub-phase
kubectl get dgdr my-model -n $NAMESPACE -o jsonpath='{.status.profilingPhase}'

# Profiling job logs
kubectl get pods -n $NAMESPACE -l nvidia.com/dgdr-name=my-model
kubectl logs -f <profiling-pod-name> -n $NAMESPACE

# View generated DGD spec (when autoApply: false)
kubectl get dgdr my-model -n $NAMESPACE \
  -o jsonpath='{.status.profilingResults.selectedConfig}' | python3 -m json.tool

# View Pareto-optimal configs from profiling
kubectl get dgdr my-model -n $NAMESPACE \
  -o jsonpath='{.status.profilingResults.pareto}'
```

### 资源所有权

- DGDR **不会** 在其创建的 DGD 上设置 owner reference。删除
  DGDR 不会删除 DGD——DGD 独立存在以便继续提供推理服务。
- 二者关系通过 label 跟踪：`dgdr.nvidia.com/name` 与
  `dgdr.nvidia.com/namespace`。
- 其他附属资源（planner ConfigMap）创建在同一 namespace
  下并打上 `dgdr.nvidia.com/name` label。

## 延伸阅读

- [Model Deployment Guide](model-deployment-guide.md) — 如何部署模型、策略选择、陷阱、示例
- [Profiler Guide](../components/profiler/profiler-guide.md) — profiling 算法、模式选择、gate 检查
- [Profiler Examples](../components/profiler/profiler-examples.md) — 针对 SLA 目标、私有模型、MoE、覆盖等的现成 YAML
- [Planner Guide](../components/planner/planner-guide.md) — 扩缩容模式、PlannerConfig 参考
- [API Reference](api-reference.md) — 完整 CRD 字段规约
- [Creating Deployments](deployment/create-deployment.md) — 完全手动控制的 DGD spec
