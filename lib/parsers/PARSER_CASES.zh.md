# Tool-Call Parser 边界案例

`src/tool_calling/` 下 **tool-call** 解析器单元测试的参考分类法。
相邻阶段的同类文档：

- **Reasoning 解析器**（`src/reasoning/`）：见 `REASONING_CASES.md`。
- **前端门控（Frontend gating）**（请求时的 `tool_choice` 等）：见
  `components/src/dynamo/frontend/tests/FRONTEND_CASES.md`。
- **流水线边界（Pipeline boundary）**（`finish_reason` 独立性等）：见
  `PIPELINE_CASES.md`。

分类按三个正交轴划分：

1. **阶段** —— 被测的模块（本文是 `PARSER.*`，`REASONING.*`、
   `FRONTEND.*`、`PIPELINE.*` 各自有文件）。
2. **模式** —— 调用接口：`batch`（整个模型输出作为单一字符串）
   或 `stream`（增量的 `delta_text` + `delta_token_ids`）。
   每种模式有自己的连续编号；batch 中的"单 tool call"为
   `PARSER.batch.1`，stream 中的为 `PARSER.stream.1`——
   编号在不同模式间不共享语义。
3. **格式范围** —— 比"所有解析器"更窄的适用范围：
   `fmt`、`xml`、`harmony`。

每个 `#[test]` 都带有一个或多个 `// PARSER.<tag>` 注解，
列出它覆盖的分类。`N/A` 在测试文件中显式标出，而非默默省略。

因为某个具体客户 ticket / PR / GH issue 而存在的测试，会在注释中
内联给出原始来源——例如 `// PARSER.batch.5 (PR #8208)`。
分类标签声明的是分类归属；圆括号内是审计线索。两个方向都可 grep：
`grep -r 'PARSER.batch.5'` 找到所有此类测试；
`grep -r '#8208'` 找到与该事件相关的所有测试（跨各层）。

`detect_tool_call_start_*` / `find_tool_call_end_position_*` 这种
白盒辅助单元测试以 `// helper` 标注；它们没有跨实现的对应物，
仅用于固定内部 Rust 函数行为，因此不带编号分类。

## 速查表

### Parser，batch 模式

通用行为契约。当把整段模型输出作为单一字符串送入时，适用于每一个
tool-call 解析器。

- **`PARSER.batch.1`** 单个 tool call —— 理想路径（一次完整、规范的调用）。
- **`PARSER.batch.2`** 多个 tool call —— 顺序或并行（一个响应中 2 个或更多）。
- **`PARSER.batch.3`** 没有 tool call（响应只有文本）。
- **`PARSER.batch.4`** 异常 / 部分 JSON args（截断、缺右花括号、语法非法）。恢复契约由实现决定；记录差异而非断言唯一真值。
- **`PARSER.batch.5`** 缺失结束 token 的恢复（在 `max_tokens` / EOS 致使闭合 fence 缺失时仍恢复调用）。
- **`PARSER.batch.6`** 空 args（`arguments={}` / 无参调用）。
- **`PARSER.batch.7`** 复杂 arg 类型（嵌套对象、数组、bool、数字、值中的 Unicode / 换行）。
- **`PARSER.batch.8`** tool call 与普通文本交错。
- **`PARSER.batch.9`** 空 content / 空 `tool_calls` 数组 / null 响应。
- **`PARSER.batch.10`** 重复 tool call（同名两次，可能 args 也相同）。

### Parser，stream 模式

来自流式引擎的逐 token 装配。逻辑案例与 batch 模式相同，
但通过 `parse_streaming_increment(delta)` 驱动。
该模式需要单独的测试 harness；编号与 batch 独立。

- **`PARSER.stream.1`** 单个 tool call 跨 N 个 chunk（任意 chunk 边界）。
- **`PARSER.stream.2`** 多个 tool call，每个跨多个 chunk。
- **`PARSER.stream.3`** 部分 token 切分（chunk 边界把语法 token 切在中间）。部分 token 匹配必须返回"继续缓冲"，不能作为普通文本刷出。
- **`PARSER.stream.4`** 流式终止 —— 末 chunk 携带 `finish_reason=tool_calls` / EOS；解析器把进行中的调用刷出。

### 与格式相关（范围比"所有解析器"更窄）

