---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Frontend 配置参考
subtitle: Frontend 全部 CLI 参数、环境变量与 HTTP endpoint 的完整参考
---

本页文档描述 Dynamo Frontend（`python -m dynamo.frontend`）的所有配置项。

每个 CLI 参数都对应一个环境变量。CLI 参数优先级高于环境变量。

## HTTP 与网络

| CLI 参数 | 环境变量 | 默认值 | 说明 |
|-------------|---------|---------|-------------|
| `--http-host` | `DYN_HTTP_HOST` | `0.0.0.0` | HTTP 监听地址 |
| `--http-port` | `DYN_HTTP_PORT` | `8000` | HTTP 监听端口 |
| `--tls-cert-path` | `DYN_TLS_CERT_PATH` | — | TLS 证书路径（PEM）。必须与 `--tls-key-path` 同时配置 |
| `--tls-key-path` | `DYN_TLS_KEY_PATH` | — | TLS 私钥路径（PEM）。必须与 `--tls-cert-path` 同时配置 |

Rust HTTP server 还会读取以下环境变量（未暴露为 CLI 参数）：

| 环境变量 | 默认值 | 说明 |
|---------|---------|-------------|
| `DYN_HTTP_BODY_LIMIT_MB` | `192` | 请求体最大大小（MB） |
| `DYN_HTTP_GRACEFUL_SHUTDOWN_TIMEOUT_SECS` | `5` | 优雅关闭超时（秒） |

## Router

| CLI 参数 | 环境变量 | 默认值 | 说明 |
|-------------|---------|---------|-------------|
| `--router-mode` | `DYN_ROUTER_MODE` | `round-robin` | 路由策略：`round-robin`、`random`、`kv`、`direct` |
| `--router-kv-overlap-score-weight` | `DYN_ROUTER_KV_OVERLAP_SCORE_WEIGHT` | `1.0` | worker 打分中 KV cache 重叠度的权重，越大越倾向 cache 复用 |
| `--router-temperature` | `DYN_ROUTER_TEMPERATURE` | `0.0` | worker 采样的 softmax 温度。0 = 确定性 |
| `--router-kv-events` / `--no-router-kv-events` | `DYN_ROUTER_USE_KV_EVENTS` | `true` | 是否启用 worker 上报的 KV cache 状态事件。关闭后改用基于预测的路由 |
| `--router-ttl-secs` | `DYN_ROUTER_TTL_SECS` | `120.0` | 关闭 KV 事件时 block 的 TTL |
| `--router-replica-sync` / `--no-router-replica-sync` | `DYN_ROUTER_REPLICA_SYNC` | `false` | 在多个 router 实例间同步状态 |
| `--router-snapshot-threshold` | `DYN_ROUTER_SNAPSHOT_THRESHOLD` | `1000000` | 触发 snapshot 前的消息数 |
| `--router-reset-states` / `--no-router-reset-states` | `DYN_ROUTER_RESET_STATES` | `false` | 启动时重置 router 状态。**警告：** 会影响已有副本 |
| `--router-track-active-blocks` / `--no-router-track-active-blocks` | `DYN_ROUTER_TRACK_ACTIVE_BLOCKS` | `true` | 追踪进行中请求所占用的 block，用于负载均衡 |
| `--router-assume-kv-reuse` / `--no-router-assume-kv-reuse` | `DYN_ROUTER_ASSUME_KV_REUSE` | `true` | 追踪 active block 时假定存在 KV cache 复用 |
| `--router-track-output-blocks` / `--no-router-track-output-blocks` | `DYN_ROUTER_TRACK_OUTPUT_BLOCKS` | `false` | 在生成期间追踪 output block，并按进度做分数衰减 |
| `--router-track-prefill-tokens` / `--no-router-track-prefill-tokens` | `DYN_ROUTER_TRACK_PREFILL_TOKENS` | `true` | 在 worker 负载核算中追踪 prompt 侧 prefill 负载 |
| `--router-prefill-load-model` | `DYN_ROUTER_PREFILL_LOAD_MODEL` | `none` | Prompt 侧负载模型：`none` 静态负载，`aic` 使用 AIC 预测对最早的 prefill 做衰减 |
| `--router-event-threads` | `DYN_ROUTER_EVENT_THREADS` | `4` | KV indexer 工作线程数。>1 启用并发 radix tree（即便配合 `--no-router-kv-events`） |
| `--router-queue-threshold` | `DYN_ROUTER_QUEUE_THRESHOLD` | `4.0` | 排队阈值，相对 prefill 容量的比例。启用优先级调度 |
| `--router-queue-policy` | `DYN_ROUTER_QUEUE_POLICY` | `fcfs` | 排队策略：`fcfs`（尾部 TTFT）、`wspt`（平均 TTFT）、`lcfs`（仅做反序比较） |
| `--decode-fallback` / `--no-decode-fallback` | `DYN_DECODE_FALLBACK` | `false` | prefill worker 不可用时回退为 aggregated 模式 |

