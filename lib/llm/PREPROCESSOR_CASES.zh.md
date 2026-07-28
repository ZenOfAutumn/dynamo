# Frontend 预处理器 —— 单 parser 决策点

OpenAI 预处理器根据已配置的 tool-call / reasoning parser 在每次请求中所做决策的参考分类。下面每个 case 命名了预处理器所做的一个决策点；最后的**逐 parser 真值表**记录了每个 parser 的预期值。

它是 [`lib/parsers/PARSER_CASES.md`](../parsers/PARSER_CASES.md) 在预处理层的对应物。Parser 的单元测试覆盖输入形态下的输出正确性（CASE.\*）；预处理器的单元测试覆盖*在 (parser, request) 组合下，是否触发了正确的配置开关*（PRE.\*）。大多数预处理器 bug 都是“忘了把 parser X 加进真值表” —— parser 上线了，预处理器却不知道它，新路径以错误默认值悄悄运行，parser 收到格式错误的输入。

新增 parser 时，请逐行走完下面真值表中的所有 PRE.\*，并显式决定取值（包括 `N/A`）。不要悄悄继承默认值 —— 把这一行写出来。

---

## PRE.1 —— `skip_special_tokens` 默认值

**函数：** `OpenAIPreprocessor::parser_requires_special_tokens`

vLLM 在将 token 解码为文本时默认 `skip_special_tokens=true`。对于那些语法标记（`<|channel|>`、`<|message|>`、`<|tool_calls_section_begin|>`、`<|think|>` 等）属于*单个特殊 token*的 parser 来说，这会把标记从解码后的文本中剥离掉，结果 parser 看到的是没有任何结构可匹配的纯文本。

预处理器会针对此类 parser 把默认值翻转为 `false`。调用方仍可通过 `output_options.skip_special_tokens` 显式覆盖。

**取值错误时的静默后果：** 即便模型正确地输出了 `reasoning_content` / `tool_calls`，结果也会为空。没有错误路径。这正是在 harmony/gpt_oss 白名单落地之前打挂 gpt-oss 的 `test_reasoning_effort` 的故障模式。

## PRE.2 —— 按请求关闭 reasoning

**函数：** `OpenAIPreprocessor::is_reasoning_disabled_by_request`

某些 parser 应根据请求中的 `chat_template_args` 被**关闭**。具体的参数名与取值因模型家族而异：

- `kimi_k25` —— `thinking: false`
- `nemotron_nano` / `nemotron3` / `nemotron_v3` ——
  `enable_thinking: false` 或 `force_nonempty_content: true`
- `deepseek_r1` / `deepseek_v4` —— `thinking: false` 或
  `thinking_mode: "chat"`（与 V4 formatter 的 `resolve_thinking_mode` 约定保持一致；让 parser 与 prompt 同步）
- `gemma4` —— `enable_thinking: false`

闸门触发时，不会运行任何 reasoning parser；模型输出会被作为普通 content 处理。

**取值错误时的静默后果：** `reasoning_content` 被错误标注（reasoning 实际上已经关闭却被当作 reasoning 输出，反之亦然）。每个 parser 的正确性 —— 取决于各自模型的 chat template 行为。

## PRE.3 —— Force-reasoning 与 tool-continuation 的相互作用

**函数：** `OpenAIPreprocessor::postprocessor_parsing_stream` 中的内联闸门

当 chat template 在 prompt 中注入 reasoning 起始 token（例如 `<think>`）时，预处理器会设置 `prompt_injected_reasoning=true`，使 parser 立即以 reasoning 模式启动。结合 `last_is_tool` 闸门：

- `last_is_tool == true` → force-reasoning **关闭**（当前行为）。
  原理：tool-continuation 轮次会直接基于工具结果产出最终面向用户的回答；force-reasoning 会把这个最终回答错标为 `reasoning_content`。这与观察到的 SGLang 上 Kimi K2.5 的行为一致。
- `last_is_tool == false` → 仍然遵循 force-reasoning。

