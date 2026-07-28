---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 追踪（Tracing）
---

## 概述

Dynamo 支持基于 OpenTelemetry 的分布式追踪，用于可视化 Frontend 与 Worker 组件间的请求流。Trace 通过 OTLP（OpenTelemetry Protocol）导出到 Tempo，并在 Grafana 中可视化。

**要求：** 设置 `DYN_LOGGING_JSONL=true` 和 `OTEL_EXPORT_ENABLED=true` 以将 trace 导出到 Tempo。

**注意：** 启用 OTLP 导出时，Dynamo 同时导出 **trace 与日志**。Trace 发送到 Tempo，日志发送到 Loki（通过 OpenTelemetry Collector）。如需将日志发送到独立端点，设置 `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT`；否则默认使用 trace 端点。详见[Logging](logging.md#otlp-log-export)。

本指南介绍使用 Docker Compose 的单 GPU Demo 配置。Kubernetes 部署见 [Kubernetes 部署](#kubernetes-deployment)。

**注意：** 本节与 [Logging of OpenTelemetry Tracing](logging.md) 有重叠，因为 OpenTelemetry 同时涉及日志与追踪。这里描述的追踪方式用于持久化的 trace 可视化与分析。如果只是短时调试需要直接在日志中查看 trace 上下文，请参阅[Logging](logging.md) 指南。

## 环境变量

| 变量 | 描述 | 默认值 | 示例 |
|----------|-------------|---------|---------|
| `DYN_LOGGING_JSONL` | 启用 JSONL 日志格式（追踪所必需） | `false` | `true` |
| `OTEL_EXPORT_ENABLED` | 启用 OTLP trace 导出 | `false` | `true` |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` | trace 的 OTLP gRPC 端点 | `http://localhost:4317` | `http://tempo:4317` |
| `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT` | 日志的 OTLP gRPC 端点（默认与 trace 一致） | 与 trace 相同 | `http://localhost:4317` |
| `OTEL_SERVICE_NAME` | 用于识别组件的服务名 | `dynamo` | `dynamo-frontend` |

## 快速开始

### 1. 启动可观测性栈

启动可观测性栈（Prometheus、Grafana、Tempo、各 exporter）。说明见[可观测性快速开始](README.md#getting-started-quickly)。

### 2. 启动 Dynamo 组件（单 GPU）

对于简单的单 GPU 部署，运行聚合式追踪启动脚本。该脚本启用追踪，设置每组件的服务名，并启动一个前端及一个 vLLM worker：

```bash
cd examples/backends/vllm/launch
./agg_tracing.sh
```

如需覆盖 Tempo 端点（默认 `http://localhost:4317`）：

```bash
export OTEL_EXPORTER_OTLP_TRACES_ENDPOINT=http://tempo:4317
./agg_tracing.sh
```

它在一块 GPU 上运行单个聚合式 worker，提供更简单的追踪测试设置。

### 备选：分离式部署（2 GPU）

对于带追踪的分离式部署，运行分离式追踪启动脚本。该脚本设置追踪，并启动一个前端、GPU 0 上的 decode worker 和 GPU 1 上的 prefill worker：

```bash
cd examples/backends/vllm/launch
./disagg_tracing.sh
```

它将 prefill 与 decode 分离到不同 GPU 以获得更好的资源利用。

### 3. 生成 trace

向前端发送请求以生成 trace（聚合与分离部署都适用）。启动脚本会在启动时打印一条带正确模型名的示例 `curl` 命令。

**提示：** 添加 `x-request-id` 头可以方便地在 Grafana 中搜索特定 trace：

```bash
curl -H 'Content-Type: application/json' \
-H 'x-request-id: test-trace-001' \
-d '{
  "model": "<MODEL>",
  "max_completion_tokens": 100,
  "messages": [
    {"role": "user", "content": "What is the capital of France?"}
  ]
}' \
http://localhost:8000/v1/chat/completions
```

### 4. 在 Grafana Tempo 中查看 trace

1. 打开 Grafana：`http://localhost:3000`
2. 使用用户名 `dynamo`、密码 `dynamo` 登录
3. 进入 **Explore**（左侧栏的指南针图标）
4. 选择 **Tempo** 作为数据源（默认应已选中）
5. 在查询类型中选择 **"Search"**（不是 TraceQL，也不是 Service Graph）
6. 使用 **Search** 标签页查找 trace：
   - 按 **服务名** 搜索（例如 `dynamo-frontend`）
   - 按 **Span 名** 搜索（例如 `http-request`、`handle_payload`）
   - 按 **Tags** 搜索（例如 `x_request_id=test-trace-001`）
7. 点击某个 trace 查看详细的火焰图

#### 示例 trace 视图

下图展示了 trace 在 Grafana Tempo 中的样子：

![Trace 示例](../assets/img/trace.png)

### 5. 停止服务

完成后，停止可观测性栈。Docker Compose 命令见[可观测性快速开始](README.md#getting-started-quickly)。

---

## Kubernetes 部署

对于 Kubernetes 部署，请确保已部署可访问的 Tempo 实例（例如 `http://tempo.observability.svc.cluster.local:4317`）。

### 修改 DynamoGraphDeployment 以启用追踪

提供了支持追踪的示例部署变体：

- **聚合式：** `examples/backends/vllm/deploy/agg_tracing.yaml`
- **分离式：** `examples/backends/vllm/deploy/disagg_tracing.yaml`

它们在基础 `agg.yaml` / `disagg.yaml` 部署上添加了[环境变量](#environment-variables)。要覆盖 Tempo 端点，编辑 YAML 中的 `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`。

应用支持追踪的部署：

```bash
kubectl apply -f examples/backends/vllm/deploy/disagg_tracing.yaml
```

trace 会被导出到 Tempo，可在 Grafana 中查看。

