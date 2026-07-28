# lib/protocols

为 Dynamo 的 HTTP 接口提供与 OpenAI 兼容的请求/响应类型。底层基于 `async-openai` crate，并在我们需要某些上游暂未接受或尚未合并的行为时，有选择地用 Dynamo 自有的覆盖（override）来替换。

如果你要在这里扩展或调试类型，请先完整阅读本文件再动手。每次改动都绕不开同一个核心问题：**这个类型我们到底是直接 re-export 上游的，还是自己拥有？** 这份文档的存在，就是为了让答案保持一致。

## 核心矛盾

`async-openai` 维护良好，但在"放宽输入约束"类的 PR 上推进很慢。其维护者一般倾向于让代码完全匹配 OpenAPI 规范，即便 OpenAI 的*托管* API 在输入上其实接受比规范更宽松的形式。参见 `64bit/async-openai#535`（可选的 `ReasoningItem.id`）以及之前关于可选 `OutputMessage.id`/`status` 的工作——这些都源自 Agents-SDK / Codex 的真实流量，但严格按规范是不合法的。

我们没法卡在等待上游合并上，也没法 fork 整个 crate——它体量巨大、更新频繁。我们最终的规则是：**默认 re-export 上游，只对能修复我们所需行为的最小类型子树取得所有权。**

## 所有权决策准则

默认走上游。只有在以下至少一种情形下才自己拥有某个类型：

1. **上游拒绝真实客户端发送的形状。** 主要场景。例如：上游把 `OutputMessage.id`/`status`/`annotations` 标为必填，但 Codex / Agents SDK 在输入中经常省略它们。
2. **我们需要给 schema 加一个不属于上游的 Dynamo 专有字段。** 例如：`CreateChatCompletionRequest.mm_processor_kwargs`（vLLM 多模态）、`ChatCompletionRequestAssistantMessage.reasoning_content`（R1 / QwQ）、`ChatCompletionStreamOptions.continuous_usage_stats`。
3. **上游类型强制了一种破坏下游后端的形状。** 例如：上游 `FunctionCall.arguments` 是 `String`；LangChain 等会以对象形式发送。我们拥有 `FunctionCall`，通过自定义反序列化器同时接受两种形式并归一化为 `String`。
4. **上游存在已知 bug。** 例如：`ChatCompletionMessageToolCall.type` 此前不总是被序列化；我们用 `#[serde(default = "default_function_type")]` 拥有它，以保持线上协议兼容。

**不要仅因为相邻类型被自己拥有就一并拥有。** 把"爆炸半径"控制到最小。如果拥有 `OutputMessage` 会级联导致还要拥有 `Response`、`OutputItem`、流式事件以及大半个 crate——停下来，找个更窄的修法（见下文"命名：避免双侧冲突"）。

## 布局

- `src/types/chat.rs` —— Chat Completions（请求、响应、流式、消息）。大量自有：多模态内容、推理（reasoning）、连续用量统计、灵活的 `arguments`。
- `src/types/responses/mod.rs` —— Responses API（Codex、Agents SDK）。输入链自有；输出链完全使用上游。
- `src/types/completion.rs` —— 旧版 completions。基本上沿用上游。
- `src/types/anthropic.rs` —— Anthropic Messages API。完全自有（`async-openai` 中没有等价物）。
- `src/types/embeddings`、`src/types/images` —— 完全 re-export 上游（无 Dynamo 扩展）。

## Re-export 约定

当你需要选择性地覆盖（shadow）时，使用**显式 re-export**（`pub use foo::{A, B, C}`），而不是 glob。Glob（`pub use foo::*`）允许放在模块顶部——Rust 允许本地 `pub struct Foo` 覆盖一个 glob 引入的 `Foo`（glob 只会发出 `unused_imports` 警告）。但显式列表能让所有权划分对读者一目了然，并且当上游重命名或删除某个类型时能在编译期就抓到错误。

`src/types/responses/mod.rs` 使用 glob re-export，因为类型面非常大（200+ 个）。`src/types/chat.rs` 使用显式列表，因为面是可控的，而且 Dynamo 自有的占比更高。两种模式都可以接受；选择哪种取决于：要排除掉自己拥有的那部分，需要枚举多少类型。

## 命名：避免双侧冲突

**陷阱。** 上游有时会在请求输入端和响应输出端复用同一个类型。`OutputMessage` 是经典例子：它既出现在 `MessageItem::Output(...)`（输入侧——回显之前的 assistant 轮次），也出现在 `OutputItem::Message(...)`（输出侧——我们刚生成的 assistant 消息）。

如果我们放宽 `OutputMessage`（把 `id`/`status` 改为可选）并覆盖上游同名类型，那么所有在输出侧构造 `OutputItem::Message(OutputMessage { ... })` 的地方都会出问题：`OutputItem::Message` 变体内部持有的是上游类型，不是我们的，而我们放宽后的结构体并不匹配。

