# dynamo-parsers

用于从 LLM 原始输出中解析 **tool call** 与 **reasoning content** 的 Rust crate。具备线格式（wire-format）感知、流式优先、模型家族感知能力。

它是 Dynamo chat-completions 流水线中模型之后的部分：给定来自 vLLM 或 SGLang 的 token 流，提取出结构化的 `Vec<ToolCall>` 与 `reasoning_content` 返还给客户端。模型之前的部分（prompt 格式化）位于 `lib/llm/src/preprocessor/prompt/`。

## crate 内容

两个顶层模块，各自有独立的 parser 注册表：

```
lib/parsers/
├── src/
│   ├── tool_calling/        ← tool-call extraction (18 registered parsers)
│   │   ├── parsers.rs         — registry + dispatch (detect_and_parse_tool_call)
│   │   ├── config.rs          — per-parser ToolCallConfig
│   │   ├── response.rs        — ToolCallResponse shape (wire type)
│   │   ├── dsml/              — DeepSeek V3.2 / V4 DSML grammar
│   │   ├── gemma4/            — Google Gemma 4 custom non-JSON grammar (`<|"|>`-delimited strings)
│   │   ├── xml/               — hermes, glm47, kimi_k2, minimax_m2, qwen3_coder
│   │   ├── json/              — deepseek_v3, deepseek_v3_1, nemotron_deci/nano, jamba, mistral, phi4, llama3_json
│   │   ├── harmony/           — OpenAI gpt-oss (Harmony token stream, uses openai_harmony crate)
│   │   └── pythonic/          — Python function-call syntax (some Llama variants)
│   └── reasoning/           ← reasoning-content extraction (15 registered parsers)
│       ├── mod.rs             — registry + dispatch
│       ├── base_parser.rs     — BasicReasoningParser (<think> ... </think>)
│       ├── gemma4_parser.rs   — Gemma 4 (`<|channel>thought\n...<channel|>`)
│       ├── gpt_oss_parser.rs  — Harmony channel parsing
│       ├── granite_parser.rs  — Granite-style
│       └── minimax_append_think_parser.rs  — MiniMax inline-reasoning
```

## 一次请求在 crate 中的流向

```
  token stream from engine
            │
            ▼
  ┌─────────────────────────────────┐
  │ reasoning parser                │  — registered by name via
  │   (basic / gpt_oss / ...)       │    reasoning::mod.rs get_reasoning_parser_map()
  │                                 │    returns: (reasoning_content, non_reasoning_tail)
  └─────────────────────────────────┘
            │
            ▼ (non-reasoning tail)
  ┌─────────────────────────────────┐
  │ tool-call parser                │  — registered by name via
  │   dispatched on parser name     │    tool_calling::parsers::get_tool_parser_map()
  │   which picks a ParserConfig:   │
  │     - Dsml(DsmlParserConfig)    │  → try_tool_call_parse_dsml
  │     - Json(JsonParserConfig)    │  → try_tool_call_parse_json
  │     - Xml(XmlParserConfig)      │  → try_tool_call_parse_xml
  │     - KimiK2(KimiK2ParserConfig)│  → try_tool_call_parse_kimi_k2
  │     - Pythonic / Harmony        │
  └─────────────────────────────────┘
            │
            ▼
  Vec<ToolCallResponse> + normal_text
