---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: "Dynamo 对 TokenSpeed 的 Day 0 支持"
subtitle: "在 Dynamo 中运行 TokenSpeed 的简短发布说明"
description: "Dynamo 通过 Dynamo 前端为 Kimi K2.5 添加了对 TokenSpeed 的 Day 0 支持。"
keywords: TokenSpeed, Dynamo, LightSeek, Kimi K2.5, disaggregated serving, KV cache routing, LLM inference
last-updated: May 6, 2026
---

[TokenSpeed](https://lightseek.org/blog/lightseek-tokenspeed.html)（[GitHub](https://github.com/lightseekorg/tokenspeed)）今日作为 LightSeek 面向智能体（agentic）工作负载的全新推理引擎发布。最初的代码仓库为预览版本，未来几周将陆续推出更多模型覆盖与运行时特性。

有两点值得特别指出。首先，TokenSpeed 包含了为 Blackwell 上的长上下文 Kimi 类工作负载新增的 MLA 内核工作。其次，TokenSpeed 在 `tokenspeed-scheduler/` 中提供了一个原生 C++ 调度器，将请求流转和缓存操作建模为显式的状态机，而 Python 仍作为运行时与集成层。

Dynamo 现已通过 `python -m dynamo.tokenspeed` 提供将 TokenSpeed 作为 Dynamo 后端运行的 Day 0 支持。Dynamo 前端仍然作为面向用户的 OpenAI 兼容 API 入口，负责请求路由、流式响应以及取消逻辑。

请参阅 [Kimi K2.5 TokenSpeed recipe](https://github.com/ai-dynamo/dynamo/tree/main/recipes/kimi-k2.5/tokenspeed/agg/nvidia)，获取当前的 Dynamo 启动配方。

进展非常迅速。上游 TokenSpeed 标注了正在进行中的工作，包括模型覆盖、P/D、EPLB、KV store、Mamba cache、VLM、metrics、Hopper 优化以及相关运行时特性。