同样的 `PARSER.fmt|xml|harmony` 标签也会出现在 `src/reasoning/` 测试中——
当某个 reasoning 解析器消费对应格式时。
标签描述的是 *语法*，而非解析器阶段。

#### `PARSER.fmt.*` —— 格式变体

- **`PARSER.fmt.1`** 函数名约定 —— 允许的标识符字符（连字符、下划线、点）、前缀变体（`functions.NAME` vs 裸 `NAME`），以及对异常函数 ID 的拒绝。
- **`PARSER.fmt.2`** 空白 / 格式宽容 —— 语法 token 之内或之间的空白。
- **`PARSER.fmt.3`** Token / 线格式变体 —— 同语义的多种合法拼写。例：Kimi K2 的单数 `<|tool_call_section_*|>` vs 复数 `<|tool_calls_section_*|>` section token；Mistral pre-v11（`[TOOL_CALLS][{"name":...,"arguments":...}]` JSON 数组体） vs v11+（`[TOOL_CALLS]name{...args}` 名后接对象）；Llama 3 带或不带 `<|python_tag|>` 起始 fence；Hermes `qwen25` 注册表别名。解析器必须接受所有已配置的变体并拒绝未为当前 config 注册的变体。
- **`PARSER.fmt.4`** 空 section / 无内容 wrapper —— 起止 fence 之间什么都没有。
- **`PARSER.fmt.5`** 参数形态约定 —— 调用体内部 JSON 信封形态，与 `PARSER.fmt.1` 的函数名表面不同。三个子轴：原生 call-ID 保留（Kimi K2 的 `functions.NAME:N` 在 `ToolCall.id` 上原样保留）；JSON 字段顺序宽容（`{name, arguments}` vs `{arguments, name}`，含 `arguments` 自身包含名为 `"name"` 的键的情况）；参数键别名（`arguments` 与 `parameters` 可互换）。

#### `PARSER.xml.*` —— 仅 XML 族

（`hermes`、`glm47`、`qwen3_coder`、`minimax_m2`、`kimi_k2`。）

- **`PARSER.xml.1`** XML entity / HTML 反转义处理（`&lt;`、`&amp;`、`&quot;` 等）。
- **`PARSER.xml.2`** 基于 schema 的类型强制（按声明的参数 schema，把 string 转为 number/bool/array）。

#### `PARSER.harmony.*` —— 仅 Harmony（gpt-oss）

- **`PARSER.harmony.1`** Channel / recipient 解析（analysis / commentary / final 通道；`to=functions.X` 的 recipient）。
- **`PARSER.harmony.2`** 信封标签语法 —— `<|channel|>commentary to=functions.X <|constrain|>json<|message|>{...}<|call|>` 及其合法变体。

### 全局空白点（任意层级都没有的测试）

- 函数名中的 Unicode（非 ASCII tool 名、emoji）。
- args 中的数值溢出（极大整数 / float 超出 JSON spec 范围）。
- 空函数名（`"name": ""`）。
- 并发并行请求（解析过程的进程级竞争）。
- 引导式解码 ↔ tool-call 交互（受约束生成产出异常 args）。
- 极长输出（单次调用 ≥10 KB 的 tool-call JSON）。
- 流中错误注入 / 中断（解析中途 worker 被杀、网络断开）。
- schema 参数数量不匹配（模型相对于声明 schema 多发或少发参数）。
- regex 超时 / 灾难模式 guard、解析器异常隔离、长普通内容快路径。vLLM 对 `llama3_json` / `llama4_pythonic` / `pythonic` 有显式 `test_regex_timeout_handling`，对 Mistral 有 `test_extract_tool_calls_streaming_exception_returns_none`；Dynamo 依赖 Rust `regex` crate 的线性时间保证，但未对失败隔离路径进行固化。

### 已知生产空白点（解析器完全缺失）

- **Mistral v11+ 线格式**（`[TOOL_CALLS]name{...args}` 名后接对象）。Dynamo 的 `ToolCallConfig::mistral()` 与底层的 `base_json_parser.rs` 仅处理 pre-v11（`[TOOL_CALLS][{name, arguments}]` JSON 数组体）。v11 是当前 Mistral-Small / Mistral-Large 的生产路径；vLLM 在 `mistral_tool_parser` fixture 下对其做了大量测试。变体分类参见 `PARSER.fmt.3`。

---

