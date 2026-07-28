---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Planner
---

## 为什么 LLM 推理需要不同的自动扩缩器

传统 Web 服务的扩缩很直白：监控 CPU 或请求速率，高负载时增加副本，低负载时减少副本。HPA、KEDA 等工具在这里非常好用，因为负载与延迟之间大致呈线性关系 —— 请求翻倍就大约意味着 CPU 翻倍，简单的阈值策略就足以维持响应时间稳定。

LLM 推理打破了这些假设：

- **延迟取决于请求内容，而不仅是请求数。** 一条 32K-token prompt 的请求所消耗的算力比一条短请求高出几个数量级。每秒 2 个请求在不同输入/输出序列长度下，对 GPU 的实际负载可能完全不同。
- **prefill 与 decode 的扩缩特性不同。** 在解耦（disaggregated）服务下，prefill 是计算受限（随输入长度伸缩），decode 是内存受限（随并发序列与 KV cache 使用量伸缩）。单一副本数无法刻画两者。
- **真正重要的指标不是标准指标。** 用户关心的 SLA —— TTFT（首字延迟）与 ITL（字间延迟） —— 与 CPU 利用率或请求吞吐没有干净的映射关系。HPA 无法将 "保持 P95 TTFT 在 500ms 以下" 作为目标，因为这需要理解序列长度、GPU 内存压力与延迟之间的关系。
- **扩缩决策代价昂贵。** 拉起一个 GPU worker 需要数分钟而非数秒。过度扩容浪费 GPU 小时的云成本；扩缩不足则违反 SLA。自动扩缩器需要预测需求，而不仅是被动反应。

Dynamo **Planner**（调度器）正是为这些约束量身打造的自动扩缩器。它理解引擎 profiling 数据，跟踪每个 worker 的 GPU 利用率，预测流量模式，并直接面向 TTFT 与 ITL SLA 做扩缩决策 —— 而不是面向代理指标。

## 入门：优化目标（Optimization Targets）

Planner 提供三种 `optimization_target` 设置，用于控制扩缩决策方式：

| 目标 | 描述 | 是否需要 SLA？ | 是否需要 Profiling？ |
|--------|-------------|:-------------:|:-------------------:|
| **`throughput`**（默认） | 通过队列深度与 KV 缓存利用率扩缩，最大化吞吐。引擎饱和时扩容，利用率下降时缩容。 | 否 | 否 |
| **`latency`** | 激进扩容以保持队列短，最小化延迟。在更低的利用率阈值下就开始扩容。 | 否 | 否 |
| **`sla`** | 使用回归式性能模型，针对具体 TTFT/ITL SLA 数值。最精确，但需要配置。 | 是（`ttft_ms`、`itl_ms`） | 推荐 |

**建议从默认的 `throughput` 目标开始** —— 零配置即可工作。若工作负载对延迟敏感，切换到 `latency`；若需要在预部署 profiling 下精确锁定 SLA，则切换到 `sla`。

> **第一次接触 Planner？** 请先阅读 [Planner Guide](planner-guide.md) 获取包含 profiling 与部署的完整流程。

> **需要多 DGD 协调？** 请参阅 [Global Planner Guide](global-planner.md)，了解跨多个 DGD 的共享策略协调以及单 endpoint 多 pool 的部署。

## 扩缩模式（Scaling Modes）

Planner 支持两种可独立运行也可同时启用的扩缩模式：

- **吞吐式扩缩（Throughput-based scaling）**：使用预部署的引擎性能数据（来自 self-benchmark 或 profiler）与流量预测，计算满足 TTFT/ITL 目标所需的副本数。以较长间隔（默认 180s）调整。这是生产部署的主模式。
- **负载式扩缩（Load-based scaling）**：使用来自 Dynamo 事件面的 ForwardPassMetrics（FPM），以在线线性回归来做扩缩决策。无需预部署数据或 KV Router。以短间隔（默认 5s）调整，以便快速响应流量突发。

