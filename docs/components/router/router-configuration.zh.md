---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 配置与调优
subtitle: 路由器标志、事件传输、负载跟踪与调优指引
---

本页汇总了适用于前端内嵌部署与独立部署的主要路由器标志。路由成本模型与 worker 选择行为请参阅[路由概念](router-concepts.md)。

## 路由行为

- `--router-kv-overlap-score-weight`：控制前缀缓存重叠在 prefill 成本计算中的重要性。值越高越改善首 token 时间（TTFT），代价是 token 间延迟（ITL）。设为 0 时，路由器忽略前缀缓存，使用纯负载均衡。默认 1。
- `--router-temperature`：通过对路由器成本 logits 进行 softmax 采样，控制 worker 选择的随机性。0（默认）保证确定性选择成本最低的 worker，更高的值引入更多随机性。
- `--router-track-prefill-tokens`：在 worker 成本模型中启用 prompt 侧的负载计入。如希望队列阈值、`active_prefill_tokens` 与 AIC prefill 负载衰减反映 prompt 工作量，应保持启用。
- `--router-prefill-load-model`：选择路由器的 prompt 侧负载模型。`none` 保持现有的静态 prompt 负载计入。`aic` 为每个被准入的请求预测一个期望的 prefill 持续时间，并仅对每个 worker 上最旧的活跃 prefill 请求懒衰减。
- `--router-queue-threshold`：prefill token 容量的队列阈值比例（默认：4.0）。当所有 worker 都超过 `max_num_batched_tokens` 的该比例时，路由器将入站请求保留在优先队列中，待容量释放后再发送。这种做法是延迟分发而非拒绝工作，因此路由决策可在请求实际发送给 worker 时使用最新的负载指标。它还通过 `nvext.agent_hints` 中的 `priority` hint 启用优先级调度。必须大于 0。设为 `None` 关闭排队。SGLang 后端 `max_num_batched_tokens` 的填充方式注意事项请见[调优指引](#tuning-guidelines)。
- `--router-queue-policy`：路由器队列的调度策略（默认：`fcfs`）。

`fcfs` 按调整后的到达时间（`priority_jump - arrival_offset`）排序，优化尾部 TTFT。
`lcfs` 按调整后的反向到达时间（`priority_jump + arrival_offset`）排序，主要用于受控对比实验。
`wspt` 按 `(1 + priority_jump) / isl_tokens` 排序，优化平均 TTFT。

对于 `--router-mode device-aware-weighted`，请将 `DYN_ENCODER_CUDA_TO_CPU_RATIO` 设为单个非 CPU worker 相对单个 CPU worker 的近似吞吐量比。默认 `8`。

## KV 事件传输与持久化

- `--no-router-kv-events`：禁用 KV 事件跟踪。默认情况下，路由器使用 KV 事件监控 worker 的 block 创建与删除。禁用后，路由器根据自身的路由决策预测缓存状态，并使用基于 TTL 的过期。
- `--router-durable-kv-events`：**已弃用。** 启用 KV 事件传输的 JetStream 模式。本地 indexer 模式下的事件平面订阅者现在是推荐路径。
- `--router-reset-states`：仅适用于 JetStream 模式（`--router-durable-kv-events`）。启动时通过清空 JetStream 事件流和 NATS 对象存储重置路由器状态，从全新状态开始。
- `--router-snapshot-threshold`：仅适用于 JetStream 模式（`--router-durable-kv-events`）。设置触发快照前 JetStream 中的消息数量。

## Block 跟踪

- `--no-router-track-active-blocks`：禁用对正在生成或解码阶段使用的活跃 block 的跟踪。在仅做 prefill 的 worker 上路由时禁用此项。
- `--router-track-output-blocks`：**实验性。** 启用生成期间的输出 block 跟踪。启用后，路由器在生成 token 时添加占位 block，并基于相对期望输出序列长度（`nvext` 中的 `agent_hints.osl`）的进度施加分数衰减。
- `--no-router-assume-kv-reuse`：当跟踪活跃 block 时，禁用 KV 缓存复用假设。在分离式部署中，如果传输的 block 在 decode 侧实际上不会去重，使用该选项很有用。
- `--no-router-track-prefill-tokens`：在路由器活跃负载模型中禁用 prompt 侧 prefill token 计入。适用于仅 decode 路由路径，prompt 处理已在别处完成。
- `--router-replica-sync`：默认禁用。启用基于 NATS 的路由器副本之间本地路由决策的同步。

## KV Indexer / Approx KV Indexer

- `--router-ttl-secs`：路由器本地缓存预测中 block 的存活时间（秒）。在使用 `--no-router-kv-events` 时默认 120.0 秒。
- `--router-event-threads`：KV indexer worker 线程数（默认：4）。值大于 1 时，使用并发 radix 树用于事件驱动路由，以及在 `--no-router-kv-events` 下用于近似路由；设为 1 强制使用单线程 indexer。

要为自定义推理引擎实现 KV 事件发布，请参阅 [KV Event Publishing for Custom Engines](../../integrations/kv-events-custom-engines.md)。
逐请求 agent hint（`priority`、`osl`、`speculative_prefill`）的细节请参阅 [NVIDIA Request Extensions (`nvext`)](../frontend/nvext.md#agent-hints)。

### 会话控制与粘性路由

当请求携带 `nvext.session_control` 时，KV 路由器会激活两个额外组件：

- **AgentController**：向 worker 的 `session_control` 端点发送会话生命周期 RPC（`open_session`、`close_session`）。事件平面客户端在首次会话请求时延迟初始化。
- **StickySessionRouter**：维护一个内存中的 `session_id -> worker_id` 亲和图，使用滑动窗口 TTL。同一 `session_id` 的后续请求会被路由到固定的 worker，绕过 KV 重叠打分。

这些组件随 `--router-mode kv` 自动启用——无需额外标志。不带 `session_control` 的请求不受影响，仍走标准的 KV 感知路由路径。会话控制目前需要带 `--enable-streaming-session` 的 SGLang 后端。详情见 [SGLang for Agentic Workloads -- Session Control](../../backends/sglang/agents.md#session-control-for-subagent-kv-isolation-experimental)。

## 调优指引

`--router-kv-overlap-score-weight` 是平衡 prefill 效率与 decode 负载的主要旋钮。以 prefill 为主的工作负载受益于更高的权重，将请求引向缓存重叠更佳的 worker，降低 TTFT。以 decode 为主的工作负载受益于较低的权重，更均匀地分布 decode 负载并降低 ITL。默认值 1.0 是一个合理的起点。该权重也可通过 `nvext.agent_hints.kv_overlap_score_weight` 在每请求级别覆盖。

当你不确定后端引擎是否正确发出 KV 事件时，使用 `--no-router-kv-events`。该模式下路由器回退到近似路由，从自身路由决策预测缓存状态并使用基于 TTL 的过期。

在分离式部署中，当 decode worker 不复用已传输的 KV block 时使用 `--no-router-assume-kv-reuse`。否则当存在重复时路由器会少计 decode block，导致负载估计不准确。

当某个路由器服务于仅 decode 流量且 prompt 处理已在别处完成时，使用 `--no-router-track-prefill-tokens`。这可让 decode 路由决策聚焦于 decode 侧负载，而不是在交接后短暂地把 prompt token 算到 decode worker 上。

当你的工作负载以输出为主，且希望路由器在负载均衡中考虑输出侧 KV 缓存的增长时，使用 `--router-track-output-blocks`。如果你还在每个请求中传入 `nvext.agent_hints.osl`，路由器会对输出 block 施加分数衰减，使临近完成的请求贡献更少未来负载。

`--router-queue-threshold` 控制何时将入站请求保留在优先队列中。当所有 worker 都超过配置的 `max_num_batched_tokens` 比例时，路由器等待，待容量释放后再发出工作。设为 `None` 完全禁用排队。

**SGLang 后端注意。** 自 [#8220](https://github.com/ai-dynamo/dynamo/pull/8220) 起，SGLang worker 在其 Model Deployment Card 中发布的 `max_num_batched_tokens` 的取值取决于 server 参数：

- 如果设置了 `--max-prefill-tokens`，MDC 的 `max_num_batched_tokens` 等于该值（每步 prefill 窗口——多数用户期望的值）。
- 如果未设置 `--max-prefill-tokens`，MDC 的 `max_num_batched_tokens` 回退到 SGLang `scheduler_info` 中的 `max_total_num_tokens`，即**整个 KV 缓存池**（以 token 为单位）。在大显存且 `mem-fraction-static` 较高的 GPU 上，这个池可能达数十万 token——远大于 `chunked-prefill-size`。

阈值按 `active_tokens > threshold * max_num_batched_tokens` 应用，这种回退会膨胀有效分母，使得像 `1.0` 这样的阈值实际上几乎不会触发排队。要在 SGLang 上获得最初设想的"每步 prefill 窗口的比例"语义，要么显式在 SGLang 后端设置 `--max-prefill-tokens` 使 MDC 值与 prefill 窗口一致；要么使用更小的 `--router-queue-threshold`（例如 `0.1`）来补偿被膨胀的分母。

当你希望 prompt 侧负载跟踪根据 AIC 预测的持续时间衰减最旧的活跃 prefill 请求，而不是保持 prompt 负载在首 token 前静止时，使用 `--router-prefill-load-model aic`。这需要 `--router-track-prefill-tokens` 和共享的 `--aic-*` 配置。

当你的工作负载混合了短请求和长请求并希望最小化平均 TTFT 时使用 `--router-queue-policy wspt`。当你希望最小化尾部 TTFT 时使用默认的 `fcfs`。

## Prometheus 指标

路由器在前端 HTTP 端口（默认 8000）的 `/metrics` 上暴露 Prometheus 指标：

- **路由器请求指标**（`dynamo_component_router_*`）：通过组件的指标层级注册，并通过 `drt_metrics` bridge 暴露在前端。在 KV 模式下按请求填充；在非 KV 模式下注册为零值。standalone 路由器也注册这些指标，可在设置 `DYN_SYSTEM_PORT` 时使用。
- **路由开销指标**（`dynamo_router_overhead_*`）与**逐 worker gauge**（`dynamo_frontend_worker_*`）：注册于前端自身的 Prometheus registry。这些是仅前端的，standalone 路由器不可用。

完整的路由器指标列表请参阅[指标参考](../../observability/metrics.md#router-metrics)。
