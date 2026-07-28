---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 推理（Reasoning）
subtitle: 为输出思考内容的模型配置推理解析器
---

某些模型会将推理或思考内容与最终回复分开输出。Dynamo 可通过在后端 worker 上配置 `--dyn-reasoning-parser`，将这部分输出拆分为 `reasoning_content` 和正常的 assistant 内容。

> [!TIP]
> 本页介绍 Dynamo 原生默认路径下的解析器名称。如需对比所有预处理选项（包括 vLLM/SGLang chat-processor 切换与 tokenizer 委托）以及路由兼容性，请参阅 [Chat Processor Options](chat-processor-options.md)。

## 前置条件

要启用推理解析，请按如下方式启动后端 worker：

- `--dyn-reasoning-parser`：从下面支持的列表中选择推理解析器

```bash
# <backend> can be sglang, trtllm, vllm, etc. based on your installation
python -m dynamo.<backend> --help
```

> [!TIP]
> 部分模型同时需要推理解析器和工具调用解析器。支持的工具调用解析器名称请参阅 [Tool Calling](tool-calling.md)。

## 支持的推理解析器

下表列出了 Dynamo 注册表中当前支持的推理解析器。**Upstream name** 列展示当 vLLM 或 SGLang 解析器名称与 Dynamo 不同时的对应关系——这与使用 `--dyn-chat-processor vllm` 或 `sglang` 时相关（见 [Chat Processor Options](chat-processor-options.md)）。upstream 一栏为空表示同名在所有地方都通用。`Dynamo-only` 表示该格式没有上游解析器。

标记为 **force-reasoning** 的解析器会从第一个 token 起就输出推理内容，无需显式开标签（如 `<think>` 等）。其余解析器要求模型输出中存在开标签。

| 解析器名称 | 模型 | 上游名称 | force-reasoning | 备注 |
|---|---|---|---|---|
| `basic` | 通用 CoT 模型 | Dynamo-only | No | 普通 `<think>...</think>` |
| `deepseek_r1` | DeepSeek R1, DeepSeek V3.1, DeepSeek V3.2 | | Yes | 对 V3.1/V3.2 须显式传入（无别名） |
| `deepseek_v4` | DeepSeek V4 Pro / Flash | vLLM: `deepseek_v4`；SGLang: `deepseek-v4` | No | `<think>...</think>`。别名：`deepseek-v4`、`deepseekv4` |
| `gemma4` | Google Gemma 4（thinking 模型） | vLLM: `gemma4` | No | `<\|channel>thought\n...<channel\|>`，会去除 `thought\n` 角色标签。别名：`gemma-4` |
| `glm45` | GLM-4.5、GLM-4.7 | Dynamo-only | No | `nemotron_deci` 的别名。`<think>...</think>` |
| `gpt_oss` | gpt-oss-20b / -120b | Dynamo-only | No | Harmony channel 推理格式 |
| `granite` | Granite 3.x | | No | `Here's my thought process:` / `Here's my response:` |
| `kimi` | Kimi K2 Instruct / Thinking | Dynamo-only | No | `◁think▷...◁/think▷` |
| `kimi_k25` | Kimi K2.5 | Dynamo-only | Yes | 带 force-reasoning 的 `<think>...</think>` |
| `minimax_append_think` | MiniMax M2 / M2.1 | Dynamo-only | No | 隐式前置开标签 `<think>` |
| `mistral` | Magistral | | Yes | `[THINK]...[/THINK]` |
| `nemotron3` | Nemotron-3 / Mini | vLLM: `nemotron_v3` | Yes | `deepseek_r1` 的别名。也接受 `nemotron_v3` |
| `nemotron_deci` | Nemotron-Super / -Ultra / -Deci, Llama-Nemotron | Dynamo-only | No | `<think>...</think>` |
| `nemotron_nano` | Nemotron-Nano | Dynamo-only | Yes | `deepseek_r1` 的别名 |
| `qwen3` | QwQ-32B、Qwen3-Think、Qwen3-Coder、Qwen3.5 | | No | `<think>...</think>` |
| `step3` | Step-3 / Step-3-Reasoning | Dynamo-only | Yes | `<think>...</think>` |

## 常见解析器组合

部分模型需要同时配置两种解析器。常见组合包括：

- `openai/gpt-oss-*`：`--dyn-tool-call-parser harmony --dyn-reasoning-parser gpt_oss`
- `deepseek-ai/DeepSeek-V4-*`：`--dyn-tool-call-parser deepseek_v4 --dyn-reasoning-parser deepseek_v4`
- `zai-org/GLM-4.7`：`--dyn-tool-call-parser glm47 --dyn-reasoning-parser glm45`
- `moonshotai/Kimi-K2.5*`：`--dyn-tool-call-parser kimi_k2 --dyn-reasoning-parser kimi_k25`
- `google/gemma-4-*` thinking 模型：`--dyn-tool-call-parser gemma4 --dyn-reasoning-parser gemma4 --custom-jinja-template examples/chat_templates/gemma4_tool.jinja`
- `Qwen/Qwen3.5*`：`--dyn-tool-call-parser qwen3_coder --dyn-reasoning-parser qwen3`
- MiniMax M2.1 风格输出：`--dyn-tool-call-parser minimax_m2 --dyn-reasoning-parser minimax_append_think`

## 与工具调用的相互作用

推理解析在工具调用解析之前进行。如果模型同时输出推理内容和工具调用，请同时配置两种解析器，这样 Dynamo 就能先分离推理文本，再从剩余的 assistant 输出中解析工具调用。
