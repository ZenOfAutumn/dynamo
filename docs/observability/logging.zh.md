---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Logging
---

## 概览

Dynamo 同时提供文本与 JSONL 两种结构化日志。当启用
JSONL 时，日志支持 `trace_id` 与 `span_id` 字段，用于
分布式追踪。span 的创建与退出事件可通过 `DYN_LOGGING_SPAN_EVENTS`
环境变量按需启用。

## 环境变量

| 变量 | 说明 | 默认值 | 示例 |
|----------|-------------|---------|---------|
| `DYN_LOGGING_JSONL` | 启用 JSONL 日志格式 | `false` | `true` |
| `DYN_LOGGING_SPAN_EVENTS` | 启用 span 进入/关闭事件日志（`SPAN_FIRST_ENTRY`、`SPAN_CLOSED` 消息） | `false` | `true` |
| `DYN_LOG` | 按目标设置日志级别 `<default_level>,<module_path>=<level>,<module_path>=<level>` | `info` | `DYN_LOG=info,dynamo_runtime::system_status_server:trace` |
| `DYN_LOG_USE_LOCAL_TZ` | 时间戳使用本地时区（默认 UTC） | `false` | `true` |
| `DYN_LOGGING_CONFIG_PATH` | 自定义 TOML 日志配置的路径 | none | `/path/to/config.toml` |
| `VLLM_LOGGING_LEVEL` | vLLM 后端日志级别（与 `DYN_LOG` 独立） | `INFO` | `DEBUG` |
| `TLLM_LOG_LEVEL` | TensorRT-LLM 后端日志级别（与 `DYN_LOG` 独立） | `INFO` | `DEBUG` |
| `DYN_SKIP_SGLANG_LOG_FORMATTING` | 禁用 Dynamo 对 SGLang 的日志配置 | `false` | `true` |
| `OTEL_SERVICE_NAME` | trace 与 span 信息的服务名 | `dynamo` | `dynamo-frontend` |
| `OTEL_EXPORT_ENABLED` | 同时启用 trace 与 logs 的 OTLP 导出 | `false` | `true` |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` | trace 的 OTLP gRPC 端点 | `http://localhost:4317` | `http://tempo:4317` |
| `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT` | logs 的 OTLP gRPC 端点（未设置时回落到 `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`） | 与 traces endpoint 相同 | `http://localhost:4317` |

## OTLP 日志导出

当 `OTEL_EXPORT_ENABLED=true` 时，Dynamo 会通过 OTLP 同时导出
**trace 与 logs**。日志被发送到 OpenTelemetry Collector，由其
路由到 Grafana Loki 进行聚合与查询。

默认情况下，logs 与 traces 导出到相同端点（`OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`）。
如需将 logs 发送到不同端点，请设置 `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT`：

```bash
export OTEL_EXPORT_ENABLED=true
export OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://localhost:4317
# Optional: send logs to a different endpoint
# export OTEL_EXPORTER_OTLP_LOGS_ENDPOINT=http://localhost:4317
```

