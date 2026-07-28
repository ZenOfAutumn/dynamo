---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Tool Calling
subtitle: 通过函数调用将 Dynamo 接入外部工具和服务
---

你可以通过函数调用（也称为 tool calling）将 Dynamo 接入外部工具与服务。通过提供一组可用函数，Dynamo 可以选择输出相应函数的参数，你再去执行这些函数，并使用相关的外部信息来扩充 prompt。

Tool calling（即 function calling）通过请求参数 `tool_choice` 与 `tools` 控制。

> [!TIP]
> 本页介绍的是 Dynamo 默认原生路径下的 parser 名称。关于所有预处理选项（包括 vLLM/SGLang chat-processor 切换与 tokenizer 委托）以及路由兼容性的对比，请参考 [Chat Processor Options](chat-processor-options.md)。

## 先决条件

启用该功能时，请在启动后端 worker 时设置以下 flag：

- `--dyn-tool-call-parser`：从下方支持列表中选择 tool call parser

```bash
# <backend> can be sglang, trtllm, vllm, etc. based on your installation
python -m dynamo.<backend> --help
```

> [!NOTE]
> 如果用户没有提供 tool call parser，Dynamo 会基于 &lt;TOOLCALL&gt; 与 &lt;|python_tag|&gt; 这两种 tool 标签尝试默认的 tool call 解析。

> [!TIP]
> 如果你模型默认的 chat template 不支持 tool calling，但模型本身支持，则可以通过 `python -m dynamo.<backend> --custom-jinja-template </path/to/template.jinja>` 为该 worker 指定自定义 chat template。

> [!TIP]
> 如果你的模型还会输出需要从普通输出中分离出来的 reasoning 内容，请参考 [Reasoning](reasoning.md) 了解 `--dyn-reasoning-parser` 支持的取值。

## 受支持的 Tool Call Parser

下表列出了 Dynamo 注册表中当前受支持的 tool call parser。**Upstream name** 列展示了 vLLM 或 SGLang 的 parser 名称与 Dynamo 不同的情况 —— 当使用 `--dyn-chat-processor vllm` 或 `sglang` 时相关（参见 [Chat Processor Options](chat-processor-options.md)）。该列为空表示同名在所有上下游均可使用。`Dynamo-only` 表示该格式没有上游 parser 实现。

| Parser 名称 | 模型 | Upstream 名称 | 备注 |
|---|---|---|---|
| `deepseek_v3` | DeepSeek V3、DeepSeek R1-0528+ | SGLang: `deepseekv3` | 特殊 Unicode 标记 |
| `deepseek_v3_1` | DeepSeek V3.1 | Dynamo-only | JSON 分隔符 |
| `deepseek_v3_2` | DeepSeek V3.2+ | Dynamo-only | DSML 标签（`<｜DSML｜function_calls>...`） |
| `deepseek_v4` | DeepSeek V4 Pro / Flash | vLLM: `deepseek_v4`；SGLang: `deepseekv4` | DSML 标签（`<｜DSML｜tool_calls>...`）。别名：`deepseek-v4`、`deepseekv4` |
| `default` | *(兜底)* | Dynamo-only | 空 JSON 配置（无起止 token）。生产环境请优先使用模型专属 parser。 |
| `gemma4` | Google Gemma 4（thinking 模型） | vLLM: `gemma4` | 自定义非 JSON 语法，使用 `<\|"\|>` 字符串分隔符与 `<\|tool_call>...<tool_call\|>` 标记。别名：`gemma-4`。请配合 `--dyn-reasoning-parser gemma4` 与 `--custom-jinja-template examples/chat_templates/gemma4_tool.jinja` 使用 |
| `glm47` | GLM-4.5、GLM-4.7 | Dynamo-only | XML `<arg_key>/<arg_value>` |
| `harmony` | gpt-oss-20b / -120b | Dynamo-only | Harmony channel 格式 |
| `hermes` | Qwen2.5-\*、QwQ-32B、Qwen3-Instruct、Qwen3-Think、NousHermes-2/3 | vLLM: `qwen2_5`；SGLang: `qwen25`（Qwen 模型） | `<tool_call>` JSON |
| `jamba` | Jamba 1.5 / 1.6 / 1.7 | Dynamo-only | `<tool_calls>` JSON |
| `kimi_k2` | Kimi K2 Instruct/Thinking、Kimi K2.5 | | 配合 `--dyn-reasoning-parser kimi` 或 `kimi_k25` 使用 |
| `llama3_json` | Llama 3 / 3.1 / 3.2 / 3.3 Instruct | | `<\|python_tag\|>` tool 语法 |
| `minimax_m2` | MiniMax M2 / M2.1 | vLLM: `minimax` | XML `<minimax:tool_call>` |
| `mistral` | Mistral / Mixtral / Mistral-Nemo、Magistral | | `[TOOL_CALLS]...[/TOOL_CALLS]` |
| `nemotron_deci` | Nemotron-Super / -Ultra / -Deci、Llama-Nemotron-Ultra / -Super | Dynamo-only | `<TOOLCALL>` JSON |
| `nemotron_nano` | Nemotron-Nano | Dynamo-only | `qwen3_coder` 的别名 |
| `phi4` | Phi-4、Phi-4-mini、Phi-4-mini-reasoning | vLLM: `phi4_mini_json` | `functools[...]` JSON |
| `pythonic` | Llama 4（Scout / Maverick） | | Python list 形式的 tool 语法 |
| `qwen3_coder` | Qwen3-Coder、Qwen3.5 | | XML `<tool_call><function=...>` |

