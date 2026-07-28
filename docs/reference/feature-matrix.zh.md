---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Feature Matrix
---

本文给出了 Dynamo 关键特性在各受支持后端（backend）上的兼容性矩阵。

*更新于 Dynamo v1.1.1*

**图例：**
*   ✅ ：支持
*   🚧 ：开发中 / 实验性 / 受限

## 快速对比

| 特性 | SGLang | TensorRT-LLM | vLLM | 来源 |
| :--- | :---: | :---: | :---: | :--- |
| **解耦服务（Disaggregated Serving）** | ✅ | ✅ | ✅ | [设计文档][disagg] |
| **KV-Aware 路由** | ✅ | ✅ | ✅ | [Router 文档][kv-routing] |
| **基于 SLA 的 Planner** | ✅ | ✅ | ✅ | [Planner 文档][planner] |
| **KV Block Manager** | 🚧 | ✅ | ✅ | [KVBM 文档][kvbm] |
| **多模态（图像）** | ✅ | ✅ | ✅ | [多模态文档][mm] |
| **多模态（视频）** | ✅ | | ✅ | [多模态文档][mm] |
| **多模态（音频）** | | | 🚧 | [多模态文档][mm] |
| **请求迁移（Request Migration）** | ✅ | 🚧 | ✅ | [Migration 文档][migration] |
| **请求取消（Request Cancellation）** | 🚧 | ✅ | ✅ | 各后端 README |
| **LoRA** | | | ✅ | [K8s 指南][lora] |
| **工具调用（Tool Calling）** | ✅ | ✅ | ✅ | [Tool Calling 文档][tools] |
| **推测解码（Speculative Decoding）** | 🚧 | ✅ | ✅ | 各后端 README |
| **Dynamo Snapshot** | ✅ | | ✅ | [Snapshot 文档][snapshot] |

## 1. vLLM 后端

vLLM 在 Dynamo 中的特性覆盖最广，全面支持解耦服务、KV-aware 路由、KV block 管理、
LoRA 适配器，以及包括视频与音频在内的多模态推理（inference）。

*来源：[docs/backends/vllm/README.md][vllm-readme]*

| 特性 | 解耦服务 | KV-Aware 路由 | 基于 SLA 的 Planner | KV Block Manager | 多模态 | 请求迁移 | 请求取消 | LoRA | 工具调用 | 推测解码 |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **解耦服务** | — | | | | | | | | | |
| **KV-Aware 路由** | ✅ | — | | | | | | | | |
| **基于 SLA 的 Planner** | ✅ | ✅ | — | | | | | | | |
| **KV Block Manager** | ✅ | ✅ | ✅ | — | | | | | | |
| **多模态** | ✅ | <sup>1</sup> | — | ✅ | — | | | | | |
| **请求迁移** | ✅ | ✅ | ✅ | ✅ | ✅ | — | | | | |
| **请求取消** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — | | | |
| **LoRA** | ✅ | ✅<sup>2</sup> | — | ✅ | — | ✅ | ✅ | — | | |
| **工具调用** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — | |
| **推测解码** | ✅ | ✅ | — | ✅ | — | ✅ | ✅ | — | ✅ | — |

> **注：**
> 1. **多模态 + KV-Aware 路由**：KV 路由器使用基于 token 的哈希，目前还不支持图像 / 视频哈希，因此会回退（fallback）到随机 / 轮询路由。 ([来源][kv-routing])
> 2. **KV-Aware LoRA 路由**：vLLM 支持基于 LoRA 适配器亲和性来路由请求。
> 3. **音频支持**：vLLM 支持 Qwen2-Audio 等音频模型（实验性）。 ([来源][mm-vllm])
> 4. **视频支持**：vLLM 支持基于帧采样的视频输入。 ([来源][mm-vllm])
> 5. **推测解码**：已有 Eagle3 支持的相关文档。 ([来源][vllm-spec])

## 2. SGLang 后端

SGLang 针对高吞吐服务做了优化，配备了快速的原语，对解耦服务、KV-aware 路由、
请求迁移有较好的支持。

*来源：[docs/backends/sglang/README.md][sglang-readme]*

