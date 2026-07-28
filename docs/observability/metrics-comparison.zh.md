---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Engine Metrics Comparison
---

## 概述

本文档对比 Dynamo 所支持的三种推理后端所暴露的 Prometheus 指标（metrics）：**vLLM**、**SGLang** 和 **TensorRT-LLM**。

关于 Dynamo 自身运行时（runtime）暴露的指标（`dynamo_*`），请参见 [Metrics Guide](metrics.md)。各后端的具体配置和细节，请参见：

- [vLLM Observability](../backends/vllm/vllm-observability.md)
- [SGLang Observability](../backends/sglang/sglang-observability.md)
- [TensorRT-LLM Observability](../backends/trtllm/trtllm-observability.md)

| 框架 | 指标前缀 | 独有指标数 | 测试版本 | 必需参数 |
|-----------|---------------|----------------|----------------|----------------|
| vLLM | `vllm:` | 36 | v0.19.0 | `DYN_SYSTEM_PORT=8081` |
| SGLang | `sglang:` | 48 | v0.5.9 | `DYN_SYSTEM_PORT=8081 --enable-metrics` |
| TensorRT-LLM | `trtllm_` | 14 | v1.3.0rc9 | `DYN_SYSTEM_PORT=8081 --publish-events-and-metrics` |

> **说明：** 指标名称和数量会随引擎版本更新而变化。所有指标均通过在 2026-04-10 运行 Dynamo v1.0.0 时的实时抓取进行验证。请以你实际部署中 `/metrics` 端点的输出为准。

所有框架共享来自 Dynamo 运行时的通用 `dynamo_component_*` 指标。

## Dynamo Worker 通用指标

下列后端指标在所有后端的 worker 端口（`:8081/metrics`）上均可获取。基于 2026-04-10 的实时抓取验证。

关于 Dynamo 前端和路由（router）指标（`dynamo_frontend_*`、`dynamo_component_router_*`），请参见 [Metrics Guide](metrics.md)。

| 指标名 | 类型 | 说明 |
|-------------|------|-------------|
| `dynamo_component_cancellation_total` | counter | 工作处理器（work handler）取消的请求总数 |
| `dynamo_component_gpu_cache_usage_percent` | gauge | GPU 缓存使用率（百分比，0.0–1.0） |
| `dynamo_component_inflight_requests` | gauge | 当前正在处理中的请求数量 |
| `dynamo_component_model_load_time_seconds` | gauge | 模型加载时间（秒） |
| `dynamo_component_request_bytes_total` | counter | 请求总接收字节数 |
| `dynamo_component_request_duration_seconds` | histogram | 请求处理耗时 |
| `dynamo_component_requests_total` | counter | 已处理请求总数 |
| `dynamo_component_response_bytes_total` | counter | 响应总发送字节数 |
| `dynamo_component_total_blocks` | gauge | 该 worker 上可用的 KV 缓存（KV cache）块总数 |
| `dynamo_component_uptime_seconds` | gauge | DistributedRuntime 总运行时长 |

## 各框架指标对比

下列指标是**来自引擎自身的透传指标（pass-through metrics）**——Dynamo 通过其 `/metrics` 端点暴露这些指标，但并不生成它们。表中指标名**不含前缀**，实际指标名分别带有 `vllm:`、`sglang:` 或 `trtllm_` 前缀。