## `PARSER.batch.1` —— 单个 tool call，理想路径

响应中只有一次完整、规范的调用。

- 适用于每一个 tool-call 解析器。
- 基线正确性检查。如果 `PARSER.batch.1` 不通过，下面的都没有意义。
- 对那些语法中带有模型自发 call ID 的解析器（例如 Kimi
  K2 的 `functions.NAME:N`），理想路径测试还必须断言原生 ID
  在 `ToolCall.id` 上原样保留——更宽泛的参数形态契约见
  `PARSER.fmt.5`。
  参考：vLLM Kimi K2 PR #32768。

## `PARSER.batch.2` —— 多个 tool call（顺序或并行）

一个响应中两次或更多调用，处于同一块或紧挨在一起。

- 适用于每一个 tool-call 解析器。
- 一些语法在一个块内发出并行调用（DSML、XML）；另一些
  发出顺序的顶层 sentinel（JSON 方言）。无论哪种，都应全部提取。

## `PARSER.batch.3` —— 没有 tool call

响应是纯文本，没有任何 tool-call 语法。

- 适用于每一个 tool-call 解析器。
- 必须返回空的 `Vec<ToolCall>`，并把输入作为 `normal_text`。
  零误报。

## `PARSER.batch.4` —— 异常 / 部分 JSON args

参数负载内部 JSON 截断、缺右花括号、语法非法。

- 适用于每一个 tool-call 解析器。对那些语法从不嵌入 JSON 的
  解析器（目前没有），显式标 `N/A`。
- 行为是 **由实现决定** 的：优雅回退到字符串（DSML 当前
  通过 `serde_json::from_str(...).unwrap_or_else(|_| String(...))`
  实现）、错误时丢弃、或显式报错都是合法选择。跨实现一致性测试
  应把差异记录在它们的 `KNOWN_DIVERGENCES` 注册表中，而非
  断言唯一真值。

## `PARSER.batch.5` —— 缺失结束 token 的恢复

模型响应在闭合 fence 到达之前就被截断
（Kimi K2 的 `<|tool_calls_section_end|>`、DeepSeek DSML 的
`</｜DSML｜tool_calls>` 等）——通常是因为引擎触发 `max_tokens`
或模型在生成中途发出 EOS。

- 适用于每一个具有成对起止 fence 的 tool-call 解析器。
- 对客户可见的 bug 类别：进行中调用被静默丢弃，看上去像是一次
  HTTP 200 成功响应，但既无 `tool_calls` 也无错误。
- 两种可接受的处理方式：(a) 即使没有外层闭合 fence 也能恢复已完成的
  invoke（Kimi K2 修复后即如此），或 (b) 返回显式错误。无论哪种，
  都用测试把行为锚定下来，以便未来变更是有意为之。

## `PARSER.batch.6` —— 空 args

带 `arguments={}` 或无参的 tool call。

- 适用于每一个 tool-call 解析器。
- 仍然必须返回该调用——空 args 是一次合法调用，而非缺失。

## `PARSER.batch.7` —— 复杂参数类型

嵌套对象、数组、布尔、数字、混合类型、Unicode 值，以及
参数值中的换行符。

- 适用于每一个 tool-call 解析器。
- 对那些携带类型提示的语法（DSML 的 `string="true|false"`），
  验证 JSON 往返。对没有提示的 XML 语法，类型强制部分位于
  `PARSER.xml.2`——这里只验证复杂值能完整通过，没有截断或
  转义 bug。

## `PARSER.batch.8` —— 普通文本与 tool call 交错

模型在 tool-call 块的之前 / 之后 / 之间发出叙述文本。
解析器必须正确切分内容：文本 → `normal_text`，调用 →
`tool_calls`。

- 适用于每一个 tool-call 解析器。
- 当叙述是 reasoning 内容（`<think>...</think>` 或类似形式）时，
  该测试还会练习 reasoning 解析器的交接。如 reasoning 解析器
  也对该输入运行，请同时打 `REASONING.batch.2` 标签。

### 子案例

跨实现一致性矩阵的 `b8` 列展示出明显分歧（vLLM 与 SGLang
在 XML 风格族中会把 wrapper 之后的尾部文本丢弃；Dynamo 保留）。
子案例把测试所练习的位置形态固定下来，让分歧落到精确的行上。

