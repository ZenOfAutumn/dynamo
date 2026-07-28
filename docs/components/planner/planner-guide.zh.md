---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Planner Guide
---

Dynamo Planner 是一个自动扩缩容控制器，可在运行时调整预填充（prefill）与解码（decode）引擎副本数量，以满足延迟 SLA。它读取流量信号（Prometheus 指标或负载预测器输出）和引擎性能档案，决定何时扩容或缩容。

如需快速概览，请参阅 [Planner 概述](README.md)。如需了解架构内部细节，请参阅 [Planner 设计](../../design-docs/planner-design.md)。

## 扩缩容模式

Planner 支持三种优化目标，决定扩缩容决策的方式：

- **`throughput`**（默认）：使用基于队列深度和 KV 缓存（KV cache）利用率的静态阈值。无需 SLA 目标或性能剖析数据。开箱即用。
- **`latency`**：与 `throughput` 思路相同，但阈值更激进——更早扩容、更不能容忍排队。适用于对延迟敏感的工作负载。
- **`sla`**：使用基于回归的性能模型，并指定具体的 TTFT/ITL 目标。同时支持基于吞吐（预测式）和基于负载（反应式）的扩缩容模式。适用于需要精细控制 SLA 的高级用户。

**何时选择何种模式：**

- 优先从 **`throughput`**（默认）开始——无需配置即可立即使用。
- 如果工作负载有严格的延迟要求，且你倾向于过度配置而非排队，则切换到 **`latency`**。
- 当你拥有部署前剖析数据，并希望以特定 TTFT/ITL 值为目标时，使用 **`sla`**。

## PlannerConfig 参考

Planner 通过 `PlannerConfig` JSON/YAML 对象进行配置。当与 profiler 一起使用时，该配置位于 DGDR 规约的 `features.planner` 部分：

```yaml
features:
  planner:
    mode: disagg
    backend: vllm
    # optimization_target defaults to "throughput" — works out of the box
```

基于 SLA 的扩缩容：

```yaml
features:
  planner:
    optimization_target: sla
    enable_throughput_scaling: true
    enable_load_scaling: false
    pre_deployment_sweeping_mode: rapid
    mode: disagg
    backend: vllm
```

### 优化目标

| 字段 | 类型 | 默认值 | 说明 |
|-------|------|---------|-------------|
| `optimization_target` | string | `throughput` | `throughput`：基于队列/利用率阈值扩缩容。`latency`：激进的低延迟阈值。`sla`：以 ttft_ms/itl_ms 为目标的回归式扩缩容。 |

当 `optimization_target` 为 `throughput` 或 `latency` 时，自动启用基于负载的扩缩容，并禁用基于吞吐的扩缩容。`ttft_ms`/`itl_ms` 字段被忽略。

### 扩缩容模式字段（SLA 模式）

| 字段 | 类型 | 默认值 | 说明 |
|-------|------|---------|-------------|
| `enable_throughput_scaling` | bool | `true` | 启用基于吞吐的扩缩容（需要部署前性能数据）。仅在 `optimization_target: sla` 时使用。 |
| `enable_load_scaling` | bool | `false` | 启用基于负载的扩缩容。仅在 `optimization_target: sla` 时使用。 |

使用 `optimization_target: sla` 时，必须至少启用一种扩缩容模式。

### 部署前 Sweeping

| 字段 | 类型 | 默认值 | 说明 |
|-------|------|---------|-------------|
| `pre_deployment_sweeping_mode` | string | `rapid` | 如何生成引擎性能数据：`rapid`（AIC 仿真，~30 秒）、`thorough`（真实 GPU，2-4 小时）或 `none`（跳过）。 |

启用基于吞吐的扩缩容时，Planner 需要引擎性能数据。在启动时，它首先尝试从 `get_perf_metrics` Dynamo 端点（参见 PR #7779）获取自基准结果。如不可用，则回退至 `profile_results_dir` 下由 profiler 生成的数据（npz 或 JSON）。两种来源都会被转换为 ForwardPassMetrics 并送入 FPM 回归模型。

### 基于吞吐的扩缩容设置

| 字段 | 类型 | 默认值 | 说明 |
|-------|------|---------|-------------|
| `throughput_adjustment_interval_seconds` | int | `180` | 基于吞吐的扩缩容决策间隔（秒）。 |
| `min_endpoint` | int | `1` | 维持的最小引擎端点数。 |
| `max_gpu_budget` | int | `8` | Planner 可分配的 GPU 总数上限。 |
| `ttft_ms` | float | `500.0` | 用于扩缩容决策的 TTFT SLA 目标（毫秒）。 |
| `itl_ms` | float | `50.0` | 用于扩缩容决策的 ITL SLA 目标（毫秒）。 |

### 基于负载的扩缩容设置

