---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Reference Guide
subtitle: Architecture, configuration, and operational details for the SGLang backend
---

## 概述

Dynamo 中的 SGLang 后端（backend）采用模块化架构，`main.py` 会根据 worker 类型分发到对应的初始化模块。每种 worker 类型都有自己的 init 模块、请求处理器、健康检查与注册逻辑。

Dynamo SGLang 使用 SGLang 自带的参数解析器 —— 所有 SGLang 引擎参数（如 `--model-path`、`--tp`、`--trust-remote-code`）都会原样传递。Dynamo 在此之上增加了用于 worker 模式选择、tokenizer 控制以及解耦（disaggregation）配置的参数。

### Worker 类型

| Worker 类型 | 说明 |
|------------|-------------|
| **Decode** *（默认）* | 标准 LLM 推理（聚合或解耦的 decode） |
| **Prefill** | 解耦 prefill 阶段（`--disaggregation-mode prefill`） |
| **Embedding** | 文本嵌入模型（`--embedding-worker`） |
| **Multimodal Encode** | 面向 frontend：视觉编码、生成 embedding（`--multimodal-encode-worker`） |
| **Multimodal Worker** | 带多模态数据的 LLM 推理（`--multimodal-worker`） |
| **Multimodal Prefill** | 多模态解耦的 prefill 阶段（`--multimodal-worker --disaggregation-mode prefill`） |
| **Image Diffusion** | 通过 DiffGenerator 进行图像生成（`--image-diffusion-worker`） |
| **Video Generation** | 通过 DiffGenerator 进行 text/image 转视频（`--video-generation-worker`） |
| **LLM Diffusion** | 类似 LLaDA 的扩散语言模型（`--dllm-algorithm <algo>`） |

## 参数参考

### Dynamo 特有参数

这些是 Dynamo 在 SGLang 自带参数之上新增的参数。

