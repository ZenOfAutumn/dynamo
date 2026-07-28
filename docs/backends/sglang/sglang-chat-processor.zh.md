---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: SGLang Chat Processor
subtitle: SGLang-native preprocessing and postprocessing for chat completions
---

SGLang chat processor 让 Dynamo 前端能够使用 SGLang 原生的预处理与后处理。它直接使用 SGLang 的分词器（tokenizer）、chat 模板、tool call parser 和 reasoning parser——对 `v1/chat/completions` 请求绕过默认的 Rust 预处理器。

## 何时使用

当 Dynamo 内置的 Rust 预处理器尚未支持你需要的某个 tool call parser 或 reasoning parser 时，可以使用 `--dyn-chat-processor sglang`。SGLang processor 会委托给 SGLang 的 Python 实现，因此 SGLang 支持的任何 parser 都可以立刻使用。

常见场景：

- Rust `tool_calling` 库尚未支持的 **tool call 格式**
- 尚未原生支持的 **reasoning parser**
- Rust 预处理器无法正确处理的 **chat 模板**

如果 Rust 预处理器缺少你需要的 parser，可以考虑[提交 issue 或 PR](https://github.com/ai-dynamo/dynamo/issues) 来添加原生支持——原生 parser 完全没有 Python GIL 开销。

## 快速开始

```bash
# Frontend with SGLang processor, tool calling, and reasoning
python -m dynamo.frontend \
  --router-mode kv \
  --dyn-chat-processor sglang \
  --tool-call-parser hermes \
  --reasoning-parser qwen3

# Workers (unchanged)
CUDA_VISIBLE_DEVICES=0 python -m dynamo.sglang \
  --model-path Qwen/Qwen3-14B-FP8 \
  --served-model-name Qwen/Qwen3-14B-FP8 \
  --tp 1 --trust-remote-code \
  --kv-events-config '{"publisher":"zmq","topic":"kv-events","endpoint":"tcp://*:5557"}'
```

## 前端参数

使用 `--dyn-chat-processor sglang` 时，以下参数传给**前端**（不是 worker）：

| 参数 | 默认值 | 说明 |
|----------|---------|-------------|
| `--dyn-chat-processor sglang` | (none) | 启用 SGLang chat processor |
| `--tool-call-parser` | `None` | tool call parser 名称（任何 SGLang 支持的 parser） |
| `--reasoning-parser` | `None` | reasoning parser 名称（任何 SGLang 支持的 parser） |

### 环境变量

| 变量 | 默认值 | 说明 |
|----------|---------|-------------|
| `DYN_SGLANG_STREAM_INTERVAL` | `20` | 触发反分词（detokenize）前累计的 token 数。值越大吞吐越高。第一个 chunk 始终立即发出（interval=1），以最小化首 token 时延。 |

## Tool Calling

processor 支持所有 SGLang 的 tool call 格式。在前端传入 `--tool-call-parser`：

```bash
python -m dynamo.frontend \
  --dyn-chat-processor sglang \
  --tool-call-parser hermes
```

任何 SGLang 支持的 parser 都可使用。完整的 tool call parser 列表参见 [SGLang 文档](https://docs.sglang.ai/)。

### 示例：Tool Call 请求

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-14B-FP8",
    "messages": [{"role": "user", "content": "What is the weather in Paris?"}],
    "tools": [{
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get weather for a city",
        "parameters": {
          "type": "object",
          "properties": {"city": {"type": "string"}},
          "required": ["city"]
        }
      }
    }],
    "tool_choice": "auto"
  }'
```

响应：

```json
{
  "choices": [{
    "message": {
      "role": "assistant",
      "tool_calls": [{
        "id": "call_8cd24396f3671048",
        "type": "function",
        "function": {
          "name": "get_weather",
          "arguments": "{\"city\": \"Paris\"}"
        }
      }],
      "reasoning_content": "The user wants weather info for Paris..."
    },
    "finish_reason": "tool_calls"
  }]
}
```

## Reasoning 解析

对于会输出思维链推理（chain-of-thought）的模型（例如 Qwen3、DeepSeek-R1），传入 `--reasoning-parser`：

```bash
python -m dynamo.frontend \
  --dyn-chat-processor sglang \
  --reasoning-parser qwen3
```

parser 会把 think 标签内的内容放进 `reasoning_content` 字段，把常规内容放进 `content` 字段。

## 从 `--use-sglang-tokenizer` 迁移

**worker** 上的 `--use-sglang-tokenizer` 已弃用。请改为在**前端**使用 `--dyn-chat-processor sglang`：

```diff
  # Before (deprecated)
- python -m dynamo.sglang --model-path <model> --use-sglang-tokenizer
- python -m dynamo.frontend

  # After
  python -m dynamo.sglang --model-path <model>
+ python -m dynamo.frontend --dyn-chat-processor sglang
```

主要差异：

| | `--use-sglang-tokenizer` | `--dyn-chat-processor sglang` |
|---|---|---|
| 位置 | worker 选项 | 前端选项 |
| KV 路由 | 不支持 | 支持 |
| Tool calling | 不支持 | 支持 |
| Reasoning | 不支持 | 支持 |
| Endpoints | 仅 `v1/chat/completions` | 仅 `v1/chat/completions` |

## 另见

- **[Tool Calling](../../agents/tool-calling.md)**：通用的 tool calling 指南
- **[参考指南](sglang-reference-guide.md)**：完整的 SGLang 后端参考
- **[Agentic Workloads](agents.md)**：面向 agent 的优先级调度与缓存固定（cache pinning）