- **`PARSER.batch.8.a`** 仅 tool call **之前** 的叙述 ——
  模型先发文本，再发一次 tool call，之后没有其他内容。多数
  解析器都能处理；这是历史上的理想路径。
- **`PARSER.batch.8.b`** 仅 tool call **之后** 的叙述 ——
  模型先发 tool call，再发尾部文本。常见分歧点：vLLM 与 SGLang
  通常在 wrapper 闭合处截断。
- **`PARSER.batch.8.c`** tool call **前后都有** 叙述（夹心）。
  组合 `.a` 与 `.b` 形态；用作超集断言。
- **`PARSER.batch.8.d`** **多个 tool call 之间** 的叙述 ——
  文本 → 调用 → 文本 → 调用 → 文本。测试调用之间的文本是否
  被保留（不仅仅是首尾）。

四个子案例共享同一个解析器契约——`tool_calls` 被提取，
`normal_text` 按位置保留。`.a`/`.b`/`.c` 是单调用形态，
仅在叙述位置上有差异；`.d` 是多调用交错形态。四者上的
断言都落在 `normal_text` 上。

## `PARSER.batch.9` —— 空 content / 空 `tool_calls` 数组 / null 响应

引擎发出带 `delta.content = ""` 的 chunk，或最终响应中
`tool_calls: []`，或参数内部出现 `null`。

- 适用于每一个 tool-call 解析器。
- 参数内部的 null 处理在解析器层（DSML 的
  `parse_parameters` 通过 `serde_json::Value::Null` 处理）。
  空 choices / 空 stream 处理通常在 e2e 集成层。

## `PARSER.batch.10` —— 重复 tool call（同名两次）

一个响应中对同一函数名的两次调用，可能 args 也相同。

- 适用于每一个 tool-call 解析器。
- 期望行为：两次调用都必须出现在 `tool_calls` 中且 ID 不同。
  （runtime / 客户端负责决定是否真要重复调用。）

---

## `PARSER.stream.1` —— 单个 tool call 跨 N 个 chunk

一次完整 tool call 通过任意大小的多个 SSE chunk 传送。
解析器增量重建该调用。

- 适用于每一个 tool-call 解析器。生产环境的主路径。

## `PARSER.stream.2` —— 多个 tool call，每个跨 N 个 chunk

响应中两次或更多调用，每次跨多个 chunk。
解析器必须在每次完整调用到达时立即输出，且不能在调用之间混用参数。

- 适用于每一个 tool-call 解析器。

## `PARSER.stream.3` —— 部分 token 切分

chunk 边界把语法 token 切在中间（起始 fence、结束 fence，
或参数名 / 值跨越 chunk 边界）。部分匹配必须返回"继续缓冲"，
而不是把已到达的内容当作普通文本刷出，并在后续 chunk 上完成匹配。

- 适用于每一个 tool-call 解析器。

## `PARSER.stream.4` —— 流式终止

末 chunk 到达时 `finish_reason=tool_calls`（或 `length` /
`stop`）。解析器把进行中的调用刷出（或按 `PARSER.batch.5` 显式
暴露截断）。

- 适用于每一个 tool-call 解析器。

---

## `PARSER.fmt.1` —— 函数名约定

解析器在 tool 函数名中接受的标识符字符，以及它识别的前缀变体
（`functions.NAME` vs 裸 `NAME`）。

- 与语法相关：适用于发出命名 tool-call ID 并自行做校验的
  解析器。多数 XML 与 harmony 解析器都属于此类。
- 各模型发出的内容不同。解析器必须取一个明确立场并用测试固定下来，
  以免未来某次 tokenizer 变更悄悄丢失合法调用。
- 参数信封形态相关问题（原生 ID 保留、JSON 字段顺序、参数键别名）
  归于 `PARSER.fmt.5`，不在此处。`PARSER.fmt.1` 严格只关心
  函数名表面。

## `PARSER.fmt.2` —— 空白 / 格式宽容

无论语法 token 内部或之间是否有偶然空白（`<|tool_call_begin|>`
之后的换行、函数 ID 周围的空格、arg JSON 内部填充等等），
解析器都接受同一个逻辑调用。

- 与语法相关。适用于任何允许 token 之间空白变化的解析器。
- 严格拒绝空白也是合法选择——无论哪种，把行为锚定下来。