直接的修法是连 `OutputItem` 也一起拥有，但这会级联到拥有 `Response`、流式事件以及它们一长串的子类型。正确的修法更小：

**规则。** 如果某个类型被上游同时用在输入侧和输出侧，给 Dynamo 自有的输入侧变体取一个*不同的名字*。输出侧通过 glob / 显式 re-export 继续使用上游名字。

`responses/mod.rs` 当前的命名：

- `InputOutputMessage` —— Dynamo 自有，已放宽；用于输入侧的 `MessageItem::Output(...)`。
- `OutputMessage` —— 上游，未改动；用于输出侧的 `OutputItem::Message(...)`。
- `InputOutputMessageContent`（输入）vs 上游 `OutputMessageContent`（输出），以及 `InputOutputTextContent`（输入）vs 上游 `OutputTextContent`（输出），遵循相同的模式。

仅在输入侧出现的类型，可以用同名覆盖上游——不会冲突。当前的覆盖：`MessageItem`、`Item`、`InputItem`、`InputParam`、`CreateResponse`。

## Responses 输入链具体形态

截至撰写时，自有的输入链如下：

```
CreateResponse
└── input: InputParam            (shadow)
    └── InputItem                (shadow)
        ├── ItemReference        (upstream)
        ├── EasyInputMessage     (upstream)
        └── Item                 (shadow, mirrors upstream variant-for-variant)
            ├── Message(MessageItem)  (shadow)
            │   ├── Input(InputMessage)  (upstream)
            │   └── Output(InputOutputMessage)  (NEW NAME — relaxed)
            │       └── content: Vec<InputOutputMessageContent>  (NEW NAME)
            │           └── OutputText(InputOutputTextContent)   (NEW NAME — relaxed)
            └── ... 19 other upstream variants (FunctionCall, Reasoning, etc.)
```

`Item` 必须逐一镜像上游变体，因为它是 `#[serde(tag = "type")]` 枚举——我们无法继承变体。如果上游往 `Item` 加新变体，我们这边也必须加，否则携带该类型的 payload 反序列化会失败。这是上游漂移唯一会咬到我们的地方；接受它，作为拥有该链的代价。

输出链（`Response`、`OutputItem`、`OutputMessage`、流式事件等）完全沿用上游。我们在输出时会生成合法的 id/status，因此不需要放宽，也没有理由自己拥有。

## 当上游最终合并了某项放宽

如果某个上游 PR 落地，把某字段改为可选（与我们放宽后的形状一致），处理清单为：

1. 在 `Cargo.toml` 中升级 `async-openai`。
2. 如果自有的 override 已经和上游一致，则删除它；如果上游只部分放宽，则相应缩小。
3. 更新调用方（如果上游字段仍存在但仍非可选，则把 `Option<T>` 改回 `T` 等等）。
4. 跑全套测试；序列化形状测试应能抓到任何回退。

不要"以防万一"地保留多余的 Dynamo 自有类型。死的所有权就是技术债。

## 当上游重命名或重构了我们 re-export 的类型

Glob re-export 会静默地接住重命名。显式 re-export 会编译失败——这正是它的意义。更新显式列表与所有调用方，确认没有语义漂移，跑测试。

## 测试模式

- 序列化形状测试（`lib/llm` 中的 `test_response_wire_format_shape`）会校验我们的序列化 JSON 是否匹配 API 规范。改动自有类型时要靠它们兜底。
- 自有类型的反序列化测试要同时覆盖放宽后的形状（之所以自己拥有的原因）和严格形状（证明没破坏符合规范的客户端）。
- 给自有类型新增 Dynamo 专有字段时，加一个省略该字段的测试，断言默认行为正确。

## 明确**不**属于本 crate 的职责

- HTTP 传输（请求执行、重试、流式帧解析）—— 那是 `lib/llm/src/http/` 的事。
- API 类型之间的语义转换（Responses → Chat、Anthropic → Chat 等）—— 这部分在 `lib/llm/src/protocols/`，会使用本处定义的类型。
- 模型相关的分词或 prompt 模板。

让本 crate 保持声明式：类型、serde derive、builder、`From` 风格的转换。业务逻辑放下游。

## 常见错误

- 因为某个类型 *靠近* bug 而拥有它，而不是因为 bug 本身。把修复缩到最窄。
- 在覆盖一个双侧类型前，没有检查输出侧的构造点。重命名前请先在 workspace 中 `grep` 构造调用。
- 通过给本地包装结构体加 `#[serde(default)]` 来给"上游 re-export 类型"加字段。这行不通——除非用 `#[serde(remote)]`，serde 无法给外部类型注入默认值，而 `#[serde(remote)]` 又要求字段对字段镜像，对"可选 vs 必填"不匹配也无能为力。
- 在新增变体时忘记更新 `From` 实现。编译器会强制检查穷尽匹配，但当目标枚举是 non-exhaustive 时，`From<Ours> for Upstream` 上的变体数量不会被检查。
