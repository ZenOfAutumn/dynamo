---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Prometheus
---

关于 TensorRT-LLM 的常规功能与配置，请参阅 [Reference Guide](trtllm-reference-guide.md)。

---

## 概述

通过 Dynamo 运行 TensorRT-LLM 时，TensorRT-LLM 的 Prometheus 指标（metrics）会自动透传，并暴露在 Dynamo 的 `/metrics` 端点上（默认端口 8081）。这样，你可以从同一个 worker backend 端点访问 TensorRT-LLM 引擎指标（带 `trtllm_` 前缀）以及 Dynamo runtime 指标（带 `dynamo_*` 前缀）。

更多性能指标可通过非 Prometheus API 获取（参见下面的 [非 Prometheus 性能指标](#non-prometheus-performance-metrics) 一节）。

截至本文档撰写时，所附 TensorRT-LLM 1.1.0rc5 暴露了 **5 个基础 Prometheus 指标**。注意：`trtllm_` 前缀是由 Dynamo 添加的。

**关于 Dynamo runtime 指标**，请参阅 [Dynamo Metrics 指南](../../observability/metrics.md)。

**关于可视化的搭建步骤**，请参阅 [Prometheus 与 Grafana 搭建指南](../../observability/prometheus-grafana.md)。

## 环境变量

| 变量 | 描述 | 默认值 | 示例 |
|----------|-------------|---------|---------|
| `DYN_SYSTEM_PORT` | 系统指标 / 健康检查端口 | `-1`（关闭） | `8081` |

## 快速开始

下面的示例为单机环境。

### 启动可观测性栈

要使用 Prometheus 与 Grafana 进行指标可视化，请先启动可观测性（observability）栈。具体步骤请参阅 [Observability Getting Started](../../observability/README.md#getting-started-quickly)。

### 启动 Dynamo 组件

启动一个前端（frontend）和一个 TensorRT-LLM 后端（backend）以测试指标：

```bash
# Start frontend (default port 8000, override with --http-port or DYN_HTTP_PORT env var)
$ python -m dynamo.frontend

# Enable system metrics server on port 8081 and enable metrics collection
$ DYN_SYSTEM_PORT=8081 python -m dynamo.trtllm --model <model_name> --publish-events-and-metrics
```

**注意：** 启用指标采集时 `backend` 必须设为 `"pytorch"`（在 `components/src/dynamo/trtllm/main.py` 中强制要求）。TensorRT-LLM 的 `MetricsCollector` 集成目前仅在 PyTorch backend 上做过测试与验证。

等 TensorRT-LLM worker 启动完成后，发送请求并查看指标：

```bash
# Send a request
curl -H 'Content-Type: application/json' \
-d '{
  "model": "<model_name>",
  "max_completion_tokens": 100,
  "messages": [{"role": "user", "content": "Explain why Roger Federer is considered one of the greatest tennis players of all time"}]
}' \
http://localhost:8000/v1/chat/completions

# Check metrics from the worker
curl -s localhost:8081/metrics | grep "^trtllm_"
```

## 暴露的指标

TensorRT-LLM 在 `/metrics` HTTP 端点上以 Prometheus Exposition Format 文本暴露指标。所有 TensorRT-LLM 引擎指标使用 `trtllm_` 前缀，并附带标签（如 `model_name`、`engine_type`、`finished_reason`）以标识来源。

**注意：** TensorRT-LLM 使用 `model_name` 而非 Dynamo 标准的 `model` 标签。

**Prometheus Exposition Format 文本示例：**

```
# HELP trtllm_request_success_total Count of successfully processed requests.
# TYPE trtllm_request_success_total counter
trtllm_request_success_total{model_name="Qwen/Qwen3-0.6B",engine_type="trtllm",finished_reason="stop"} 150.0
trtllm_request_success_total{model_name="Qwen/Qwen3-0.6B",engine_type="trtllm",finished_reason="length"} 5.0

# HELP trtllm_time_to_first_token_seconds Histogram of time to first token in seconds.
# TYPE trtllm_time_to_first_token_seconds histogram
trtllm_time_to_first_token_seconds_bucket{le="0.01",model_name="Qwen/Qwen3-0.6B",engine_type="trtllm"} 0.0
trtllm_time_to_first_token_seconds_bucket{le="0.05",model_name="Qwen/Qwen3-0.6B",engine_type="trtllm"} 12.0
trtllm_time_to_first_token_seconds_count{model_name="Qwen/Qwen3-0.6B",engine_type="trtllm"} 150.0
trtllm_time_to_first_token_seconds_sum{model_name="Qwen/Qwen3-0.6B",engine_type="trtllm"} 8.75

# HELP trtllm_e2e_request_latency_seconds Histogram of end to end request latency in seconds.
# TYPE trtllm_e2e_request_latency_seconds histogram
trtllm_e2e_request_latency_seconds_bucket{le="0.5",model_name="Qwen/Qwen3-0.6B",engine_type="trtllm"} 25.0
trtllm_e2e_request_latency_seconds_count{model_name="Qwen/Qwen3-0.6B",engine_type="trtllm"} 150.0
trtllm_e2e_request_latency_seconds_sum{model_name="Qwen/Qwen3-0.6B",engine_type="trtllm"} 45.2

# HELP trtllm_time_per_output_token_seconds Histogram of time per output token in seconds.
# TYPE trtllm_time_per_output_token_seconds histogram
trtllm_time_per_output_token_seconds_bucket{le="0.1",model_name="Qwen/Qwen3-0.6B",engine_type="trtllm"} 120.0
trtllm_time_per_output_token_seconds_count{model_name="Qwen/Qwen3-0.6B",engine_type="trtllm"} 150.0
trtllm_time_per_output_token_seconds_sum{model_name="Qwen/Qwen3-0.6B",engine_type="trtllm"} 12.5

# HELP trtllm_request_queue_time_seconds Histogram of time spent in WAITING phase for request.
# TYPE trtllm_request_queue_time_seconds histogram
trtllm_request_queue_time_seconds_bucket{le="1.0",model_name="Qwen/Qwen3-0.6B",engine_type="trtllm"} 140.0
trtllm_request_queue_time_seconds_count{model_name="Qwen/Qwen3-0.6B",engine_type="trtllm"} 150.0
trtllm_request_queue_time_seconds_sum{model_name="Qwen/Qwen3-0.6B",engine_type="trtllm"} 32.1
```

**注意：** 上面展示的具体指标只是示例，可能因 TensorRT-LLM 版本而异。请始终以你实际 `/metrics` 端点的输出为准。

### 指标分类

TensorRT-LLM 提供以下几类指标（均带 `trtllm_` 前缀）：

- **请求指标** - 请求成功统计与延迟测量
- **性能指标** - 首 token 时间（TTFT）、每个输出 token 的时间（TPOT）和排队时间

**注意：** 不同 TensorRT-LLM 版本之间指标可能会变化。请始终以你版本的 `/metrics` 端点为准。

## 可用指标

下列指标会通过 Dynamo 的 `/metrics` 端点暴露（前缀 `trtllm_` 由 Dynamo 添加），适用于 TensorRT-LLM 1.1.0rc5：

- `trtllm_request_success_total`（Counter）—— 按结束原因统计已成功处理的请求总数
  - 标签：`model_name`、`engine_type`、`finished_reason`
- `trtllm_e2e_request_latency_seconds`（Histogram）—— 端到端请求延迟（秒）
  - 标签：`model_name`、`engine_type`
- `trtllm_time_to_first_token_seconds`（Histogram）—— 首 token 时间 TTFT（秒）
  - 标签：`model_name`、`engine_type`
- `trtllm_time_per_output_token_seconds`（Histogram）—— 每个输出 token 的时间 TPOT（秒）
  - 标签：`model_name`、`engine_type`
- `trtllm_request_queue_time_seconds`（Histogram）—— 请求在队列中等待的时间（秒）
  - 标签：`model_name`、`engine_type`

随 TensorRT-LLM 版本更新，这些指标的名称与可用性可能发生变化。

TensorRT-LLM 通过 `MetricsCollector` 类提供 Prometheus 指标（参见 [tensorrt_llm/metrics/collector.py](https://github.com/NVIDIA/TensorRT-LLM/blob/main/tensorrt_llm/metrics/collector.py)）。

### 额外的运维指标

Dynamo 为 TensorRT-LLM worker 额外添加了以下运维指标，用以补充引擎原生指标所缺乏的请求级可观测性。所有指标均带 `trtllm_` 前缀，并在指定 `--publish-events-and-metrics` 时自动启用。

指标名常量定义在 `lib/runtime/src/metrics/prometheus_names.rs` 的 `trtllm_additional` 模块中。

#### 请求类型追踪

- `trtllm_request_type_image_total`（Counter）—— 包含图像/多模态内容的请求总数
  - 标签：`model_name`、`disaggregation_mode`、`engine_type`
- `trtllm_request_type_structured_output_total`（Counter）—— 使用受引导/结构化解码（JSON、regex、grammar 等）的请求总数
  - 标签：`model_name`、`disaggregation_mode`、`engine_type`

#### 中止追踪

- `trtllm_num_aborted_requests_total`（Counter）—— 被中止/取消的请求总数
  - 标签：`model_name`、`disaggregation_mode`、`engine_type`

#### KV 缓存传输指标（解耦部署）

这些指标只在解耦（prefill + decode）部署中、且实际发生 KV 缓存（KV cache）传输时记录。数据来自 TensorRT-LLM 的 `RequestPerfMetrics.timing_metrics`。

- `trtllm_kv_transfer_success_total`（Counter）—— 成功的 KV 缓存传输总数（在 prefill 侧记录）
  - 标签：`model_name`、`disaggregation_mode`、`engine_type`
- `trtllm_kv_transfer_latency_seconds`（Histogram）—— 每个请求的 KV 缓存传输延迟（秒）
  - 标签：`model_name`、`disaggregation_mode`、`engine_type`
- `trtllm_kv_transfer_bytes`（Histogram）—— 每个请求的 KV 缓存传输大小（字节）
  - 标签：`model_name`、`disaggregation_mode`、`engine_type`
  - Buckets：100KB、500KB、1MB、5MB、10MB、50MB、100MB、500MB、1GB、5GB
- `trtllm_kv_transfer_speed_gb_s`（Histogram）—— 每个请求的 KV 缓存传输速率（GB/s）
  - 标签：`model_name`、`disaggregation_mode`、`engine_type`

## 非 Prometheus 性能指标

除基础 Prometheus 指标外，TensorRT-LLM 还提供大量性能数据。这些数据目前不会暴露给 Prometheus。

### 通过代码获取

- **RequestPerfMetrics 结构**：[tensorrt_llm/executor/result.py](https://github.com/NVIDIA/TensorRT-LLM/blob/main/tensorrt_llm/executor/result.py) - KV 缓存、计时与推测式解码（speculative decoding）相关指标
- **引擎统计**：`engine.llm.get_stats_async()` - 系统级聚合统计
- **KV 缓存事件**：`engine.llm.get_kv_cache_events_async()` - 实时缓存操作

### RequestPerfMetrics JSON 结构示例

```json
{
  "timing_metrics": {
    "arrival_time": 1234567890.123,
    "first_scheduled_time": 1234567890.135,
    "first_token_time": 1234567890.150,
    "last_token_time": 1234567890.300,
    "kv_cache_size": 2048576,
    "kv_cache_transfer_start": 1234567890.140,
    "kv_cache_transfer_end": 1234567890.145
  },
  "kv_cache_metrics": {
    "num_total_allocated_blocks": 100,
    "num_new_allocated_blocks": 10,
    "num_reused_blocks": 90,
    "num_missed_blocks": 5
  },
  "speculative_decoding": {
    "acceptance_rate": 0.85,
    "total_accepted_draft_tokens": 42,
    "total_draft_tokens": 50
  }
}
```

**注意：** 上述结构在本文档撰写时有效，但可能随 TensorRT-LLM 版本更新而变化。

## 实现细节

- **Prometheus 集成**：使用 `tensorrt_llm.metrics` 中的 `MetricsCollector` 类（参见 [collector.py](https://github.com/NVIDIA/TensorRT-LLM/blob/main/tensorrt_llm/metrics/collector.py)）
- **Dynamo 集成**：使用 `register_engine_metrics_callback()` 函数，并通过 `metric_prefix_filter=["trtllm_"]` 过滤
- **引擎配置**：启用 `--publish-events-and-metrics` 时，`return_perf_metrics` 设为 `True`
- **初始化**：TensorRT-LLM 引擎初始化完成后，指标才会出现
- **元数据**：`MetricsCollector` 初始化时带有模型元数据（model name、engine type）

## 相关文档

### TensorRT-LLM 指标
- 详见上文 [非 Prometheus 性能指标](#non-prometheus-performance-metrics) 一节，了解性能数据与源代码引用
- [TensorRT-LLM Metrics Collector](https://github.com/NVIDIA/TensorRT-LLM/blob/main/tensorrt_llm/metrics/collector.py) - 源代码参考

### Dynamo 指标
- [Dynamo Metrics 指南](../../observability/metrics.md) - Dynamo runtime 指标的完整文档
- [Prometheus 与 Grafana 搭建](../../observability/prometheus-grafana.md) - 可视化搭建指南
- Dynamo runtime 指标（前缀 `dynamo_*`）会出现在与 TensorRT-LLM 指标同一个 `/metrics` 端点上
  - 实现：`lib/runtime/src/metrics.rs`（Rust runtime metrics）
  - 指标名：`lib/runtime/src/metrics/prometheus_names.rs`（指标名常量）
  - 集成代码：`components/src/dynamo/common/utils/prometheus.py` - Prometheus 工具与回调注册