| 类别 | 指标 | vLLM | SGLang | TensorRT-LLM |
|----------|--------|------|--------|---------------|
| **请求状态与队列** | | | | |
| | 运行中的请求 | `num_requests_running` | `num_running_reqs` | - |
| | 等待中/排队的请求 | `num_requests_waiting` | `num_queue_reqs` | - |
| | 排队时间 | `request_queue_time_seconds` | `queue_time_seconds` | `request_queue_time_seconds` |
| | 语法（grammar）队列 | - | `num_grammar_queue_reqs` | - |
| | 离线批处理运行中 | - | `num_running_reqs_offline_batch` | - |
| | 预填充（prefill）预分配队列 | - | `num_prefill_prealloc_queue_reqs` | - |
| | 预填充进行中队列 | - | `num_prefill_inflight_queue_reqs` | - |
| | 解码（decode）预分配队列 | - | `num_decode_prealloc_queue_reqs` | - |
| | 解码传输队列 | - | `num_decode_transfer_queue_reqs` | - |
| **延迟** | | | | |
| | 首 token 时延 | `time_to_first_token_seconds` | `time_to_first_token_seconds` | `time_to_first_token_seconds` |
| | token 间延迟 | `inter_token_latency_seconds` | `inter_token_latency_seconds` | - |
| | 端到端请求延迟 | `e2e_request_latency_seconds` | `e2e_request_latency_seconds` | `e2e_request_latency_seconds` |
| | 单 token 输出耗时 | `request_time_per_output_token_seconds` | - | `time_per_output_token_seconds` |
| | 推理时间 | `request_inference_time_seconds` | - | - |
| | 预填充时间 | `request_prefill_time_seconds` | - | - |
| | 解码时间 | `request_decode_time_seconds` | - | - |
| | 各阶段延迟 | - | `per_stage_req_latency_seconds` | - |
| **token 指标** | | | | |
| | Prompt/预填充 tokens | `prompt_tokens_total` | `prompt_tokens_total` | - |
| | 生成 tokens | `generation_tokens_total` | `generation_tokens_total` | - |
| | 单请求 prompt tokens（直方图） | `request_prompt_tokens` | - | - |
| | 单请求生成 tokens（直方图） | `request_generation_tokens` | - | - |
| | 迭代 tokens | `iteration_tokens_total` | - | - |
| | 最大生成 tokens | `request_max_num_generation_tokens` | - | - |
| | 实时 tokens | - | `realtime_tokens_total` | - |
| | 已使用 tokens | - | `num_used_tokens` | - |
| | 按来源分类的缓存 tokens | - | `cached_tokens_total` | - |
| | 预填充时已计算的 KV tokens | `request_prefill_kv_computed_tokens` | - | - |
| | 按来源分类的 prompt tokens | `prompt_tokens_by_source_total` | - | - |
| | 命中缓存的 prompt tokens | `prompt_tokens_cached_total` | - | - |
| | 重新计算的 prompt tokens | `prompt_tokens_recomputed_total` | - | - |
| **请求成功与中止** | | | | |
| | 请求成功（按原因分） | `request_success_total` | - | `request_success_total` |
| | 请求总数 | - | `num_requests_total` | - |
| | 中止的请求 | - | - | `num_aborted_requests_total` |
| **请求类型** | | | | |
| | 图像请求 | - | - | `request_type_image_total` |
| | 结构化输出请求 | - | - | `request_type_structured_output_total` |
| **KV 缓存与显存** | | | | |
| | KV 缓存使用率（%） | `kv_cache_usage_perc` | - | - |
| | KV 缓存命中率 | - | - | `kv_cache_hit_rate` |
| | KV 缓存利用率 | - | - | `kv_cache_utilization` |
| | token 使用量 | - | `token_usage` | - |
| | 最大总 token 数 | - | `max_total_num_tokens` | - |
| | SWA token 使用量 | - | `swa_token_usage` | - |
| | Mamba 使用量 | - | `mamba_usage` | - |
| | 待处理预分配 token 使用量 | - | `pending_prealloc_token_usage` | - |
| **前缀缓存（prefix cache）** | | | | |
| | 缓存命中率 | - | `cache_hit_rate` | - |
| | 缓存配置信息 | `cache_config_info` | `cache_config_info` | - |
| | 前缀缓存查询次数 | `prefix_cache_queries_total` | - | - |
| | 前缀缓存命中次数 | `prefix_cache_hits_total` | - | - |
| | 外部前缀缓存查询次数 | `external_prefix_cache_queries_total` | - | - |
| | 外部前缀缓存命中次数 | `external_prefix_cache_hits_total` | - | - |
| **多模态缓存** | | | | |
| | MM 缓存查询次数 | `mm_cache_queries_total` | - | - |
| | MM 缓存命中次数 | `mm_cache_hits_total` | - | - |
| **引擎状态** | | | | |
| | 引擎休眠状态 | `engine_sleep_state` | - | - |
| | 引擎启动时间 | - | `engine_startup_time` | - |
| | 引擎权重加载时间 | - | `engine_load_weights_time` | - |
| | 单 GPU 估算 FLOPs | `estimated_flops_per_gpu_total` | - | - |
| | 单 GPU 估算读字节数 | `estimated_read_bytes_per_gpu_total` | - | - |
| | 单 GPU 估算写字节数 | `estimated_write_bytes_per_gpu_total` | - | - |
| | CUDA 图状态 | - | `is_cuda_graph` | - |
| | CUDA 图执行次数 | - | `cuda_graph_passes_total` | - |
| | 利用率 | - | `utilization` | - |
| | 新 token 比例 | - | `new_token_ratio` | - |
| **抢占（preemption）与回退** | | | | |
| | 抢占次数 | `num_preemptions_total` | - | - |
| | 被回退的请求 | - | `num_retracted_reqs` | - |
| | 回退发生次数 | - | `num_retractions` | - |
| | 暂停中的请求 | - | `num_paused_reqs` | - |
| **请求参数** | | | | |
| | 请求参数 n | `request_params_n` | - | - |
| | 请求参数 max_tokens | `request_params_max_tokens` | - | - |
| **吞吐与性能** | | | | |
| | 生成吞吐 | - | `gen_throughput` | - |
| | 解码序列长度之和 | - | `decode_sum_seq_lens` | - |
| **路由** | | | | |
| | 运行中唯一路由键数量 | - | `num_unique_running_routing_keys` | - |
| | 路由键全部请求数 | - | `routing_key_all_req_count` | - |
| | 路由键运行中请求数 | - | `routing_key_running_req_count` | - |
| **投机解码（speculative decoding）** | | | | |
| | 投机接受长度 | - | `spec_accept_length` | - |
| | 投机接受率 | - | `spec_accept_rate` | - |
| **KV 传输** | | | | |
| | KV 传输速度（GB/s） | - | `kv_transfer_speed_gb_s` | `kv_transfer_speed_gb_s` |
| | KV 传输延迟 | - | `kv_transfer_latency_ms` | `kv_transfer_latency_seconds` |
| | KV 传输引导阶段（ms） | - | `kv_transfer_bootstrap_ms` | - |
| | KV 传输分配阶段（ms） | - | `kv_transfer_alloc_ms` | - |
| | KV 传输总量（MB） | - | `kv_transfer_total_mb` | - |
| | KV 传输字节数 | - | - | `kv_transfer_bytes` |
| | KV 传输成功次数 | - | - | `kv_transfer_success_total` |