```

`tool_calling/parsers.rs` 中的主要公共入口：

- `detect_and_parse_tool_call(input, parser_name, schema) -> (calls, normal_text)`
- `try_tool_call_parse(input, config) -> (calls, normal_text)`（更底层，绕过注册表）
- `detect_tool_call_start(chunk, parser_name)` —— 流式：“当前 chunk 是否开启了一个 tool-call 块？”
- `find_tool_call_end_position(chunk, parser_name)` —— 流式：“块在当前 chunk 中的何处结束？”

## Parser 家族速查表

新增模型时，对应的 parser 家族通常是以下之一：

| 家族 | 语法 | 共享引擎 | 示例 |
| -- | -- | -- | -- |
| **DSML** | `<｜DSML｜tool_calls>...`，参数带类型 `string="true|false"` | `dsml/parser.rs` | DeepSeek V3.2、V4 |
| **XML** | `<tool_call>...</tool_call>`，嵌套 `<parameter>` 或 `<function>` | `xml/parser.rs`（通用）或为变体单独提供文件 | hermes、qwen3_coder、minimax_m2、glm47（独立）、kimi_k2（独立，特殊 token XML） |
| **JSON** | 起始哨兵 + 由 `{name, arguments}` 组成的纯 JSON 数组 | `json/base_json_parser.rs` | deepseek_v3、deepseek_v3_1、nemotron_deci/nano |
| **Harmony** | OpenAI Harmony token 流，含 `<\|channel\|>`、`<\|message\|>`、`<\|call\|>` | `harmony/harmony_parser.rs`（包装外部 `openai_harmony` crate） | gpt-oss-20B / 120B |
| **Pythonic** | `[func_name(arg=value, ...)]` 形式的 Python 函数调用语法 | `pythonic/pythonic_parser.rs` | 部分 Llama 变体 |
| **Gemma 4** | 自定义：`<\|tool_call>call:name{key:<\|"\|>val<\|"\|>}<tool_call\|>`，裸 key、自定义字符串分隔符 | `gemma4/parser.rs`（递归下降到 `serde_json::Value`） | Google Gemma 4 thinking 模型 |

Reasoning parser：

| 家族 | 语法 | 共享引擎 | 示例 |
| -- | -- | -- | -- |
| **Basic（think 标签）** | `<think>...</think>` | `reasoning/base_parser.rs`（BasicReasoningParser） | Qwen3、Nemotron、Kimi K2.5、DeepSeek R1 / V4、GLM-4.5+ |
| **Append-think** | `<think>...</think>` 作为内联文本保留，第一个 chunk 带 `<think>` 前缀 | `reasoning/minimax_append_think_parser.rs` | MiniMax M2 |
| **Harmony channel** | 隐藏的 `analysis` channel | `reasoning/gpt_oss_parser.rs`（包装外部 `openai_harmony`） | gpt-oss-20B / 120B |
| **Granite** | 自定义起止 token | `reasoning/granite_parser.rs` | IBM Granite |
| **Gemma 4 channel** | `<\|channel>thought\n...<channel\|>`，剥离 role-label 前缀 | `reasoning/gemma4_parser.rs` | Google Gemma 4 thinking 模型 |

## 新增 parser

1. **从上面的速查表中选择家族。** 如果某个已有的、配置驱动的家族能匹配，就在 `tool_calling/config.rs` 里添加 `ToolCallConfig::<your_model>()` 构造函数，并在 `tool_calling/parsers.rs` 中注册。完成 —— 你会自动继承所有共享 parser 与测试。

2. **如果语法确实是新颖的**，在 `tool_calling/` 下新增一个模块，并在 `config.rs` 中添加一个 `ParserConfig` variant。布局参照已有的 parser 模块。

3. **对于 reasoning**，除非语法确实有差异（append-think、Harmony channel），否则优先以别名形式复用 `BasicReasoningParser`。大多数新模型使用普通的 `<think>...</think>`，可以共享。

4. **编写测试。** 最低限度的测试集合在 [`PARSER_CASES.md`](./PARSER_CASES.md)（`PARSER.*` 分类）中。至少应包含：`PARSER.1` / `PARSER.2` / `PARSER.3` 验证正确性，`PARSER.5` 验证截断行为，`PARSER.8` / `PARSER.9` 验证带 reasoning 的流式行为，`PARSER.13` 验证文本与 tool call 交替。`N/A` 类别请在注释中显式标出，不要悄悄跳过。

## 相关文档

- [`PARSER_CASES.md`](./PARSER_CASES.md) —— 边界情况分类。每个 parser 都应被测试到的项、各家族中的 N/A，以及当前的通用空白点。
- `lib/llm/tests/data/` —— 按 (engine × model) 抓取的流式 fixture，供 `test_streaming_tool_parsers.rs` 使用。这是测试的回放侧。

## 与 Dynamo 其他部分的集成

- `lib/llm/src/preprocessor/prompt/` —— 模型之前的部分。负责写 prompt，这些 prompt（最终）会回流到这里被解析。
- `lib/llm/src/preprocessor.rs` —— 顶层请求/响应流水线。基于 `is_reasoning_disabled_by_request` 决定是否运行 reasoning parser，然后把剥离 reasoning 后的尾部送给 tool-call parser。
- `components/src/dynamo/frontend/` —— Python 前端，将解析后的输出以 OpenAI 兼容的 SSE chunk 形式输出给客户端。