当两种模式同时启用时，吞吐式扩缩提供容量底线（长期规划），而负载式扩缩负责处理底线之上的实时调整。

## 特性矩阵

| 特性 | 吞吐式 | 负载式 |
|---------|:----------------:|:-------------------------:|
| **部署形态** | | |
| Disaggregated（解耦） | 支持 | 支持 |
| Aggregated（聚合） | 支持 | 支持 |
| **LLM 框架** | | |
| SGLang | 支持 | 支持 |
| TensorRT-LLM | 支持 | 支持 |
| vLLM | 支持 | 支持 |
| **是否需要预部署数据** | 是（self-benchmark 或 profiler） | 否 |
| **负载预测器** | ARIMA、Prophet、Kalman、Constant | 不适用 |
| **路由（Router）** | | |
| 任意（round-robin、random 等） | 支持 | 不支持 |
| KV Router | 支持 | 支持 |
| **Connector** | | |
| KubernetesConnector | 支持 | 支持 |
| VirtualConnector | 支持 | 支持 |

## 何时使用哪种模式

- **吞吐式扩缩** 应在能拿到引擎性能数据时（通过 self-benchmark 或预部署 profiling）始终启用。它提供稳定的、基于预测的容量规划。
- **负载式扩缩** 应在流量突发或难以预测时启用。它能快速响应实时负载变化，且无需预部署数据。
- **两种模式同时启用**：可获得两者的优点。吞吐式扩缩提供下界（长期容量），负载式扩缩处理下界之上的突发。两者都启用时，建议为吞吐式扩缩使用更长的 `--adjustment-interval`。

## 快速上手

### 前置条件

- 已在 Kubernetes 上安装 Dynamo 平台（[安装指南](../../kubernetes/installation-guide.md)）
- 已安装 kube-prometheus-stack（[指标搭建](../../kubernetes/observability/metrics.md)）

### 默认模式（零配置）

Planner 默认开箱可用。默认情况下，`optimization_target` 是 `throughput`，使用基于队列深度与 KV 缓存利用率的静态阈值 —— 无需 SLA 或 profiling：

```yaml
# Minimal planner config — uses throughput optimization by default
features:
  planner:
    mode: disagg
    backend: vllm
```

对延迟敏感的工作负载：

```yaml
features:
  planner:
    mode: disagg
    backend: vllm
    optimization_target: latency
```

### 基于 SLA 的扩缩（进阶）

要在预部署 profiling 下精确锁定 SLA，将 `optimization_target` 设为 `sla`：

```yaml
features:
  planner:
    optimization_target: sla
    enable_throughput_scaling: true
    enable_load_scaling: true
    ttft_ms: 500.0
    itl_ms: 50.0
    pre_deployment_sweeping_mode: rapid
```

进入 SLA 扩缩最快的方式是通过 DynamoGraphDeploymentRequest，它会自动对模型 profiling：

```bash
kubectl apply -f components/src/dynamo/profiler/deploy/profile_sla_aic_dgdr.yaml -n $NAMESPACE
```

完整流程参见 [Planner Guide](planner-guide.md)。

## 当前限制

### 负载式扩缩

负载式扩缩存在以下已知限制。吞吐式扩缩不受其影响。

**需要 ForwardPassMetrics（FPM）。** 负载式扩缩使用通过 Dynamo 事件面（ForwardPassMetrics）传送的、按 engine 按 iteration 的指标。FPM 当前仅 vllm 支持，并在引擎使用 `InstrumentedScheduler` 且设置 `DYN_FORWARDPASS_METRIC_PORT` 时自动启用。负载式扩缩**不**需要 KV Router。

### 通用

**缩容期间的在途请求。** Planner 缩容某个 worker 时，worker 会被直接终止，不会等待在途请求完成。在被终止 worker 上正处于 prefill 阶段的请求将失败。在解耦部署中，正在等待来自被终止 prefill worker 的 KV 缓存传输的 decode worker 也可能受影响。**变通方案：** 把 `--min-endpoint` 设到不会低于稳态流量基线的值，并使用更低的 `--loadbased-scaling-down-sensitivity` 值降低缩容事件的频率。

