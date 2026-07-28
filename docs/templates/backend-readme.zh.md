---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Backend README
---

{/* 该后端集成的 2-3 句概述 */}

## 功能矩阵

{/* 从已有后端文档中复制实际的功能矩阵 */}
{/* 示例样式（来自 vLLM README）： */}

| 功能 | 状态 | 备注 |
|---------|--------|-------|
| 分离式服务 | ✅ | |
| KV 感知路由 | ✅ | |
| SLA 驱动的 Planner | ✅ | |
| 多模态 | ✅ | 视觉模型 |
| LoRA | 🚧 | 实验性 |

## 快速开始

### 前置条件

- {/* 列出前置条件 */}

### 用法

```bash
# Add minimal usage example from existing backend docs
# Example pattern (vLLM):
# python -m dynamo.vllm --model <model-name>
# Example pattern (SGLang):
# python -m dynamo.sglang --model <model-name>
```

### Kubernetes

```yaml
# Add DGDR example - use apiVersion: nvidia.com/v1beta1
# See recipes/ folder for production examples
```

## 配置

| 参数 | 默认值 | 描述 |
|-----------|---------|-------------|
| {/* param */} | {/* default */} | {/* description */} |

{/* 示例：填写好的 vLLM 配置看起来像：

| 参数 | 默认值 | 描述 |
|-----------|---------|-------------|
| `--model` | required | 模型路径或 HuggingFace ID |
| `--tensor-parallel-size` | `1` | 用于张量并行的 GPU 数 |
| `--max-model-len` | auto | 最大序列长度 | */}

## 后续步骤

| 文档 | 路径 | 描述 |
|----------|------|-------------|
| `<Backend> Guide` | `<backend>_guide.md` | 进阶配置 |
| 后端对比 | `../README.md` | 后端对比 |

{/* 将表格行转为 Markdown 链接 */}
