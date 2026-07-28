---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 参考指南
subtitle: vLLM 后端的配置、参数与运维细节
---

# 参考指南

## 概述

Dynamo 中的 vLLM 后端将 [vLLM](https://github.com/vllm-project/vllm) 引擎集成到 Dynamo 的分布式运行时中，支持分离式服务、KV 感知路由与请求取消。Dynamo 利用 vLLM 原生的 KV 缓存事件、基于 NIXL 的传输机制与指标上报。

Dynamo vLLM 使用 vLLM 原生的参数解析器——所有 vLLM 引擎参数都会被透传。Dynamo 添加了自己的参数用于分离模式、KV 传输和 prompt embeddings。

## 参数参考

vLLM 后端接受所有上游 vLLM 引擎参数及 Dynamo 特定参数。最权威的来源始终是 CLI：

```bash
python -m dynamo.vllm --help
```

`--help` 输出按以下分组组织：

- **Dynamo Runtime Options** — 命名空间、发现后端、请求 / 事件平面、端点类型、tool/reasoning 解析器以及自定义 chat template。这些在所有 Dynamo 后端通用，使用 `DYN_*` 环境变量。
- **Dynamo vLLM Options** — 分离模式、tokenizer 选择、sleep 模式、多模态标志、vLLM-Omni 流水线配置、headless 模式以及 ModelExpress。这些使用 `DYN_VLLM_*` 环境变量。
- **vLLM Engine Options** — 所有原生 vLLM 参数（`--model`、`--tensor-parallel-size`、`--kv-transfer-config`、`--kv-events-config`、`--enable-prefix-caching` 等）。参见 [vLLM serve args 文档](https://docs.vllm.ai/en/stable/configuration/serve_args.html)。

### 工具与推理解析器

当模型输出工具调用或推理内容时，使用 `--dyn-tool-call-parser` 与 `--dyn-reasoning-parser` 来匹配模型的输出格式。当前支持的取值见 [Tool Calling](../../agents/tool-calling.md#supported-tool-call-parsers) 与 [Reasoning](../../agents/reasoning.md#supported-reasoning-parsers)。

### Prompt Embeddings

Dynamo 支持 [vLLM prompt embeddings](https://docs.vllm.ai/en/stable/features/prompt_embeds.html) —— 预计算的嵌入会绕过 Rust 前端的 tokenization，并在 worker 中解码为张量。

- 使用 `--enable-prompt-embeds` 启用（默认禁用）
- 嵌入通过 Completions API 中的 `prompt_embeds` 字段以 base64 编码的 PyTorch 张量发送
- 对于较大的嵌入，NATS 必须配置 15MB 的最大消息体（默认部署中已设置）

## KV 事件的哈希一致性

使用 KV 感知路由时，应确保跨进程的哈希是确定性的，以避免 radix 树不一致。从下列方式中选择其一：

- 当依赖 Python 内置哈希用于 prefix 缓存时，为所有 vLLM 进程设置 `PYTHONHASHSEED=0`。
- 如果你的 vLLM 版本支持，配置确定性的 prefix 缓存算法：

```bash
vllm serve ... --enable-prefix-caching --prefix-caching-algo sha256
```

确定性事件 ID 的高层说明见 [Router Design](../../design-docs/router-design.md#deterministic-event-ids)。

## 优雅关闭

vLLM worker 使用 Dynamo 的优雅关闭机制。收到 `SIGTERM` 或 `SIGINT` 时：

1. **从发现服务注销**：worker 从服务发现中移除，不再有新请求被路由到它
2. **宽限期**：允许进行中的请求完成（通过 `DYN_GRACEFUL_SHUTDOWN_GRACE_PERIOD_SECS` 配置，默认 5s）
3. **资源清理**：释放引擎资源和临时文件（Prometheus 目录、LoRA 适配器）

所有 vLLM 端点都使用 `graceful_shutdown=True`，意味着它们会等待进行中的请求完成后再退出。一个内部的 `VllmEngineMonitor` 也会每 2 秒检查引擎健康状态，当引擎无响应时启动关闭流程。

更多细节见[优雅关闭](../../fault-tolerance/graceful-shutdown.md)。

## 健康检查

每种 worker 类型都有一个特定的健康检查 payload，用于验证完整的推理流水线：

| Worker 类型 | 健康检查策略 |
|------------|----------------------|
| Decode / 聚合式 | 使用模型 BOS token 的短生成请求（`max_tokens=1`） |
| Prefill | 与 decode 相同结构的 payload，适配为 prefill 请求格式 |
| vLLM-Omni | 使用模型 BOS token 通过 AsyncOmni 发起的短生成请求 |

健康检查注册到 Dynamo 运行时，由前端或 Kubernetes liveness 探针调用。可通过 `DYN_HEALTH_CHECK_PAYLOAD` 环境变量覆盖 payload。整体健康检查架构见[健康检查](../../observability/health-checks.md)。

## 请求取消

当用户取消请求（例如从前端断开连接）时，该请求会在所有 worker 上被自动取消，释放算力资源。

| | 预填充 | 解码 |
|-|---------|--------|
| **聚合式** | ✅ | ✅ |
| **分离式** | ✅ | ✅ |

更多细节见[请求取消架构](../../fault-tolerance/request-cancellation.md)文档。

## 请求迁移

Dynamo 支持[请求迁移](../../fault-tolerance/request-migration.md)以优雅地处理 worker 故障。启用后，当某个 worker 在生成过程中失败时，请求可以被自动迁移到健康的 worker 上。配置详情请参阅[请求迁移架构](../../fault-tolerance/request-migration.md)文档。

## 另请参阅

- **[示例](vllm-examples.md)**：所有部署模式与启动脚本
- **[vLLM README](README.md)**：快速开始与特性概览
- **[可观测性](vllm-observability.md)**：指标与监控设置
- **[配置与调优](../../components/router/router-configuration.md)**：KV 感知路由配置
- **[容错](../../fault-tolerance/README.md)**：请求迁移、取消与优雅关闭
