---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Integration README
---

{/* 该外部集成的 2-3 句概述 */}

## 版本兼容性

| Dynamo | `<Integration>` | 备注 |
|--------|---------------|-------|
| 0.9.x | 1.2.x | 推荐 |
| 0.8.x | 1.1.x | |

## 后端支持

| 后端 | 状态 | 备注 |
|---------|--------|-------|
| vLLM | ✅ | |
| SGLang | 🚧 | |
| TensorRT-LLM | ❌ | |

## 快速开始

```bash
# Add installation and usage from existing integration docs
# Example pattern (LMCache):
# python -m dynamo.vllm --model <model> --kv-transfer-config '{"kv_connector":"LMCacheConnectorV1","kv_role":"kv_both"}'
```

## 配置

| 参数 | 默认值 | 描述 |
|-----------|---------|-------------|
| {/* param */} | {/* default */} | {/* description */} |

## 指南

| 文档 | 路径 | 描述 |
|----------|------|-------------|
| `<Integration> Setup` | `<integration>_setup.md` | 安装与配置 |
| `<Integration> with vLLM` | `<integration>_vllm.md` | vLLM 专项用法 |

{/* 将表格行转为 Markdown 链接 */}

## 外部资源

- [`<Integration>` 文档](https://...)
- [`<Integration>` GitHub](https://github.com/...)
