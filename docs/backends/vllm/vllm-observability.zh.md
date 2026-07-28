---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Prometheus
---

## 概述

通过 Dynamo 运行 vLLM 时，vLLM 引擎指标（metrics）会自动透传并在 Dynamo 的 `/metrics` 端点（默认端口 8081）上暴露。这样你可以从同一个 worker 后端（backend）端点同时访问 vLLM 引擎指标（前缀 `vllm:`）与 Dynamo 运行时（runtime）指标（前缀 `dynamo_*`）。

**关于 vLLM 全部指标的完整且权威列表**，请始终参阅 [官方 vLLM Metrics Design 文档](https://docs.vllm.ai/en/stable/design/metrics.html)。

**关于 LMCache 指标与集成**，参见 [LMCache 集成指南](../../integrations/lmcache-integration.md)。

**关于 Dynamo 运行时指标**，参见 [Dynamo Metrics Guide](../../observability/metrics.md)。

**关于可视化（observability）搭建说明**，参见 [Prometheus and Grafana Setup Guide](../../observability/prometheus-grafana.md)。

## 环境变量与 flag

| 变量 | 说明 | 默认值 | 示例 |
|----------|-------------|---------|---------|
| `DYN_SYSTEM_PORT` | 系统指标/健康端口。要暴露 `/metrics` 端点必须设置。 | `-1`（禁用） | `8081` |

## 快速开始

这是一个单机示例。

### 启动可观测性（observability）栈

如需用 Prometheus 与 Grafana 可视化指标，请启动可观测性栈。说明参见 [Observability Getting Started](../../observability/README.md#getting-started-quickly)。

### 启动 Dynamo 组件

`examples/backends/vllm/launch/` 中的启动脚本默认已启用 8081 端口的指标。例如：

```bash
cd $DYNAMO_HOME/examples/backends/vllm
bash launch/agg.sh
```

部署运行后，发送请求并查看指标：

```bash
curl -s localhost:8081/metrics | grep "^vllm:"
```

## 暴露的指标

vLLM 在 `/metrics` HTTP 端点上以 Prometheus Exposition 文本格式暴露指标。所有 vLLM 引擎指标都使用 `vllm:` 前缀，并带有用于标识来源的标签（如 `model_name`、`finished_reason`、`scheduling_event`）。

**Prometheus Exposition 文本格式示例：**

```
# HELP vllm:request_success_total Number of successfully finished requests.
# TYPE vllm:request_success_total counter
vllm:request_success_total{finished_reason="length",model_name="meta-llama/Llama-3.1-8B"} 15.0
vllm:request_success_total{finished_reason="stop",model_name="meta-llama/Llama-3.1-8B"} 150.0

# HELP vllm:time_to_first_token_seconds Histogram of time to first token in seconds.
# TYPE vllm:time_to_first_token_seconds histogram
vllm:time_to_first_token_seconds_bucket{le="0.001",model_name="meta-llama/Llama-3.1-8B"} 0.0
vllm:time_to_first_token_seconds_bucket{le="0.005",model_name="meta-llama/Llama-3.1-8B"} 5.0
vllm:time_to_first_token_seconds_count{model_name="meta-llama/Llama-3.1-8B"} 165.0
vllm:time_to_first_token_seconds_sum{model_name="meta-llama/Llama-3.1-8B"} 89.38
```

**注意：** 上面展示的具体指标只是示例，会因 vLLM 版本而异。请始终查看你实际的 `/metrics` 端点，或参考[官方文档](https://docs.vllm.ai/en/stable/design/metrics.html)以获取当前列表。

### 指标分类

vLLM 提供以下分类的指标（全部使用 `vllm:` 前缀）：

- **请求指标** - 请求成功、失败与完成跟踪
- **性能指标** - 延迟、吞吐与时序测量
- **资源使用** - 系统资源消耗
- **scheduler（调度器）指标** - 调度与队列管理
- **解耦（disaggregation）指标** - 解耦部署专用的指标（启用时）

**注意：** 具体指标在 vLLM 不同版本之间可能变化。请始终参阅[官方文档](https://docs.vllm.ai/en/stable/design/metrics.html)，或检查你 vLLM 版本的 `/metrics` 端点。

## 可用指标

官方 vLLM 文档包含完整的指标定义，包括：
- 详细解释与设计动机
- Counter、Gauge 与 Histogram 指标类型
- 指标标签（如 `model_name`、`finished_reason`、`scheduling_event`）
- v1 指标迁移信息
- 后续工作与已废弃指标

关于 vLLM 全部指标的完整且权威列表，参见 [官方 vLLM Metrics Design 文档](https://docs.vllm.ai/en/stable/design/metrics.html)。

## LMCache 指标

启用 LMCache 后，LMCache 指标（前缀 `lmcache:`）会与 vLLM 和 Dynamo 指标一起，通过 Dynamo 的 `/metrics` 端点自动暴露。

体验方式是使用 LMCache 启动脚本：

```bash
cd $DYNAMO_HOME/examples/backends/vllm
bash launch/agg_lmcache.sh
```

发送请求并查看 LMCache 指标：

```bash
curl -s localhost:8081/metrics | grep "^lmcache:"
```

### 故障排查

LMCache 相关指标与日志的故障排查（包括 `PrometheusLogger instance already created with different metadata` 与 `PROMETHEUS_MULTIPROC_DIR` 警告）记录在：

- [LMCache 集成指南](../../integrations/lmcache-integration.md#troubleshooting)

**关于完整的 LMCache 配置与指标细节**，参见：
- [LMCache 集成指南](../../integrations/lmcache-integration.md) - 搭建与配置
- [LMCache Observability 文档](https://docs.lmcache.ai/production/observability/vllm_endpoint.html) - 完整指标参考

## 实现细节

- vLLM v1 通过 `prometheus_client.multiprocess` 进行多进程指标采集
- `PROMETHEUS_MULTIPROC_DIR`：（可选）。Dynamo 默认会自动管理该环境变量，把它设为一个临时目录，多进程指标会以内存映射文件方式存储其中。每个 worker 进程将其指标写到目录下不同的文件，在抓取 `/metrics` 时进行聚合。仅当用户需要完全控制指标目录时才需要显式设置该变量。
- Dynamo 使用 `MultiProcessCollector` 聚合所有 worker 进程的指标
- 暴露前会按 `vllm:` 与 `lmcache:` 前缀过滤（启用 LMCache 时）
- 该集成使用 Dynamo 的 `register_engine_metrics_callback()` 函数与全局 `REGISTRY`
- 指标会在 vLLM 引擎初始化完成后出现
- vLLM v1 指标与 v0 不同 —— 迁移细节见 [官方文档](https://docs.vllm.ai/en/stable/design/metrics.html)

## 相关文档

### vLLM 指标
- [官方 vLLM Metrics Design 文档](https://docs.vllm.ai/en/stable/design/metrics.html)
- [vLLM Production Metrics 用户指南](https://docs.vllm.ai/en/stable/usage/metrics.html)
- [vLLM GitHub - 指标实现](https://github.com/vllm-project/vllm/tree/main/vllm/v1/metrics)

### Dynamo 指标
- [Dynamo Metrics Guide](../../observability/metrics.md) - Dynamo 运行时指标的完整文档
- [Prometheus and Grafana Setup](../../observability/prometheus-grafana.md) - 可视化搭建说明
- Dynamo 运行时指标（前缀 `dynamo_*`）与 vLLM 指标一起在同一个 `/metrics` 端点上提供
  - 实现：`lib/runtime/src/metrics.rs`（Rust 运行时指标）
  - 指标名称：`lib/runtime/src/metrics/prometheus_names.rs`（指标名常量）
  - 集成代码：`components/src/dynamo/common/utils/prometheus.py` - Prometheus 工具与回调注册
