---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Observability
---

本指南介绍通过 Dynamo 运行的 SGLang 部署的指标（metrics）、追踪（tracing）与可视化。

## Prometheus 指标

通过 Dynamo 运行 SGLang 时，SGLang 引擎指标会被自动透传，并暴露在 Dynamo 的 `/metrics` 端点上（默认端口 8081）。这样，你可以从同一个 worker 后端端点同时获取 SGLang 引擎指标（前缀 `sglang:`）与 Dynamo 运行时指标（前缀 `dynamo_*`）。

**所有 SGLang 指标的完整、权威列表**请始终参阅 [SGLang 官方 Production Metrics 文档](https://docs.sglang.io/references/production_metrics.html)。

**Dynamo 运行时指标**请参阅 [Dynamo Metrics Guide](../../observability/metrics.md)。

**可视化搭建说明**请参阅 [Prometheus and Grafana Setup Guide](../../observability/prometheus-grafana.md)。

### 环境变量

| 变量 | 说明 | 默认值 | 示例 |
|----------|-------------|---------|---------|
| `DYN_SYSTEM_PORT` | 系统指标/健康检查端口 | `-1`（禁用） | `8081` |

### 快速上手

下面是单机示例。

#### 启动可观测性栈

要使用 Prometheus 与 Grafana 可视化指标，请先启动可观测性栈。说明见 [Observability Getting Started](../../observability/README.md#getting-started-quickly)。

#### 启动 Dynamo 组件

启动一个前端与 SGLang 后端来测试指标：

```bash
# Start frontend (default port 8000, override with --http-port or DYN_HTTP_PORT env var)
$ python -m dynamo.frontend

# Enable system metrics server on port 8081
$ DYN_SYSTEM_PORT=8081 python -m dynamo.sglang --model <model_name> --enable-metrics
```

等待 SGLang worker 启动，然后发送请求并查看指标：

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
curl -s localhost:8081/metrics | grep "^sglang:"
```

### 暴露的指标

SGLang 在 `/metrics` HTTP 端点上以 Prometheus Exposition Format 文本暴露指标。所有 SGLang 引擎指标都使用 `sglang:` 前缀，并带有用于标识来源的标签（如 `model_name`、`engine_type`、`tp_rank`、`pp_rank`）。

**Prometheus Exposition Format 文本示例：**

```
# HELP sglang:prompt_tokens_total Number of prefill tokens processed.
# TYPE sglang:prompt_tokens_total counter
sglang:prompt_tokens_total{model_name="meta-llama/Llama-3.1-8B-Instruct"} 8128902.0

# HELP sglang:generation_tokens_total Number of generation tokens processed.
# TYPE sglang:generation_tokens_total counter
sglang:generation_tokens_total{model_name="meta-llama/Llama-3.1-8B-Instruct"} 7557572.0

# HELP sglang:cache_hit_rate The cache hit rate
# TYPE sglang:cache_hit_rate gauge
sglang:cache_hit_rate{model_name="meta-llama/Llama-3.1-8B-Instruct"} 0.0075
```

**注意：** 上述具体指标仅作示例，可能会随 SGLang 版本变化。请实际查看 `/metrics` 端点，或参考 [官方文档](https://docs.sglang.io/references/production_metrics.html) 以获取当前列表。

### 指标分类

SGLang 提供以下几类指标（均带 `sglang:` 前缀）：

- **吞吐指标** —— token 处理速率
- **资源用量** —— 系统资源消耗
- **延迟指标** —— 请求与 token 时延
- **解耦（Disaggregation）指标** —— 解耦部署专用指标（启用时）

**注意：** 各 SGLang 版本之间具体指标可能变化。请始终参阅 [官方文档](https://docs.sglang.io/references/production_metrics.html) 或检查你版本的 `/metrics` 端点。

### 可用指标

SGLang 官方文档包含完整指标定义：
- HELP 与 TYPE 描述
- Counter、Gauge 与 Histogram 类型
- 指标标签（如 `model_name`、`engine_type`、`tp_rank`、`pp_rank`）
- Prometheus + Grafana 监控搭建指南
- 故障排查与配置示例

完整、权威的 SGLang 指标列表，请参阅 [SGLang 官方 Production Metrics 文档](https://docs.sglang.io/references/production_metrics.html)。

### 实现细节

- SGLang 通过 `prometheus_client.multiprocess.MultiProcessCollector` 进行多进程指标采集
- 在暴露前，指标会按 `sglang:` 前缀过滤
- 集成使用了 Dynamo 的 `register_engine_metrics_callback()` 函数
- 指标在 SGLang 引擎初始化完成后出现

---

## 分布式追踪

Dynamo 在 SGLang 请求管线中传播 [W3C Trace Context](https://www.w3.org/TR/trace-context/) 头部，让你能够在解耦部署中跨前端、router 与各 SGLang worker 关联追踪（trace）。

### 前置条件

SGLang 引擎内部追踪需要 `opentelemetry` 系列包。它们被声明为 SGLang 的 `[tracing]` extra。请把它们装到你的 Dynamo 环境：

```bash
uv pip install opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp opentelemetry-exporter-otlp-proto-grpc
```

没有这些包时，Dynamo 端的 span（前端、handler）仍可工作，但 SGLang 内部引擎 span 不会发出，并会出现警告：`"Tracing is disabled because the packages cannot be imported."`

### Trace 传播原理

```
Frontend (Rust)
  creates span, embeds trace_id + span_id in Context
    |
    v
Dynamo RPC (NATS transport)
  Context serialized with trace_id, span_id
    |
    v
SGLang Handler (Python)
  dynamo.common.utils.otel_tracing.build_trace_headers(context)
  builds W3C traceparent: "00-{trace_id}-{span_id}-01"
    |
    v
sgl.Engine.async_generate(
    ...,
    rid=trace_id,                        # request ID = trace ID
    external_trace_header=traceparent    # W3C header for SGLang internal spans
)
    |
    v
SGLang Engine (internal spans attached to same trace)
```

关键实现文件：
- `components/src/dynamo/common/utils/otel_tracing.py` —— W3C `traceparent` 头部构造
- `components/src/dynamo/sglang/request_handlers/handler_base.py:71-84` —— 从 Dynamo `Context` 对象中提取 trace context
- `components/src/dynamo/sglang/request_handlers/llm/decode_handler.py` —— 把 `external_trace_header` 与 `rid=trace_id` 传给 `engine.async_generate()`

### 环境变量

| 变量 | 说明 | 默认值 | 示例 |
|----------|-------------|---------|---------|
| `DYN_LOGGING_JSONL` | 启用 JSONL 日志（追踪所需） | `false` | `true` |
| `OTEL_EXPORT_ENABLED` | 启用 OTLP trace 导出 | `false` | `true` |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` | 指向 Tempo 的 OTLP gRPC 端点 | `http://localhost:4317` | `http://tempo:4317` |
| `OTEL_SERVICE_NAME` | 在 Grafana Tempo 中显示的服务名 | `dynamo` | `dynamo-worker-decode` |

### SGLang 专属参数

| 参数 | 说明 |
|------|-------------|
| `--enable-trace` | 启用 W3C trace 头向 SGLang 引擎传播 |
| `--otlp-traces-endpoint` | SGLang 内部 trace 导出的 OTLP gRPC 端点（裸 `host:port` 形式，例如 `localhost:4317`） |

二者都需要才能实现端到端贯通 SGLang 引擎的追踪。没有 `--enable-trace`，Dynamo handler 仍会创建 span，但 SGLang 内部引擎 span 不会被关联。

### 启动并启用追踪

解耦启动脚本支持 `--enable-otel`，可在所有组件上启用追踪：

```bash
# Start observability stack first
docker compose -f deploy/docker-compose.yml up -d
docker compose -f deploy/docker-observability.yml up -d

# Launch SGLang disaggregated with tracing
cd examples/backends/sglang/launch
./disagg.sh --enable-otel
```

或者手动以聚合方式部署：

```bash
export DYN_LOGGING_JSONL=true
export OTEL_EXPORT_ENABLED=true
export OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://localhost:4317

# Frontend
OTEL_SERVICE_NAME=dynamo-frontend python -m dynamo.frontend &

# SGLang worker with tracing
OTEL_SERVICE_NAME=dynamo-worker-sglang \
DYN_SYSTEM_PORT=8081 \
python -m dynamo.sglang \
  --model Qwen/Qwen3-0.6B \
  --enable-metrics \
  --enable-trace \
  --otlp-traces-endpoint localhost:4317
```

### 追踪中能看到什么

启用追踪后，每个推理请求都会产生一条贯穿请求生命周期的端到端 trace：

- **前端 `http-request` span** —— 来自 HTTP service 的根 span，包含 method/uri/trace_id
- **KV Router span** —— `kv_router.route_request`、`kv_router.select_worker`、`kv_router.compute_block_hashes`、`kv_router.find_matches`、`kv_router.compute_seq_hashes`、`kv_router.schedule`
- **Worker `handle_payload` span** —— worker 端的 Dynamo RPC handler，带有 component/endpoint/namespace 标签
- **SGLang 引擎 span** —— `Req <id>`、`Scheduler`、`Tokenizer`、`request_process`、`prefill_forward`、`decode_loop`、`Bootstrap Room`（解耦时）
- **语义约定** —— `gen_ai.usage.prompt_tokens`、`gen_ai.usage.completion_tokens`、`gen_ai.latency.time_to_first_token` 等

KV 路由请求的示例 trace 树：

```
dynamo-frontend: http-request (root)
  dynamo-frontend: kv_router.route_request
    dynamo-frontend: kv_router.select_worker
      kv_router.compute_block_hashes
      kv_router.find_matches
      kv_router.compute_seq_hashes
      kv_router.schedule
    dynamo-worker-1: handle_payload
      sglang: Bootstrap Room 0x0
        sglang: Req <trace-id-prefix>
          sglang: Scheduler [TP 0]
            request_process
            prefill_forward
            decode_loop (repeated per token)
          sglang: Tokenizer
            tokenize
            dispatch
```

![End-to-end trace in Grafana Tempo showing frontend, KV router, worker, and SGLang engine spans](../../assets/img/sglang-trace.png)

### 查看 trace

1. 打开 `http://localhost:3000` 上的 Grafana（用户名：`dynamo`，密码：`dynamo`）
2. 进入 **Explore**（指南针图标）
3. 选择 **Tempo** 作为数据源
4. 使用 **Search** 标签：
   - 按 **Service Name** 过滤（如 `dynamo-frontend`、`dynamo-worker-1`、`sglang`）
   - 按 **Span Name** 过滤（如 `http-request`、`handle_payload`、`Req *`、`decode_loop`）
   - 按 **Tags** 过滤（如 `rid=<trace-id>`、`gen_ai.response.model=Qwen/Qwen3-0.6B`）
5. 点击 trace 即可查看跨"前端 → router → worker → 引擎"的火焰图

发请求时带上 `x-request-id` 便于查找：

```bash
curl -H 'Content-Type: application/json' \
  -H 'x-request-id: my-trace-001' \
  -d '{"model": "Qwen/Qwen3-0.6B", "max_completion_tokens": 50,
       "messages": [{"role": "user", "content": "Explain why Roger Federer is considered one of the greatest tennis players of all time"}]}' \
  http://localhost:8000/v1/chat/completions
```

关于 Tempo/Grafana 追踪基础设施的更多细节，请参阅 [Dynamo Tracing Guide](../../observability/tracing.md)。

---

## SGLang Grafana Dashboard

Dynamo 在 `deploy/observability/grafana_dashboards/sglang.json` 中预置了一个 SGLang 的 Grafana dashboard。可观测性栈启动时它会被自动加载。

### Dashboard 面板

dashboard 分为五个区域：

| 区域 | 面板 | 关注点 |
|---------|--------|---------------|
| **请求时延** | E2E 请求时延、TTFT、Inter-Token 时延 | 长尾时延回退、prefill 压力下的 TTFT 尖峰 |
| **吞吐与队列** | Token 生成吞吐（tok/s）、运行中与排队请求数、请求速率 | 吞吐饱和、队列深度增长 |
| **缓存与 PIN** | 缓存命中率、活跃 PIN 数、回退（Retractions） | KV 缓存复用效率、解耦路由带来的 PIN 压力 |
| **内存压力** | GPU KV 缓存使用率 %、Host（CPU）KV 缓存使用率 %、驱逐与 load-back 速率 | OOM 风险、HiCache 卸载活动 |
| **HiCache 时延** | 驱逐 P99 时延、Load-back P99 时延 | KV 卸载路径上的 PCIe/NVLink 瓶颈 |

### 访问 dashboard

1. 打开 `http://localhost:3000` 上的 Grafana
2. 用 `dynamo` / `dynamo` 登录
3. 点击左侧栏的 **Dashboards**
4. 选择 **SGLang Engine**

其他可用 dashboard：
- **Dynamo Dashboard**（`dynamo.json`）—— 前端与组件指标
- **DCGM Metrics**（`dcgm-metrics.json`）—— GPU 利用率、显存、功耗
- **KVBM**（`kvbm.json`）—— KV block manager 指标
- **Disagg Dashboard**（`disagg-dashboard.json`）—— 解耦服务指标

---

## 在远程 VM 上访问

在远程 VM（云实例、裸机等）上开发时，可观测性端口仅绑定在 VM 内部的 `localhost`。可以用以下两种方式访问。

### 方式 1：SSH 端口转发（推荐）

通过 SSH 转发相关端口。无需改防火墙，流量是加密的。

```bash
# Forward Grafana (3000), Prometheus (9090), and Tempo (3200)
ssh -L 3000:localhost:3000 \
    -L 9090:localhost:9090 \
    -L 3200:localhost:3200 \
    user@your-vm-ip
```

之后在本地浏览器打开 `http://localhost:3000`。

后台长隧道：

```bash
ssh -fN \
    -L 3000:localhost:3000 \
    -L 9090:localhost:9090 \
    -L 3200:localhost:3200 \
    user@your-vm-ip
```

### 方式 2：防火墙规则

直接放开端口。仅在受信网络下使用。

```bash
# Ubuntu/Debian
sudo ufw allow 3000/tcp   # Grafana
sudo ufw allow 9090/tcp   # Prometheus

# Or for cloud VMs, add inbound rules in your security group for ports 3000, 9090
```

之后直接访问 `http://<vm-ip>:3000`。

### 无界面 / Agent 访问

对于 CI 流水线、AI 编码 Agent 或其他无浏览器的工作流，可以通过 API 直接查询 Grafana 与 Prometheus：

```bash
# Query Prometheus for SGLang token throughput
curl -s 'http://localhost:9090/api/v1/query?query=rate(sglang:generation_tokens_total[1m])' | python3 -m json.tool

# Query Prometheus for GPU KV cache usage
curl -s 'http://localhost:9090/api/v1/query?query=dynamo_component_gpu_cache_usage_percent' | python3 -m json.tool

# List available Grafana dashboards
curl -s -u dynamo:dynamo http://localhost:3000/api/search | python3 -m json.tool

# Get the SGLang dashboard by title
curl -s -u dynamo:dynamo 'http://localhost:3000/api/search?query=SGLang' | python3 -m json.tool

# Fetch a specific dashboard by UID
curl -s -u dynamo:dynamo http://localhost:3000/api/dashboards/uid/<dashboard-uid> | python3 -m json.tool

# Snapshot current metrics via Prometheus range query (last hour)
START=$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)
END=$(date -u +%Y-%m-%dT%H:%M:%SZ)
curl -s "http://localhost:9090/api/v1/query_range?query=sglang:cache_hit_rate&start=${START}&end=${END}&step=15s"
```

这对自动化基准测试管线非常有用——你可以在采集性能结果的同时通过程序拿到指标。

---

## 相关文档

### SGLang 指标
- [SGLang 官方 Production Metrics](https://docs.sglang.io/references/production_metrics.html)
- [SGLang GitHub - Metrics Collector](https://github.com/sgl-project/sglang/blob/v0.5.9/python/sglang/srt/metrics/collector.py)

### Dynamo 可观测性
- [Dynamo Metrics Guide](../../observability/metrics.md) —— Dynamo 运行时指标的完整文档
- [Dynamo Tracing Guide](../../observability/tracing.md) —— 基于 OpenTelemetry 与 Tempo 的分布式追踪
- [Prometheus and Grafana Setup](../../observability/prometheus-grafana.md) —— 可视化搭建说明
- Dynamo 运行时指标（前缀 `dynamo_*`）与 SGLang 指标一同暴露在同一个 `/metrics` 端点
  - 实现：`lib/runtime/src/metrics.rs`（Rust 运行时指标）
  - 指标名：`lib/runtime/src/metrics/prometheus_names.rs`（指标名常量）
  - 集成代码：`components/src/dynamo/common/utils/prometheus.py` —— Prometheus 工具与回调注册