> [!TIP]
> 对于 Kimi K2.5 thinking 模型，请同时使用 `--dyn-tool-call-parser kimi_k2` 与 [Reasoning](reasoning.md) 中的 `--dyn-reasoning-parser kimi_k25`，以便从同一个响应中同时正确解析 `<think>` 块与 tool call。

## 示例

### 启动 Dynamo Frontend 与 Backend

```bash
# launch backend worker
python -m dynamo.vllm --model openai/gpt-oss-20b --dyn-tool-call-parser harmony

# launch frontend worker
python -m dynamo.frontend
```

### Tool Calling 请求示例

- 示例 1
```python
from openai import OpenAI
import json

client = OpenAI(base_url="http://localhost:8081/v1", api_key="dummy")

def get_weather(location: str, unit: str):
    return f"Getting the weather for {location} in {unit}..."
tool_functions = {"get_weather": get_weather}

tools = [{
    "type": "function",
    "function": {
        "name": "get_weather",
        "description": "Get the current weather in a given location",
        "parameters": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "City and state, e.g., 'San Francisco, CA'"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
            },
            "required": ["location", "unit"]
        }
    }
}]

response = client.chat.completions.create(
    model="openai/gpt-oss-20b",
    messages=[{"role": "user", "content": "What's the weather like in San Francisco in Celsius?"}],
    tools=tools,
    tool_choice="auto",
    max_tokens=10000
)
print(f"{response}")
tool_call = response.choices[0].message.tool_calls[0].function
print(f"Function called: {tool_call.name}")
print(f"Arguments: {tool_call.arguments}")
print(f"Result: {tool_functions[tool_call.name](**json.loads(tool_call.arguments))}")
```

- 示例 2
```python

# Use tools defined in example 1

time_tool = {
    "type": "function",
    "function": {
        "name": "get_current_time_nyc",
        "description": "Get the current time in NYC.",
        "parameters": {}
    }
}


tools.append(time_tool)

messages = [
    {"role": "user", "content": "What's the current time in New York?"}
]


response = client.chat.completions.create(
    model="openai/gpt-oss-20b", #client.models.list().data[1].id,
    messages=messages,
    tools=tools,
    tool_choice="auto",
    max_tokens=100,
)
print(f"{response}")
tool_call = response.choices[0].message.tool_calls[0].function
print(f"Function called: {tool_call.name}")
print(f"Arguments: {tool_call.arguments}")
```

- 示例 3


```python

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_tourist_attractions",
            "description": "Get a list of top tourist attractions for a given city.",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "The name of the city to find attractions for.",
                    }
                },
                "required": ["city"],
            },
        },
    },
]

def get_messages():
    return [
        {
            "role": "user",
            "content": (
                "I'm planning a trip to Tokyo next week. what are some top tourist attractions in Tokyo? "
            ),
        },
    ]


messages = get_messages()

response = client.chat.completions.create(
    model="openai/gpt-oss-20b",
    messages=messages,
    tools=tools,
    tool_choice="auto",
    max_tokens=100,
)
print(f"{response}")
tool_call = response.choices[0].message.tool_calls[0].function
print(f"Function called: {tool_call.name}")
print(f"Arguments: {tool_call.arguments}")
```