本地可观测性栈（参见 [Getting Started](README.md#getting-started-quickly)）包含
一个 OpenTelemetry Collector，在 `localhost:4317` 接收 OTLP，并将
trace 路由到 Tempo、logs 路由到 Loki。在 Grafana 中，Loki 数据源
预配置了一个派生字段，用于把 `trace_id` 标签链接到 Tempo，因此
你可以直接从一行日志跳到对应的 trace。

## 快速开始

### 启动可观测性栈

要使用 Grafana Loki 收集与可视化日志，或在日志中查看 trace 上下文
并配合 Grafana Tempo，请启动可观测性栈。详见 [Observability Getting Started](README.md#getting-started-quickly)。
该栈包含 Loki、OpenTelemetry Collector 与 Tempo——已经预先连通。

### 启用结构化日志

启用结构化 JSONL 日志：

```bash
export DYN_LOGGING_JSONL=true
export DYN_LOG=debug

# Start your Dynamo components (default port 8000, override with --http-port or DYN_HTTP_PORT env var)
python -m dynamo.frontend &
python -m dynamo.vllm --model Qwen/Qwen3-0.6B --enforce-eager &
```

日志会以带 trace 上下文的 JSONL 格式写入 stderr。

## 可用的日志级别

| **日志级别（从最少到最详细）** | **说明**                                                                 |
|-------------------------------------------|---------------------------------------------------------------------------------|
| **ERROR**                                 | 严重错误（例如不可恢复的失败、资源耗尽）              |
| **WARN**                                  | 异常或降级情形（例如重试、可恢复错误）           |
| **INFO**                                  | 运行信息（例如启动/关闭、重大事件）                 |
| **DEBUG**                                 | 一般调试信息（例如变量值、流程控制）            |
| **TRACE**                                 | 极底层、非常详细的信息（例如内部算法步骤）           |

## 可读格式示例

环境变量设置：

```
export DYN_LOG="info,dynamo_runtime::system_status_server:trace"
export DYN_LOGGING_JSONL="false"
```

输出日志格式：

```
2025-09-02T15:50:01.770028Z  INFO main.init: VllmWorker for Qwen/Qwen3-0.6B has been initialized
2025-09-02T15:50:01.770195Z  INFO main.init: Reading Events from tcp://127.0.0.1:21555
2025-09-02T15:50:01.770265Z  INFO main.init: Getting engine runtime configuration metadata from vLLM engine...
2025-09-02T15:50:01.770316Z  INFO main.get_engine_cache_info: Cache config values: {'num_gpu_blocks': 24064}
2025-09-02T15:50:01.770358Z  INFO main.get_engine_cache_info: Scheduler config values: {'max_num_seqs': 256, 'max_num_batched_tokens': 2048}
```

## JSONL 格式示例

环境变量设置：

```
export DYN_LOG="info,dynamo_runtime::system_status_server:trace"
export DYN_LOGGING_JSONL="true"
```

输出日志格式：

```
{"time":"2025-09-02T15:53:31.943377Z","level":"INFO","target":"log","message":"VllmWorker for Qwen/Qwen3-0.6B has been initialized","log.file":"/opt/dynamo/venv/lib/python3.12/site-packages/dynamo/vllm/main.py","log.line":191,"log.target":"main.init"}
{"time":"2025-09-02T15:53:31.943550Z","level":"INFO","target":"log","message":"Reading Events from tcp://127.0.0.1:26771","log.file":"/opt/dynamo/venv/lib/python3.12/site-packages/dynamo/vllm/main.py","log.line":212,"log.target":"main.init"}
{"time":"2025-09-02T15:53:31.943636Z","level":"INFO","target":"log","message":"Getting engine runtime configuration metadata from vLLM engine...","log.file":"/opt/dynamo/venv/lib/python3.12/site-packages/dynamo/vllm/main.py","log.line":220,"log.target":"main.init"}
{"time":"2025-09-02T15:53:31.943701Z","level":"INFO","target":"log","message":"Cache config values: {'num_gpu_blocks': 24064}","log.file":"/opt/dynamo/venv/lib/python3.12/site-packages/dynamo/vllm/main.py","log.line":267,"log.target":"main.get_engine_cache_info"}
{"time":"2025-09-02T15:53:31.943747Z","level":"INFO","target":"log","message":"Scheduler config values: {'max_num_seqs': 256, 'max_num_batched_tokens': 2048}","log.file":"/opt/dynamo/venv/lib/python3.12/site-packages/dynamo/vllm/main.py","log.line":268,"log.target":"main.get_engine_cache_info"}
```

## 在日志中输出 Trace 与 Span ID

当启用 `DYN_LOGGING_JSONL` 时，所有日志都会包含 `trace_id` 与 `span_id`
字段，并自动为每个请求创建 span。这适合短时调试场景：你希望
在日志中查看 trace 上下文，但不想搭建完整的追踪后端，并想把日志
与 trace 关联起来。

trace 与 span 信息使用 OpenTelemetry 的格式与库，因此 ID 与
基于 OpenTelemetry 的追踪后端（如 Tempo 或 Jaeger）兼容；之后
若启用 trace 导出可直接对接。

**注意：** 本节内容与 [Distributed Tracing with Tempo](tracing.md) 有重叠。
关于 Grafana Tempo 中的 trace 可视化与持久化追踪分析，
请参见 [Distributed Tracing with Tempo](tracing.md)。

### 日志配置

要在日志中看到 trace 信息：

```bash
export DYN_LOGGING_JSONL=true
export DYN_LOG=debug  # Set to debug to see detailed trace logs

# Start your Dynamo components (e.g., frontend and worker) (default port 8000, override with --http-port or DYN_HTTP_PORT env var)
python -m dynamo.frontend &
python -m dynamo.vllm --model Qwen/Qwen3-0.6B --enforce-eager &
```

这会启用包含 `trace_id` 与 `span_id` 字段的 JSONL 日志。trace 出现
在日志里，但不会被导出到任何后端。

### 示例请求

发送一个请求来生成带 trace 上下文的日志：

```bash
curl -H 'Content-Type: application/json' \
-H 'x-request-id: test-trace-001' \
-d '{
  "model": "Qwen/Qwen3-0.6B",
  "max_completion_tokens": 100,
  "messages": [
    {"role": "user", "content": "What is the capital of France?"}
  ]
}' \
http://localhost:8000/v1/chat/completions
```

查看日志（stderr）的 JSONL 输出，你会看到包含 `trace_id`、`span_id`
与 `x_request_id` 字段的内容。

## 日志中的 Trace 与 Span 信息

本节展示 trace 与 span 信息在 JSONL 日志中的表现。即便没有 trace
可视化后端，这些日志也能帮助理解请求流。

### 在 Grafana 中的解耦 Trace 示例

在 Grafana 中查看对应 trace 时，应能看到类似下图：

![Disaggregated Trace Example](../assets/img/grafana-disagg-trace.png)
### Trace 概览

Dynamo 在解耦服务（disaggregated serving）部署中创建跨多个服务的
分布式 trace。下面的小节描述你在 Grafana 中查看 chat completion 请求
时会看到的关键 span。

#### 解耦模式下的可用 Span

在解耦模式下运行 Dynamo 时，一次典型请求会创建以下 span：

##### 1. `http-request`（前端 - 根 Span）

整个请求生命期的根 span，由 **dynamo-frontend** 服务创建。

**关键属性：**
- **服务**：`dynamo-frontend`
- **操作**：处理客户端 HTTP 请求至完成
- **时长**：端到端的总请求时间（含 prefill + decode）
- **方法**：HTTP 方法（通常为 `POST`）
- **URI**：请求端点（例如 `/v1/chat/completions`）
- **状态**：请求完成状态
- **子 span**：通常 2-3 个子 span（路由 span + worker span）

该 span 表示从前端收到 HTTP 请求到向客户端发送最终响应的完整请求流程。

##### 2. `prefill_routing`（前端 - 路由 Span）

`http-request` 的子 span，由 **dynamo-frontend** 服务在路由阶段创建。

**关键属性：**
- **服务**：`dynamo-frontend`
- **操作**：将 prefill 请求路由到合适的 prefill worker
- **时长**：选择 worker 与 prefill 跨度所耗时间。
- **父 span**：`http-request` span

该 span 记录路由逻辑、决策过程，以及向 prefill worker 发出的请求。

##### 3. `handle_payload`（Prefill Worker Span）

`http-request` 的子 span，在 **dynamo-worker-vllm-prefill** 服务中创建。

**关键属性：**
- **服务**：`dynamo-worker-vllm-prefill`（SGLang 为 `dynamo-worker-sglang-prefill`）
- **操作**：处理生成的 prefill 阶段
- **时长**：prefill 计算耗时（通常为毫秒到秒级）
- **组件**：`prefill`
- **端点**：`generate`
- **父 span**：`http-request` span

该 span 表示在 prefill 专用 worker 上的实际 prefill 计算，包括 prompt
处理与初始 KV cache 的生成。

##### 4. `handle_payload`（Decode Worker Span）

`http-request` 的子 span，在 **dynamo-worker-vllm-decode** 服务中创建。

**关键属性：**
- **服务**：`dynamo-worker-vllm-decode`（SGLang 为 `dynamo-worker-sglang-decode`）
- **操作**：处理生成的 decode 阶段
- **时长**：生成所有输出 token 的时间（通常为秒级）
- **组件**：`decode` 或 `backend`
- **端点**：`generate`
- **父 span**：`http-request` span

该 span 表示在 decode 专用 worker 上的迭代式 token 生成阶段，
该 worker 会消费来自 prefill 的 KV cache 并产生输出 token。


#### 理解 Span 度量

每个 span 提供若干有用的度量：

| 度量 | 说明 |
|--------|-------------|
| **时长** | 从 span 开始到结束的总时间 |
| **忙时（Busy Time）** | 实际处理（不含等待）时间 |
| **闲时（Idle Time）** | 等待时间（例如等待网络、其他服务） |
| **开始时间** | span 开始的时刻 |
| **子数量** | 直接子 span 的数量 |

关系 **时长 = 忙时 + 闲时** 有助于定位时间花在哪里以及潜在瓶颈。

## 日志中的自定义 Request ID

你可以通过 `x-request-id` 请求头提供自定义请求 ID。该 ID 会附加到
该请求的所有 span 与日志上，便于把 trace 与应用层的请求跟踪关联。

### 携带自定义 Request ID 的示例请求

```sh
curl -X POST http://localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'x-request-id: 8372eac7-5f43-4d76-beca-0a94cfb311d0' \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [
      {
        "role": "user",
        "content": "Explain why Roger Federer is considered one of the greatest tennis players of all time"
      }
    ],
    "stream": false,
    "max_tokens": 1000
  }'
```

该请求的所有 span 与日志都会包含值为 `8372eac7-5f43-4d76-beca-0a94cfb311d0`
的 `x_request_id` 属性。

### 包含自定义 Request ID 的前端日志

注意 `x_request_id` 字段如何出现在所有日志条目中，与
`trace_id`（`80196f3e3a6fdf06d23bb9ada3788518`）以及 `span_id` 并列：

```
{"time":"2025-10-31T21:06:45.397194Z","level":"DEBUG","file":"/opt/dynamo/lib/runtime/src/pipeline/network/tcp/server.rs","line":230,"target":"dynamo_runtime::pipeline::network::tcp::server","message":"Registering new TcpStream on 10.0.4.65:41959","method":"POST","span_id":"f7e487a9d2a6bf38","span_name":"http-request","trace_id":"80196f3e3a6fdf06d23bb9ada3788518","uri":"/v1/chat/completions","version":"HTTP/1.1","x_request_id":"8372eac7-5f43-4d76-beca-0a94cfb311d0"}
{"time":"2025-10-31T21:06:45.418584Z","level":"DEBUG","file":"/opt/dynamo/lib/llm/src/kv_router/prefill_router.rs","line":232,"target":"dynamo_llm::kv_router::prefill_router","message":"Prefill succeeded, using disaggregated params for decode","method":"POST","span_id":"f7e487a9d2a6bf38","span_name":"http-request","trace_id":"80196f3e3a6fdf06d23bb9ada3788518","uri":"/v1/chat/completions","version":"HTTP/1.1","x_request_id":"8372eac7-5f43-4d76-beca-0a94cfb311d0"}
{"time":"2025-10-31T21:06:45.418854Z","level":"DEBUG","file":"/opt/dynamo/lib/runtime/src/pipeline/network/tcp/server.rs","line":230,"target":"dynamo_runtime::pipeline::network::tcp::server","message":"Registering new TcpStream on 10.0.4.65:41959","method":"POST","span_id":"f7e487a9d2a6bf38","span_name":"http-request","trace_id":"80196f3e3a6fdf06d23bb9ada3788518","uri":"/v1/chat/completions","version":"HTTP/1.1","x_request_id":"8372eac7-5f43-4d76-beca-0a94cfb311d0"}
```



## 后端引擎日志级别

Dynamo 的 `DYN_LOG` 环境变量控制 Dynamo 自身的日志。每种推理后端
都有自己的日志级别开关，**与 `DYN_LOG` 独立**。

### vLLM

vLLM 日志级别由 `VLLM_LOGGING_LEVEL` 环境变量控制。
默认 `INFO`，与 `DYN_LOG` 完全独立。

```bash
# Set vLLM to debug while keeping Dynamo at info
export DYN_LOG=info
export VLLM_LOGGING_LEVEL=DEBUG
```

合法取值：`DEBUG`、`INFO`、`WARNING`、`ERROR`、`CRITICAL`。

### TensorRT-LLM

TensorRT-LLM 日志级别由 `TLLM_LOG_LEVEL` 环境变量控制。
默认 `INFO`，与 `DYN_LOG` 完全独立。

```bash
# Set TRT-LLM to info while keeping Dynamo at warn
export DYN_LOG=warn
export TLLM_LOG_LEVEL=INFO
```

合法取值：`TRACE`、`DEBUG`、`INFO`、`WARNING`、`ERROR`、`INTERNAL_ERROR`。

**注意：** `TLLM_LOG_LEVEL` 在 TensorRT-LLM 导入时只读取一次。
必须在进程启动前设置好。

### SGLang

SGLang 的日志当前通过 Dynamo 进行配置，默认遵循
`DYN_LOG` 级别。如需禁用 Dynamo 对 SGLang 的日志配置并独立管理，
请设置：

```bash
export DYN_SKIP_SGLANG_LOG_FORMATTING=true
```

或者在 SGLang worker 命令中传入 `--log-level` 参数以直接设置 SGLang
引擎的日志级别（例如 `--log-level DEBUG`）。这与 `DYN_LOG` 独立。

## 相关文档

- [Distributed Tracing with Tempo](tracing.md)
- [Log Aggregation in Kubernetes](../kubernetes/observability/logging.md)
- [Observability Getting Started](README.md)
- [Distributed Runtime Architecture](../design-docs/distributed-runtime.md)
- [Dynamo Architecture Overview](../design-docs/architecture.md)
- [Backend Guide](../development/backend-guide.md)