## 文档

| 文档 | 描述 |
|----------|-------------|
| [Planner Guide](planner-guide.md) | 部署、配置、集成 |
| [Planner Design](../../design-docs/planner-design.md) | 架构与算法内部 |
| [Planner Examples](planner-examples.md) | DGDR YAML 示例、样例配置、进阶模式 |
| [Global Planner Guide](global-planner.md) | 多 DGD 协调、共享 GPU 预算、单 endpoint 多 pool 的部署 |

## 配置参考

### 关键参数

| 参数 | 默认值 | 描述 |
|----------|---------|-------------|
| **通用** | | |
| `--namespace` | `$DYN_NAMESPACE` 或 `dynamo` | Dynamo 逻辑命名空间 |
| `--backend` | `vllm` | 后端框架（`sglang`、`trtllm`、`vllm`） |
| `--mode` | `disagg` | Planner 模式（`disagg`、`prefill`、`decode`、`agg`） |
| `--optimization-target` | `throughput` | 扩缩目标：`throughput`（队列/利用率阈值）、`latency`（激进低延迟）、`sla`（基于回归的 SLA 锁定） |
| `--environment` | `kubernetes` | 部署环境 |
| `ttft_ms` | `500.0` | 目标 TTFT（毫秒） |
| `itl_ms` | `50.0` | 目标 ITL（毫秒） |
| `--max-gpu-budget` | `8` | 跨所有 worker 的最大 GPU 数 |
| `--min-endpoint` | `1` | 每种 worker 的最小副本数 |
| `--decode-engine-num-gpu` | `1` | 每个 decode 引擎的 GPU 数 |
| `--prefill-engine-num-gpu` | `1` | 每个 prefill 引擎的 GPU 数 |
| `--no-operation` | `false` | 观察模式（不实际扩缩） |
| **吞吐式扩缩** | | |
| `--enable-throughput-scaling` | `true` | 启用吞吐式扩缩 |
| `--adjustment-interval` | `180` | 吞吐式扩缩决策的秒数间隔 |
| `--profile-results-dir` | `profiling_results` | profiling 数据路径（NPZ/JSON） |
| `--load-predictor` | `arima` | 预测模型（`arima`、`prophet`、`kalman`、`constant`） |
| **负载式扩缩** | | |
| `--enable-loadbased-scaling` | `false` | 启用负载式扩缩 |
| `--loadbased-adjustment-interval` | `5` | FPM 回归更新与负载式扩缩决策的秒数间隔 |
| `--max-num-fpm-samples` | `64` | 用于回归的 FPM 观测最大保留数 |
| `--fpm-sample-bucket-size` | `16` | 用于观测淘汰的桶数（必须是完全平方数） |
| `--loadbased-scaling-down-sensitivity` | `80` | 缩容敏感度 0-100（0=从不、100=激进） |
| `--loadbased-metric-samples` | `10` | 每个 adjustment interval 的指标采样数 |
| `--loadbased-min-observations` | `5` | 回归生效前的最少观测数 |

### 环境变量

| 变量 | 默认值 | 描述 |
|----------|---------|-------------|
| `DYN_NAMESPACE` | `dynamo` | Dynamo 逻辑命名空间 |
| `DYN_PARENT_DGD_K8S_NAME` | （必填） | 父 DGD K8s 资源名 |
| `PROMETHEUS_ENDPOINT` | `http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090` | Prometheus URL |
| `PLANNER_PROMETHEUS_PORT` | `0`（禁用） | Planner 自身 Prometheus 指标端口 |

## 监控

### Grafana 看板

部署 planner 看板：

```bash
kubectl apply -n monitoring -f deploy/observability/k8s/grafana-planner-dashboard-configmap.yaml
```