## `PARSER.fmt.3` —— Token / 线格式变体

同语义的多种合法拼写。例如：

- **Kimi K2**：单数 `<|tool_call_section_*|>` vs 复数
  `<|tool_calls_section_*|>` section token。
- **Mistral**：pre-v11（`[TOOL_CALLS][{"name":...,"arguments":...}]`
  JSON 数组体） vs v11+（`[TOOL_CALLS]name{...args}`
  名后接对象）。两种形态来自不同 tokenizer 版本；
  生产流量两者都有。
- **Llama 3**：带或不带 `<|python_tag|>` 起始 fence——
  内层 JSON 相同，外层信封不同。
- **Hermes**：`qwen25` 解析器名别名解析为同一个
  `ToolCallConfig::hermes()` 配置；两个名字必须分发结果一致。

解析器必须接受所有已配置的变体并拒绝未为当前 config 注册的变体。

- 与语法相关。当解析器的 config 或注册表枚举多于一种 token 形态拼写时适用。

## `PARSER.fmt.4` —— 空 section / 无内容 wrapper

起止 fence 之间什么都没有
（`<|tool_calls_section_begin|><|tool_calls_section_end|>`）。
解析器必须产生零个调用，并把周围文本作为 `normal_text` 保留。

- 与语法相关。适用于具有成对起止 fence 的解析器。

## `PARSER.fmt.5` —— 参数形态约定

调用体内部的 JSON 信封形态。与 `PARSER.fmt.1`（函数名表面）不同——
这些轴描述参数对象的布局，而非函数标识符。

三个子轴：

1. **原生 call-ID 保留** —— 当模型发出自己的 call ID
   （例如 Kimi K2 的 `functions.NAME:N`）时，解析器必须在
   `ToolCall.id` 上原样保留它，而不是用解析器内部计数器合成。
   参考：vLLM Kimi K2 PR #32768。
2. **JSON 字段顺序宽容** —— `{name, arguments}` 与
   `{arguments, name}` 都必须解析成功，包括 `arguments`
   自身包含一个名为 `"name"` 的键的情况。参考：
   vLLM Mistral 的 `argument_before_name` 与
   `argument_before_name_and_name_in_argument`，参数化 ID 出现在
   3 个测试函数中：`test_extract_tool_calls_pre_v11_tokenizer`、
   `test_extract_tool_calls_streaming_pre_v11_tokenizer`、
   `test_extract_tool_calls_streaming_one_chunk`。
3. **参数键别名** —— `arguments` 与 `parameters` 可互换。
   参考：vLLM Llama 3 JSON 的
   `test_extract_tool_calls_with_arguments_key`。

- 与语法相关。适用于线格式中嵌入命名键 JSON 对象的 JSON 族
  解析器（Mistral、Llama 3 JSON、Hermes）；子轴 1 也适用于
  暴露模型自发 ID 的 XML 族解析器（具体是 Kimi K2）。
- 每个子轴可独立标记 N/A——例如 Hermes 没有原生 call-ID 表面，
  其子轴 1 即 N/A。

---

## `PARSER.xml.1` —— XML entity / HTML 反转义处理

参数值中包含 XML 编码的 entity（`&lt;`、`&amp;`、
`&quot;`、`&apos;`、像 `&#38;` 这样的数字 entity），必须在
把值暴露给客户端前解码。

- 仅适用于 XML 族 tool-call 解析器：`hermes`、`glm47`、
  `qwen3_coder`、`minimax_m2`、`kimi_k2`（虽然 Kimi K2 的
  外层 fence 是特殊 token，但内部参数负载是 XML 风格的）。
- **DSML N/A** —— `string="true|false"` 属性告诉解析器是
  JSON 反编码还是原样透传；没有 entity 解码这一步。
- **JSON 族与 Harmony N/A** —— JSON 由 `serde_json` 处理自身
  转义语义。

## `PARSER.xml.2` —— 基于 schema 的类型强制

解析器使用声明的 tool schema，按声明的参数类型把字符串 args
强制为 number / bool / array。

- 仅适用于线格式中没有显式类型注解的 XML 族解析器。`xml/parser.rs`、
  `glm47_parser.rs` 会做这件事。
- **DSML N/A** —— `string="true|false"` 属性按参数携带了
  类型意图，无需查 schema。
