---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Tokenizer
---

Dynamo 前端为基于 BPE 的 `tokenizer.json` 模型支持多种 tokenizer 后端。`BPE` 是底层的分词算法，并非某个后端独有的特性：默认的 HuggingFace 路径与 `fastokens` 路径都可以服务这些模型。后端的选择决定了在请求被发送到推理引擎之前，由哪个实现来执行分词。

## Tokenizer 后端

#### `default` HuggingFace Tokenizers

默认后端使用 [HuggingFace `tokenizers`](https://github.com/huggingface/tokenizers) 库（Rust 实现）。
它支持 `tokenizer.json` 文件中的各项特性（normalizer、pre-tokenizer、post-processor、decoder、带特殊 token 标志的附加 token，以及 byte-fallback）。

#### `fastokens` 高性能编码器

`fastokens` 后端使用 [`fastokens`](https://github.com/Atero-ai/fastokens) crate，这是一个为受支持的 BPE `tokenizer.json` 模型在吞吐量上做了优化的专用编码器。
它是一个 _混合_ 后端：编码使用 `fastokens`，而解码回退到 HuggingFace，以便增量去分词、byte-fallback 与特殊 token 处理都能正常工作。

当分词成为可测量的瓶颈时（例如高并发、以预填充为主的工作负载），可使用此后端。

#### 兼容性说明：

- 适用于标准的 BPE `tokenizer.json` 文件（Qwen、LLaMA、GPT 系列、Mistral、DeepSeek 等）。
- 如果 `fastokens` 无法加载某个特定的 tokenizer 文件，前端会记录一条警告并透明地回退到 HuggingFace；请求绝不会被丢弃。
- 对 TikToken 格式的 tokenizer（`.model` / `.tiktoken` 文件）无效，这些文件始终使用 TikToken 后端。

## 配置

通过 CLI 标志或环境变量来设置后端。CLI 标志优先级更高。

| CLI 参数 | 环境变量 | 有效值 | 默认值 |
|---|---|---|---|
| `--tokenizer` | `DYN_TOKENIZER` | `default`, `fastokens` | `default` |

**示例：**

```bash
# CLI flag
python -m dynamo.frontend --tokenizer fastokens

# Environment variable
export DYN_TOKENIZER=fastokens
python -m dynamo.frontend
```

## Dynamo 前端行为

当设置 `DYN_TOKENIZER=fastokens` 时：

1. 前端会将该环境变量传递给 Rust 运行时。
2. 在为模型构建 tokenizer 时，`ModelDeploymentCard::tokenizer()` 尝试从同一个 `tokenizer.json` 文件加载 `fastokens::Tokenizer`。
3. 如果加载成功，将创建一个混合的 `FastTokenizer`，使用 `fastokens` 进行编码，使用 HuggingFace 进行解码。
4. 如果加载失败（特性不支持、文件缺失等），前端会记录警告并回退到标准的 HuggingFace 后端；无需运维人员介入。