看板展示：
- 随时间变化的 worker 数与 GPU 占用
- 观测到的 TTFT、ITL、请求速率、序列长度
- 预测的负载与建议副本数
- FPM 回归模型状态

### Prometheus 指标

设置 `PLANNER_PROMETHEUS_PORT` 后，planner 会暴露自己的指标端点。导出的指标使用 `dynamo_planner_*` 命名约定（下划线与标准单位后缀），取代旧的 `planner:*` 风格。

**吞吐式扩缩** 从集群范围 Prometheus 拉取流量指标：
- 请求计数与时长
- TTFT 与 ITL 分布
- 输入/输出序列长度

**负载式扩缩** 使用 Dynamo 事件面的 ForwardPassMetrics（FPM）：
- 每 iteration 的耗时、调度的 prefill/decode token 数与排队请求状态
- 通过 `FpmEventSubscriber` 投递，自动发现引擎并跟踪生命周期
- 不需要 router `/metrics` 抓取

planner 端口上的核心 gauge 包括副本数（`dynamo_planner_num_prefill_replicas`、`dynamo_planner_num_decode_replicas`）、观测流量（`dynamo_planner_observed_*`）、副本决策（`dynamo_planner_predicted_num_prefill_replicas`、`dynamo_planner_predicted_num_decode_replicas`），以及累计 `dynamo_planner_gpu_hours`。

吞吐预测 gauge `dynamo_planner_predicted_requests_per_second`、`dynamo_planner_predicted_input_sequence_tokens` 与 `dynamo_planner_predicted_output_sequence_tokens` 来自吞吐式扩缩的流量预测，与观测的序列长度指标一同暴露。

#### 诊断指标

附加指标用于看板与离线分析：

- **基于回归的延迟估计：** `dynamo_planner_estimated_ttft_ms` 与 `dynamo_planner_estimated_itl_ms` 反映在线回归在所有引擎上估计的最大 TTFT 与 ITL。
- **引擎容量：** `dynamo_planner_engine_prefill_requests_per_second` 与 `dynamo_planner_engine_decode_requests_per_second` 报告在配置 SLA 下单引擎的 prefill 与 decode 容量。
- **扩缩决策原因：** `dynamo_planner_load_scaling_decision` 与 `dynamo_planner_throughput_scaling_decision` 是 Enum gauge，其状态标签记录了每种模式选择扩容、保持或跳过的原因（例如 `scale_up`、`no_fpm_data`、`set_lower_bound`）。
- **每引擎 FPM 队列深度：** `dynamo_planner_engine_queued_prefill_tokens`、`dynamo_planner_engine_queued_decode_kv_tokens` 与 `dynamo_planner_engine_inflight_decode_kv_tokens` 带有 `worker_id` 与 `dp_rank` 标签。

### HTML 诊断报告

planner 可周期性地输出自包含的 HTML 诊断文件，并嵌入交互式 Plotly 图表。

在 `PlannerConfig` 中（或其等效 YAML / 构造器配置）配置：

- `report_interval_hours`：以**模拟**时间为基础的报告间隔（默认 `24.0` 小时）；设为 `None` 可禁用。
- `report_output_dir`：HTML 文件输出目录（默认 `./planner_reports`）。
- `live_dashboard_port`：实时 HTTP 看板端口（默认 `8080`）。设为 `0` 可禁用。aiohttp 服务器会在该端口启动，并将当前累计的快照数据以交互式 Plotly 报告形式提供在 `http://<host>:<port>/`。与周期性报告不同，实时看板**不会**清空快照 —— 它始终显示自上一次周期性报告（或自启动起，如周期性报告被禁用）以来累计的全部数据。

报告基于每个 tick 的快照聚合，并使用 `TickInput.now_s` 作为时间戳，因此在线（wall clock）运行与使用模拟时钟的 **replay** 中表现一致。典型图表覆盖 worker 数、观测延迟与估计延迟相对 SLA 目标、请求速率、引擎容量、扩缩决策时间线，以及输入/输出序列长度。