**冲突点：** DSv4 不认同此闸门 —— V4 formatter 在合并的工具结果之后会向 prompt 中*植入* `<think>`，因此即便 `last_is_tool` 为真，DSv4 也需要 force-reasoning **开启**。该问题在 [#8901](https://github.com/ai-dynamo/dynamo/pull/8901) 中跟踪。最终方案大概率需要按 parser 处理这个闸门（仿照 PRE.2 中按 parser 匹配的形态），而非全局行为。

**取值错误时的静默后果：** `</think>` 字面量泄漏到 `content`（DSv4），或最终回答被错标为 reasoning（Kimi K2.5）。

## PRE.4 —— `tool_choice` 强制 guided JSON

**函数：** `OpenAIPreprocessor::postprocessor_parsing_stream` 中的内联闸门

`tool_choice = required | named` 会强制后端进入 guided decoding，将输出约束为不带 reasoning 包裹的纯 JSON。预处理器在此情形下会关闭 reasoning parser。

**取值错误时的静默后果：** 那些会无条件注入 `<think>` 的 parser（如 `minimax_append_think`）会污染送进 jail 的 tool-call JSON。

## PRE.5 —— `ignore_eos` / EOS token id

**函数：** `stop_conditions.apply_ignore_eos`

当 `ignore_eos = true` 时，预处理器不会传递模型的 EOS token id；否则会传递。这是通用规则 —— 与 parser 无关 —— 列在这里只是提醒：该决策由预处理器负责。

## PRE.6 —— 关闭 Nemotron 时去除前导 `<think>`

**函数：** `OpenAIPreprocessor::strip_leading_reasoning_start_from_stream`

当 PRE.2 关闭某个 Nemotron force-reasoning parser 时，Dynamo 不应再运行该 reasoning parser。然而，兼容 vLLM 的 Nemotron 模板在诸如 `force_nonempty_content=true` 等关闭 thinking 模式下仍可能输出前导的 `<think>`。预处理器只剥离这一前导标记，缓冲跨 chunk 切割的前缀（如 `"<thi"` + `"nk>answer"`），按流式 choice 跟踪状态，并将剩余字节作为正常 `content` 输出。

该路径专属于 `nemotron_nano` / `nemotron3` / `nemotron_v3`；当 `tool_choice=required|named` 已强制 guided JSON 时跳过此路径。

**取值错误时的静默后果：** `content` 中泄漏前导 `<think>`，或当前缀被分散在不同流 chunk / choice 间时丢失内容。

---

## 逐 parser 真值表

新增 parser 时请逐行走查此表。**`?`** 表示尚未验证 —— 如果你知道答案，请补全。

| Parser | PRE.1 需要特殊 token | PRE.2 reasoning 闸门 | PRE.6 关闭 Nemotron 时剥离前导 `<think>` | PRE.3 tool-continuation 时是否覆盖 force-reasoning | 备注 |
|---|---|---|---|---|---|
| `harmony` (tool) / `gpt_oss` (reasoning) | **YES** | — | — | — | Channel: `<\|channel\|>analysis<\|message\|>...<\|end\|>`。gpt-oss-20B/120B。 |
| `gemma4` (tool + reasoning) | **YES** | `enable_thinking=false` | — | — | `<\|think\|>...<\|/think\|>`。 |
| `kimi_k25` (reasoning) | ? —— 标记为 `<\|tool_calls_section_*\|>`，可能为 YES | `thinking=false` | — | last_is_tool 时 OFF（当前为全局） | K2/K2.5/K2.6 中为特殊 token 标记。 |
| `deepseek_v3` (tool) | ? —— Unicode 标记（`<｜tool_calls_section_begin｜>`）；可能为 YES | — | — | — | DSv3 语法。 |
| `deepseek_v3_2` / `deepseek_v4` (DSML) | ? —— DSML 标记（`<｜DSML｜tool_calls>`）；可能为 YES | `thinking=false` / `thinking_mode=chat` | — | 即便 last_is_tool 也**需要 ON**（V4 formatter 植入 `<think>`）；见 #8901 | DSv3.2 / DSv4 语法。 |
| `deepseek_r1` (reasoning) | NO（使用普通 `<think>`） | `thinking=false` | — | — | DeepSeek-R1。 |
| `nemotron_deci` (tool) / `nemotron_nano` / `nemotron3` / `nemotron_v3` (reasoning) | ? | `enable_thinking=false` / `force_nonempty_content=true`（仅 nano/n3/v3） | PRE.2 关闭 reasoning 时为 YES | — | Nemotron 系列；`nemotron_v3` 是兼容 vLLM 的别名。 |
| `llama3_json` (tool) | ? —— `<\|python_tag\|>` 是特殊 token，可能为 YES | — | — | — | Llama 3.x。 |
| `hermes` (tool) | NO | — | — | — | 普通 XML `<tool_call>...</tool_call>`。 |
| `qwen3_coder` (tool) | NO | — | — | — | 普通 XML `<tool_call><function=...>`。 |
| `pythonic` (tool) | NO | — | — | — | Python list 字面量。 |
| `mistral` (tool) | NO | — | — | — | `[TOOL_CALLS]` 纯文本。 |
| `phi4` (tool) | NO | — | — | — | `functools[...]` 纯文本。 |
| `minimax_m2` (tool) / `minimax_append_think` (reasoning) | NO | — | — | `tool_choice=required/named` 时 OFF（通用，PRE.4） | XML 标记，纯文本。 |
| `glm47` (tool) | NO | — | — | — | 普通 XML。 |
| `jamba` (tool) | NO | — | — | — | `<tool_calls>` 纯文本包裹。 |
| `qwen` (reasoning，basic `<think>`) | NO | — | — | — | 普通 `<think>...</think>`。 |

---

## 新增 parser —— checklist

1. PRE.1：parser 的语法标记是否被模型 tokenizer 视为特殊 token？做一个简单的 sanity check：用模型 tokenizer 编码该标记字符串，如果它返回特殊 token 范围内的单个 token id，就**把该 parser 加入 `parser_requires_special_tokens`**。
2. PRE.2：parser 是否需要根据 `chat_template_args` 静默关闭？查看模型的 chat template：它是否通过 `enable_thinking` / `thinking` / `thinking_mode` 之类的 flag 来限制 parser 标记的输出？如是，请在 `is_reasoning_disabled_by_request` 中添加该 case。
3. PRE.6：如果 PRE.2 关闭了 parser，后端是否仍可能输出一个应当被作为普通 content 的前导 reasoning 标记？如是，请添加显式的剥离/透传 case 与流式测试覆盖。
4. PRE.3：当上一轮是 tool 调用时，模型是会重新进入 reasoning（DSv4），还是直接给出回答（Kimi K2.5）？请在此表中显式注明；当前代码使用全局闸门，可能需要改为按 parser 处理（见 #8901）。
5. PRE.4：确认 `tool_choice = required/named` 与 parser 行为不冲突。在该情形下通用默认是关闭 reasoning 解析。
6. 在上面真值表中新增一行并填入显式取值。`N/A` 可以接受，但必须显式标明，不可省略。
7. 在 `lib/llm/src/preprocessor.rs` 的 `#[cfg(test)] mod` 中添加单元测试，断言 `parser_requires_special_tokens(...)` 对新 parser 返回预期值。表驱动，本文件中每个 parser 一行。