- **JSON 族 N/A** —— JSON 有原生类型。
- **Harmony N/A** —— payload 是 channel 信封内的 JSON。

---

## `PARSER.harmony.1` —— Channel / recipient 解析

OpenAI Harmony 的 token 流携带 channel 元数据
（`<|channel|>analysis|commentary|final<|message|>`）和 recipient
目标（`to=functions.foo`）。解析器必须把 `commentary` channel 的
内容路由到 tool-call 提取，而把 `analysis` 暴露为 reasoning、
`final` 暴露为对用户可见的输出。

- **仅 Harmony。** 其他族 N/A。

## `PARSER.harmony.2` —— 信封标签语法

Harmony 把 tool call 与 reasoning 包在多 tag 信封中：
`<|channel|>commentary to=functions.X <|constrain|>json<|message|>{...}<|call|>`，
对应的 reasoning 信封是 `<|channel|>analysis<|message|>...<|end|>`。
解析器必须正确穿越其各种合法变体：

- **完整信封** —— 所有 tag 齐全（理想路径）。
- **缺 `<|start|>` / assistant 前缀** —— 模型输出落在已有 turn 内部。
- **缺 `<|call|>`（截断恢复）** —— 引擎在发送中途触发
  `max_tokens`；明确行为（恢复或显式暴露错误）。
- **同一 turn 内 reasoning + tool** ——
  `<|channel|>analysis ... <|end|><|start|>assistant<|channel|>commentary ...`
  链。
- **流式 chunk 边界穿过信封** —— chunk 切在 `<|constrain|>`、
  `<|message|>`、`to=functions.X` 等内部。流式解析器必须
  继续缓冲直到下一个 tag 完成。

在 harmony 格式文本上练习时与其他分类交叉
（例如对 harmony 风味的截断恢复打 `// PARSER.batch.5, PARSER.harmony.2`）。

- **仅 Harmony。** 其他族 N/A。

---

## 客户事件回归测试

当一个测试因某个具体客户 ticket / PR / GH issue 揭露的 bug 而存在时，
请在 `#[test]` 注释中内联给出该来源：

```rust
#[test] // PARSER.batch.5 (PR #8208)
fn test_parse_malformed_no_section_end() { ... }
```

分类标签仍然指明所测试的分类；圆括号内指明原始事件。
不需要单独的 "regression" 分类法。

---

## 适用性总览

| 分类块 | 解析器 | 备注 |
| -- | -- | -- |
| `PARSER.batch.{1..10}` | 全部 | 通用行为契约 |
| `PARSER.stream.{1..4}` | 全部 | 流式接口；同样的逻辑案例由不同 harness 驱动 |
| `PARSER.fmt.{1..5}` | 与语法相关 | 仅在语法允许时需要每个变体；`.5`（参数形态）偏 JSON 族 |
| `PARSER.xml.{1,2}` | 仅 XML 族 | entity 解码 + 基于 schema 的强制 |
| `PARSER.harmony.{1,2}` | 仅 Harmony | channel 路由 + 信封变体 |

## 添加新 tool-call 解析器：必备清单

最小可行集合：

1. `PARSER.batch.{1, 2, 3}` —— 基线正确性。
2. `PARSER.batch.4`（或显式 `N/A`）—— 处理或拒绝异常输入。
3. `PARSER.batch.5` —— 锚定外层 fence 缺失时的行为。
   静默丢弃即是潜在回归。
4. `PARSER.batch.{6, 7}` —— 空与复杂 args。
5. `PARSER.stream.{1, 2, 3, 4}` —— 流式。对在流式前端之后的
   任何解析器而言基本不可妥协。
6. `PARSER.batch.8` —— 文本交错。
7. `PARSER.batch.{9, 10}` —— 空/null 与重复调用。
8. 适用时的格式变体（`PARSER.fmt.{1..5}`）：覆盖解析器语法允许的
   全部。不适用的标 `N/A`。JSON 族解析器（Mistral、Llama 3 JSON、
   Hermes）应特别锚定 `PARSER.fmt.5`（参数形态：原生 ID /
   字段顺序 / arguments-vs-parameters 别名）。
9. 适用时的家族特定分类：XML 语法用 `PARSER.xml.{1, 2}`，
   Harmony 用 `PARSER.harmony.{1, 2}`。

reasoning 解析器请见 `REASONING_CASES.md`。