## AIC Prefill 负载模型

仅在同时启用 `--router-mode kv` 与 `--router-prefill-load-model aic` 时使用。

| CLI 参数 | 环境变量 | 默认值 | 说明 |
|-------------|---------|---------|-------------|
| `--aic-backend` | `DYN_AIC_BACKEND` | — | AIC 中建模的 backend 系列，例如 `vllm` 或 `sglang` |
| `--aic-system` | `DYN_AIC_SYSTEM` | — | AIC 硬件 / 系统标识，例如 `h200_sxm` |
| `--aic-model-path` | `DYN_AIC_MODEL_PATH` | — | 用于 AIC 性能查询的模型路径或模型标识 |
| `--aic-backend-version` | `DYN_AIC_BACKEND_VERSION` | 由 backend 决定 | 锁定的 AIC 数据库版本。未指定时使用 backend 默认值 |
| `--aic-tp-size` | `DYN_AIC_TP_SIZE` | `1` | AIC 中建模的 tensor-parallel 大小 |

启用后，frontend 内嵌的 KV router 会基于所选 worker 由 overlap 推导出的 cached prefix，为每个被准入的请求预测一份预期 prefill 时长。随后 router 在 prompt 侧负载核算中只对每个 worker 上最早的 active prefill 做衰减。

## 故障容错

| CLI 参数 | 环境变量 | 默认值 | 说明 |
|-------------|---------|---------|-------------|
| `--migration-limit` | `DYN_MIGRATION_LIMIT` | `0` | 单个 worker 断连后允许的最大请求迁移次数。0 = 禁用 |
| `--active-decode-blocks-threshold` | `DYN_ACTIVE_DECODE_BLOCKS_THRESHOLD` | `1.0` | 用于判定繁忙的 KV cache 占用比（0.0–1.0）。传 `None` 禁用 |
| `--active-prefill-tokens-threshold` | `DYN_ACTIVE_PREFILL_TOKENS_THRESHOLD` | `10000000` | prefill 繁忙判定的绝对 token 数阈值。传 `None` 禁用 |
| `--active-prefill-tokens-threshold-frac` | `DYN_ACTIVE_PREFILL_TOKENS_THRESHOLD_FRAC` | `10.0` | prefill 繁忙判定相对 `max_num_batched_tokens` 的比例阈值。与绝对阈值是 OR 关系。传 `None` 禁用 |

## 模型发现

| CLI 参数 | 环境变量 | 默认值 | 说明 |
|-------------|---------|---------|-------------|
| `--namespace` | `DYN_NAMESPACE` | — | 精确匹配的模型发现 namespace |
| `--namespace-prefix` | `DYN_NAMESPACE_PREFIX` | — | 模型发现 namespace 前缀（例如 `ns` 同时命中 `ns` 与 `ns-abc123`）。优先级高于 `--namespace` |
| `--model-name` | `DYN_MODEL_NAME` | — | 覆盖模型名 |
| `--model-path` | `DYN_MODEL_PATH` | — | 本地模型目录路径（用于私有/自定义模型） |
| `--kv-cache-block-size` | `DYN_KV_CACHE_BLOCK_SIZE` | — | 覆盖 KV cache block 大小 |

## 基础设施

| CLI 参数 | 环境变量 | 默认值 | 说明 |
|-------------|---------|---------|-------------|
| `--discovery-backend` | `DYN_DISCOVERY_BACKEND` | `etcd` | 服务发现：`kubernetes`、`etcd`、`file`、`mem` |
| `--request-plane` | `DYN_REQUEST_PLANE` | `tcp` | 请求分发：`tcp`（最快）、`nats`、`http` |
| `--event-plane` | `DYN_EVENT_PLANE` | `nats` | 事件发布：`nats`、`zmq` |

## KServe gRPC

| CLI 参数 | 环境变量 | 默认值 | 说明 |
|-------------|---------|---------|-------------|
| `--kserve-grpc-server` / `--no-kserve-grpc-server` | `DYN_KSERVE_GRPC_SERVER` | `false` | 启动 KServe gRPC v2 server |
| `--grpc-metrics-port` | `DYN_GRPC_METRICS_PORT` | `8788` | gRPC 服务的 HTTP metrics 端口 |

KServe 消息格式与集成细节请参见 [Frontend Guide](frontend-guide.md)。

## 监控

| CLI 参数 | 环境变量 | 默认值 | 说明 |
|-------------|---------|---------|-------------|
| `--metrics-prefix` | `DYN_METRICS_PREFIX` | `dynamo_frontend` | frontend Prometheus 指标的前缀 |
| `--dump-config-to` | `DYN_DUMP_CONFIG_TO` | — | 把解析后的配置 dump 到指定文件路径 |

## Tokenizer

| CLI 参数 | 环境变量 | 默认值 | 说明 |
|-------------|---------|---------|-------------|
| `--tokenizer` | `DYN_TOKENIZER` | `default` | tokenizer：`default`（HuggingFace）或 `fastokens`（高性能 Rust tokenizer）。参见 [Tokenizer](Tokenizer.md) |