| 特性 | 解耦服务 | KV-Aware 路由 | 基于 SLA 的 Planner | KV Block Manager | 多模态 | 请求迁移 | 请求取消 | LoRA | 工具调用 | 推测解码 |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **解耦服务** | — | | | | | | | | | |
| **KV-Aware 路由** | ✅ | — | | | | | | | | |
| **基于 SLA 的 Planner** | ✅ | ✅ | — | | | | | | | |
| **KV Block Manager** | 🚧 | 🚧 | 🚧 | — | | | | | | |
| **多模态** | ✅<sup>2</sup> | <sup>1</sup> | — | 🚧 | — | | | | | |
| **请求迁移** | ✅ | ✅ | ✅ | 🚧 | ✅ | — | | | | |
| **请求取消** | 🚧<sup>3</sup> | ✅ | ✅ | 🚧 | 🚧 | ✅ | — | | | |
| **LoRA** | | | | 🚧 | | | | — | | |
| **工具调用** | ✅ | ✅ | ✅ | 🚧 | ✅ | ✅ | ✅ | | — | |
| **推测解码** | 🚧 | 🚧 | — | 🚧 | — | 🚧 | — | | 🚧 | — |

> **注：**
> 1. **多模态 + KV-Aware 路由**：暂不支持。 ([来源][kv-routing])
> 2. **多模态形态**：仅支持 **E/PD** 与 **E/P/D**（需要独立的视觉 encoder）。**不**支持简单的 Aggregated（EPD）以及 Traditional Disagg（EP/D）。 ([来源][mm-sglang])
> 3. **请求取消**：在解耦模式下，远端 prefill 阶段的取消暂不支持。 ([来源][sglang-readme])
> 4. **推测解码**：代码层已有钩子（publisher 中的 `spec_decode_stats`），但尚无示例与文档。

## 3. TensorRT-LLM 后端

TensorRT-LLM 提供极致的推理性能与优化，对 KVBM 集成和解耦服务都有完善支持。

*来源：[docs/backends/trtllm/README.md][trtllm-readme]*

| 特性 | 解耦服务 | KV-Aware 路由 | 基于 SLA 的 Planner | KV Block Manager | 多模态 | 请求迁移 | 请求取消 | LoRA | 工具调用 | 推测解码 |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **解耦服务** | — | | | | | | | | | |
| **KV-Aware 路由** | ✅ | — | | | | | | | | |
| **基于 SLA 的 Planner** | ✅ | ✅ | — | | | | | | | |
| **KV Block Manager** | ✅ | ✅ | ✅ | — | | | | | | |
| **多模态** | ✅<sup>1</sup> | <sup>2</sup> | — | ✅ | — | | | | | |
| **请求迁移** | ✅ | ✅ | ✅ | ✅ | 🚧 | — | | | | |
| **请求取消** | ✅<sup>3</sup> | ✅<sup>3</sup> | ✅<sup>3</sup> | ✅<sup>3</sup> | ✅<sup>3</sup> | ✅<sup>3</sup> | — | | | |
| **LoRA** | | | | | | | | — | | |
| **工具调用** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | | — | |
| **推测解码** | ✅ | ✅ | — | ✅ | — | ✅ | ✅ | | ✅ | — |

> **注：**
> 1. **多模态解耦**：完全支持 **EP/D**（Traditional）模式。**E/P/D**（Full Disaggregation）尚在开发中，目前仅支持预先计算好的 embedding。 ([来源][mm-trtllm])
> 2. **多模态 + KV-Aware 路由**：暂不支持。KV 路由器目前仅跟踪基于 token 的 block。 ([来源][kv-routing])
> 3. **请求取消**：受已知问题影响，TensorRT-LLM 引擎暂时不会被通知到请求取消，因此被取消请求的相关资源不会被释放。

---


{/* Backend READMEs — paths relative to rendered URL /resources/feature-matrix */}
[vllm-readme]: ../backends/v-llm
[sglang-readme]: ../backends/sg-lang
[trtllm-readme]: ../backends/tensor-rt-llm

{/* Design Docs */}
[disagg]: ../design-docs/disaggregated-serving
[kv-routing]: ../components/router/router-guide
[planner]: ../components/planner
[kvbm]: ../components/kvbm
[migration]: ../user-guides/fault-tolerance/request-migration
[tools]: ../user-guides/agents/tool-calling

{/* Multimodal */}
[mm]: ../user-guides/multimodal
[mm-vllm]: https://github.com/ai-dynamo/dynamo/blob/main/docs/features/multimodal/multimodal-vllm.md
[mm-trtllm]: https://github.com/ai-dynamo/dynamo/blob/main/docs/features/multimodal/multimodal-trtllm.md
[mm-sglang]: https://github.com/ai-dynamo/dynamo/blob/main/docs/features/multimodal/multimodal-sglang.md

{/* Feature-specific */}
[lora]: ../kubernetes-deployment/deployment-guide/managing-models-with-dynamo-model
[vllm-spec]: ../additional-resources/speculative-decoding/speculative-decoding-with-v-llm
[trtllm-eagle]: ../additional-resources/tensor-rt-llm-details/llama-4-eagle

{/* Dynamo Snapshot */}
[snapshot]: ../kubernetes-deployment/deployment-guide/snapshot