| 参数 | 环境变量 | 默认值 | 说明 |
|----------|---------|---------|-------------|
| `--endpoint` | `DYN_ENDPOINT` | 自动生成 | 格式为 `dyn://namespace.component.endpoint` 的 Dynamo 端点 |
| `--use-sglang-tokenizer` | `DYN_SGL_USE_TOKENIZER` | `false` | **[已废弃]** 请改在 frontend 上使用 `--dyn-chat-processor sglang`。参见 [SGLang Chat Processor](sglang-chat-processor.md)。 |
| `--dyn-tool-call-parser` | `DYN_TOOL_CALL_PARSER` | `None` | [Tool call](../../agents/tool-calling.md#supported-tool-call-parsers) 解析器（覆盖 SGLang 的 `--tool-call-parser`） |
| `--dyn-reasoning-parser` | `DYN_REASONING_PARSER` | `None` | 用于 chain-of-thought 模型的 [Reasoning](../../agents/reasoning.md#supported-reasoning-parsers) 解析器 |
| `--custom-jinja-template` | `DYN_CUSTOM_JINJA_TEMPLATE` | `None` | 自定义 chat template 路径（与 `--use-sglang-tokenizer` 不兼容） |
| `--embedding-worker` | `DYN_SGL_EMBEDDING_WORKER` | `false` | 以 embedding worker 方式运行（同时设置 SGLang 的 `--is-embedding`） |
| `--multimodal-encode-worker` | `DYN_SGL_MULTIMODAL_ENCODE_WORKER` | `false` | 以 [multimodal](../../features/multimodal/multimodal-sglang.md) encode worker 运行（面向 frontend） |
| `--multimodal-worker` | `DYN_SGL_MULTIMODAL_WORKER` | `false` | 以多模态 LLM worker 运行 |
| `--image-diffusion-worker` | `DYN_SGL_IMAGE_DIFFUSION_WORKER` | `false` | 以 [image diffusion](sglang-diffusion.md#image-diffusion) worker 运行 |
| `--video-generation-worker` | `DYN_SGL_VIDEO_GENERATION_WORKER` | `false` | 以 [video generation](sglang-diffusion.md#video-generation) worker 运行 |
| `--disagg-config` | `DYN_SGL_DISAGG_CONFIG` | `None` | 解耦配置 YAML 文件路径 |
| `--disagg-config-key` | `DYN_SGL_DISAGG_CONFIG_KEY` | `None` | 用于在解耦配置中选择的 key（如 `prefill`、`decode`） |

<Note>
`--disagg-config` 与 `--disagg-config-key` 必须同时提供。所选小节会被写入临时 YAML 文件，并通过 SGLang 的 `--config` flag 传入。
</Note>

两个 flag 当前支持的解析器名称分别记录在 [Tool Calling](../../agents/tool-calling.md#supported-tool-call-parsers) 与 [Reasoning](../../agents/reasoning.md#supported-reasoning-parsers) 中。

## Tokenizer 行为

默认情况下，Dynamo 通过其基于 Rust 的 frontend 处理 tokenization 与 detokenization，并将 `input_ids` 传给 SGLang。这样所有 frontend 端点（`v1/chat/completions`、`v1/completions`、`v1/embeddings`）都可工作。

如需 SGLang 原生的预处理（tool calling、reasoning 解析、chat template），请在 frontend 上使用 `--dyn-chat-processor sglang`。架构与用法参见 [SGLang Chat Processor](sglang-chat-processor.md)。

<Warning>
`--use-sglang-tokenizer` 已废弃。请改在 frontend 上使用 `--dyn-chat-processor sglang`，它提供同样的 SGLang 原生处理，并支持 KV 路由与 completions 端点。
</Warning>

## 请求取消（Request Cancellation）

当客户端断开连接时，Dynamo 会自动跨所有 worker 取消进行中的请求，释放计算资源。一个后台取消监视器会检测到断开并中止 SGLang 请求。

| 模式 | Prefill | Decode |
|------|---------|--------|
| **聚合（Aggregated）** | ✅ | ✅ |
| **解耦（Disaggregated）** | ⚠️ | ✅ |

<Warning>解耦模式下远端 prefill 期间的取消当前不支持。</Warning>

关于取消架构的细节，参见 [Request Cancellation](../../fault-tolerance/request-cancellation.md)。

## 优雅停机（Graceful Shutdown）

SGLang worker 使用 Dynamo 的优雅停机机制。当收到 `SIGTERM` 或 `SIGINT` 时：

1. **从 discovery 注销**：worker 会从服务发现中移除，新请求不再路由至此
2. **宽限期**：允许进行中的请求完成
3. **延迟处理器**：在宽限期之后调用 SGLang 内部信号处理器（启动期间通过 monkey-patch `loop.add_signal_handler` 捕获）

这样可在滚动更新或缩容期间确保零请求丢失。

更多细节参见 [Graceful Shutdown](../../fault-tolerance/graceful-shutdown.md)。

## 健康检查（Health Checks）

每种 worker 类型都有针对完整推理流水线进行验证的专用健康检查负载：

| Worker 类型 | 健康检查策略 |
|------------|----------------------|
| Decode / 聚合 | 简短生成请求（`max_new_tokens=1`） |
| Prefill | 包装好的、prefill 专用的请求结构 |
| Image Diffusion | 最小化的图像生成请求 |
| Video Generation | 最小化的视频生成请求 |
| Embedding | 标准 embedding 请求 |

健康检查会注册到 Dynamo 运行时，由 frontend 或 Kubernetes liveness probe 调用。整体健康检查架构参见 [Health Checks](../../observability/health-checks.md)。

## 指标与 KV 事件

### Prometheus 指标（metrics）

通过在 worker 上加 `--enable-metrics` 来启用指标。设置 `DYN_SYSTEM_PORT` 以暴露 `/metrics` 端点：

```bash
DYN_SYSTEM_PORT=8081 python -m dynamo.sglang --model-path Qwen/Qwen3-0.6B --enable-metrics
```

SGLang 引擎指标（`sglang:*` 前缀）与 Dynamo 运行时指标（`dynamo_*` 前缀）会从同一个端点对外提供。

指标细节参见 [SGLang Observability](sglang-observability.md)。可视化搭建参见 [Prometheus + Grafana](../../observability/prometheus-grafana.md)。

### KV 事件

配置 `--kv-events-config` 后，worker 会发布 KV 缓存（KV cache）事件（block 创建/删除），供 [KV 感知路由](../../components/router/README.md) 使用。事件由 SGLang 的 scheduler 通过 ZMQ 发布，再经由 Dynamo 的事件平面转发。

在 DP attention 模式（`--enable-dp-attention`）下，发布器会为节点上每个 DP rank 处理一份独立的 KV 事件流。

## 引擎路由（Engine Routes）

SGLang worker 通过 Dynamo 的 system server 暴露运维端点：

| 路由 | 说明 |
|-------|-------------|
| `/engine/start_profile` | 启动 PyTorch profiling |
| `/engine/stop_profile` | 停止 profiling 并保存 trace |
| `/engine/release_memory_occupation` | 为维护释放 GPU 内存 |
| `/engine/resume_memory_occupation` | 释放后恢复 GPU 内存 |
| `/engine/update_weights_from_distributor` | 通过 distributor 更新模型权重 |
| `/engine/update_weights_from_disk` | 从磁盘更新模型权重 |
| `/engine/update_weight_version` | 更新权重版本元数据 |

## 另请参阅

- **[Examples](sglang-examples.md)**：所有部署模式
- **[Disaggregation](sglang-disaggregation.md)**：P/D 架构与 KV 传输
- **[Diffusion](sglang-diffusion.md)**：LLM、图像与视频扩散模型
- **[Configuration and Tuning](../../components/router/router-configuration.md)**：KV 感知路由配置
