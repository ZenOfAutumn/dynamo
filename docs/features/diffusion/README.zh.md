---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 扩散模型
subtitle: 在 Dynamo 中部署用于文生图、文生视频等任务的扩散模型
---

## 概述

Dynamo 支持跨多个后端为扩散模型提供服务，可基于文本提示生成图像和视频。这些后端通过与 LLM 推理相同的 Dynamo 流水线基础设施暴露扩散能力，包括前端路由、扩缩容以及可观测性。

## 支持矩阵

| 模态 | vLLM-Omni | SGLang | TRT-LLM |
|----------|-----------|--------|---------|
| 文生文 | ✅ | ✅ | ❌ |
| 文生图 | ✅ | ✅ | ✅ |
| 文生视频 | ✅ | ✅ | ✅ |
| 图生视频 | ✅ | ❌ | ❌ |

**状态：** ✅ 支持 | ❌ 不支持

## 后端文档

各后端的部署指南、配置与示例：

- **[vLLM-Omni](../../backends/vllm/vllm-omni.md)**
- **[SGLang Diffusion](../../backends/sglang/sglang-diffusion.md)**
- **[TRT-LLM Diffusion](../../backends/trtllm/trtllm-diffusion.md)**
- **[FastVideo（自定义 worker）](fastvideo.md)**