## 实验性

| CLI 参数 | 环境变量 | 默认值 | 说明 |
|-------------|---------|---------|-------------|
| `--enable-anthropic-api` | `DYN_ENABLE_ANTHROPIC_API` | `false` | 启用 `/v1/messages`（Anthropic Messages API） |
| `--dyn-chat-processor` | `DYN_CHAT_PROCESSOR` | `dynamo` | Chat 处理器：`dynamo` 或 `vllm` |
| `--dyn-debug-perf` | `DYN_DEBUG_PERF` | `false` | 输出预处理函数的逐函数耗时（仅 vllm processor） |
| `--dyn-preprocess-workers` | `DYN_PREPROCESS_WORKERS` | `0` | **[实验性]** 用于预处理与输出处理的 worker 进程数。取值 `> 0` 时，会创建一个含 N 个 worker 的 `ProcessPoolExecutor`，把 CPU 密集型工作（tokenization、chat template 渲染、detokenization）从主事件循环卸载到独立进程，每个进程拥有各自的 GIL，从而缓解单进程 GIL 竞争、提升高并发下的吞吐。`0`（默认）表示所有处理都在主事件循环上完成，不额外起进程。启动时会预热 worker 池（提前加载 tokenizer 等）。仅 `sglang` chat processor 实际支持；`vllm` processor 在该值非 0 时会直接报错拒绝启动 |
| `-i` / `--interactive` | `DYN_INTERACTIVE` | `false` | 交互式文本对话模式 |

## HTTP Endpoint

frontend 暴露以下 HTTP endpoint：

### OpenAI 兼容

| 方法 | 路径 | 说明 |
|--------|------|-------------|
| `POST` | `/v1/chat/completions` | Chat completions（流式与非流式） |
| `POST` | `/v1/completions` | 文本 completions |
| `POST` | `/v1/embeddings` | 文本 embeddings |
| `POST` | `/v1/responses` | Responses API |
| `POST` | `/v1/images/generations` | 图像生成 |
| `POST` | `/v1/videos/generations` | 视频生成 |
| `POST` | `/v1/videos/generations/stream` | 视频生成（流式） |
| `GET` | `/v1/models` | 列出可用模型 |

### Anthropic（实验性）

| 方法 | 路径 | 说明 |
|--------|------|-------------|
| `POST` | `/v1/messages` | Anthropic Messages API（需 `--enable-anthropic-api`） |
| `POST` | `/v1/messages/count_tokens` | Anthropic API 的 token 计数 |

### 基础设施

| 方法 | 路径 | 说明 |
|--------|------|-------------|
| `GET` | `/health` | 健康检查 |
| `GET` | `/live` | 存活检查 |
| `GET` | `/metrics` | Prometheus metrics |
| `GET` | `/openapi.json` | OpenAPI 规范 |
| `GET` | `/docs` | Swagger UI |
| `POST` | `/busy_threshold` | 设置繁忙阈值 |
| `GET` | `/busy_threshold` | 获取当前繁忙阈值 |

### 自定义 endpoint 路径

所有 endpoint 路径都可以通过环境变量覆盖：

| 环境变量 | 默认路径 |
|---------|-------------|
| `DYN_HTTP_SVC_CHAT_PATH_ENV` | `/v1/chat/completions` |
| `DYN_HTTP_SVC_CMP_PATH_ENV` | `/v1/completions` |
| `DYN_HTTP_SVC_EMB_PATH_ENV` | `/v1/embeddings` |
| `DYN_HTTP_SVC_RESPONSES_PATH_ENV` | `/v1/responses` |
| `DYN_HTTP_SVC_MODELS_PATH_ENV` | `/v1/models` |
| `DYN_HTTP_SVC_ANTHROPIC_PATH_ENV` | `/v1/messages` |
| `DYN_HTTP_SVC_HEALTH_PATH_ENV` | `/health` |
| `DYN_HTTP_SVC_LIVE_PATH_ENV` | `/live` |
| `DYN_HTTP_SVC_METRICS_PATH_ENV` | `/metrics` |

## 已弃用

| CLI 参数 | 环境变量 | 说明 |
|-------------|---------|-------------|
| `--router-durable-kv-events` | `DYN_ROUTER_DURABLE_KV_EVENTS` | 改用 event-plane 本地 indexer |

## 另请参阅

- [Frontend Overview](README.md) —— 快速上手与特性矩阵
- [Frontend Guide](frontend-guide.md) —— KServe gRPC 配置
- [NVIDIA Request Extensions (nvext)](nvext.md) —— 自定义请求字段
- [Configuration and Tuning](../router/router-configuration.md) —— 详细的路由配置
- [Metrics](../../observability/metrics.md) —— 可用的 Prometheus 指标
- [Fault Tolerance](../../fault-tolerance/README.md) —— 请求迁移与拒绝