| 字段 | 类型 | 默认值 | 说明 |
|-------|------|---------|-------------|
| `load_adjustment_interval_seconds` | int | `5` | FPM 回归更新和基于负载的扩缩容决策的间隔（秒）。即便仅启用了吞吐扩缩容，实时 FPM 观测也以此间隔送入回归。必须短于 `throughput_adjustment_interval_seconds`。 |
| `max_num_fpm_samples` | int | `64` | 用于回归保留的最大 FPM 观测数。 |
| `fpm_sample_bucket_size` | int | `16` | 观测淘汰的桶数（必须为完全平方数）。 |
| `load_scaling_down_sensitivity` | int | `80` | 缩容敏感度 0–100（0=从不，100=激进）。 |
| `load_metric_samples` | int | `10` | 每次决策收集的指标样本数。 |
| `load_min_observations` | int | `5` | 进行扩缩容决策前的最小观测数。 |

### 通用设置

| 字段 | 类型 | 默认值 | 说明 |
|-------|------|---------|-------------|
| `mode` | string | `disagg` | Planner 模式：`disagg`、`prefill`、`decode` 或 `agg`。 |
| `backend` | string | `vllm` | 后端：`vllm`、`sglang`、`trtllm` 或 `mocker`。 |
| `environment` | string | `kubernetes` | 运行时环境：`kubernetes`、`virtual` 或 `global-planner`。 |
| `namespace` | string | env `DYN_NAMESPACE` | 部署所在的 Kubernetes 命名空间。 |

### 流量预测设置

| 字段 | 类型 | 默认值 | 说明 |
|-------|------|---------|-------------|
| `load_predictor` | string | `arima` | 预测方法：`constant`、`arima`、`kalman` 或 `prophet`。 |
| `load_predictor_log1p` | bool | `false` | 在预测前对负载数据应用 log1p 变换。 |
| `prophet_window_size` | int | `50` | Prophet 预测器的窗口大小（秒）。 |
| `load_predictor_warmup_trace` | string | `null` | 用于引导预测的预热 trace 文件路径。 |

### 卡尔曼滤波器设置

| 字段 | 类型 | 默认值 | 说明 |
|-------|------|---------|-------------|
| `kalman_q_level` | float | `1.0` | level 分量的过程噪声。 |
| `kalman_q_trend` | float | `0.1` | trend 分量的过程噪声。 |
| `kalman_r` | float | `10.0` | 测量噪声。 |
| `kalman_min_points` | int | `5` | 启用卡尔曼预测前所需的最小数据点数。 |

### 诊断报告

| 字段 | 类型 | 默认值 | 说明 |
|-------|------|---------|-------------|
| `report_interval_hours` | float 或 `null` | `24.0` | 每隔 N 小时（仿真时间）生成一份 HTML 诊断报告。设置为 `null` 可禁用周期性报告生成。 |
| `report_output_dir` | string | `./planner_reports` | HTML 诊断报告所在目录。 |
| `live_dashboard_port` | int | `8080` | 实时诊断仪表盘 HTTP 服务器端口。设为 `0` 可禁用。启用时，访问 `http://host:port/` 可查看累积快照的实时 Plotly 报告。 |

报告中显现的诊断信号同样以 `dynamo_planner_*` 前缀导出为 Prometheus 指标——例如估算的 TTFT/ITL（`dynamo_planner_estimated_ttft_ms`、`dynamo_planner_estimated_itl_ms`）、按引擎的容量与 FPM 队列深度，以及负载/吞吐扩缩容决策枚举。

## 与 Profiler 的集成

当 profiler 启用 planner 运行时，它会：

1. 选择最佳的 prefill 与 decode 引擎配置
2. 生成引擎性能数据（prefill TTFT 与 ISL 关系、decode ITL 与 KV 缓存利用率关系）
3. 将 `PlannerConfig` 与性能数据保存到独立的 Kubernetes ConfigMap 中
4. 将 planner 服务添加到生成的 DGD 中，并配置为从这些 ConfigMap 读取

Planner 通过 `--config /path/to/planner_config.json` 接收其配置，该路径由 `planner-config-XXXX` ConfigMap 挂载。性能剖析数据由 `planner-profile-data-XXXX` ConfigMap 挂载。

完整的剖析工作流以及如何配置部署前 sweeping，请参阅 [Profiler 指南](../profiler/profiler-guide.md)。

## 分层部署

如果你想为某个模型对外暴露一个公共端点，但又有多个针对不同请求类别优化的私有 DGD，可使用分层部署：

- 一个包含 `Frontend`、`GlobalRouter` 与 `GlobalPlanner` 的控制 DGD
- 一个或多个 prefill 池 DGD
- 一个或多个 decode 池 DGD

在当前工作流中，需为每个目标池独立运行剖析，然后手动组合最终的控制 DGD 加各池 DGD。参见 [Global Planner 指南](global-planner.md)。

## 另请参阅

- [Planner 概述](README.md) —— 为什么 LLM 推理需要不同的自动扩缩容器
- [Planner 设计](../../design-docs/planner-design.md) —— 架构与算法内部细节
- [Planner 示例](planner-examples.md) —— DGDR YAML 示例、配置范例、高级模式
- [Global Planner 指南](global-planner.md) —— 多 DGD 协调、共享 GPU 预算、单端点多池部署
- [Profiler 指南](../profiler/profiler-guide.md) —— 性能剖析数据的生成方式
