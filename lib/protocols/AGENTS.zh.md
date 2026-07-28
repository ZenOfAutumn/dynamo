# lib/protocols

Dynamo HTTP 接口的 OpenAI 兼容请求/响应类型。基于 `async-openai` crate 构建，并在我们需要而上游不会接受或尚未合并的行为上做了选择性的 Dynamo 自有覆盖。

如果你要扩展或调试这里的类型，请在编辑前通读本文件。每次改动都围绕一个核心问题：**这是否上游 re-export，还是我们自己拥有？** 本文档让答案保持一致。

## 核心张力

`async-openai` 维护良好但对输入宽松度（input-laxity）的 PR 推进缓慢。维护者通常希望改动严格匹配 OpenAPI 规范，即便 OpenAI 的*托管*接口对输入接受比规范更宽松的形态。参见 `64bit/async-openai#535`（可选 `ReasoningItem.id`）以及之前关于可选 `OutputMessage.id`/`status` 的工作 —— 这些都来自规范理论上会拒绝的真实 Agents-SDK / Codex 流量。

我们不能让 Dynamo 阻塞在上游合并上。我们也不能 fork 整个 crate —— 它非常庞大且更新频繁。我们最终的规则是：**默认 re-export 上游；只拥有最窄的类型子树以修复我们需要的行为**。

## 所有权准则

默认使用上游。仅在以下条件至少满足一项时才拥有某类型：

1. **上游拒绝真实客户端发送的形态。** 主要驱动场景。例如：上游将 `OutputMessage.id`/`status`/`annotations` 标为必填，但 Codex / Agents SDK 在输入上经常省略。
2. **我们需要为 schema 添加 Dynamo 特有字段**，且不应进入上游。例如：`CreateChatCompletionRequest.mm_processor_kwargs`（vLLM 多模态）、`ChatCompletionRequestAssistantMessage.reasoning_content`（R1 / QwQ）、`ChatCompletionStreamOptions.continuous_usage_stats`。
3. **上游类型强加的形态会破坏下游后端。** 例如：上游 `FunctionCall.arguments` 是 `String`；LangChain 等会以对象形式发送。我们拥有 `FunctionCall`，通过自定义 deserializer 同时接受两者并归一化为 `String`。
4. **上游存在已知 bug。** 例如：`ChatCompletionMessageToolCall.type` 不一定被序列化；我们用 `#[serde(default = "default_function_type")]` 拥有它以保留连线兼容。

**不要因为相邻类型被拥有就也拥有该类型。** 保持影响范围小。如果拥有 `OutputMessage` 会级联到 `Response`、`OutputItem`、流事件以及一半 crate —— 停下，找到更窄的修复（见下文「命名：避免双向冲突」）。

## 布局

- `src/types/chat.rs` —— Chat Completions（请求、响应、流、消息）。大量自有：多模态内容、reasoning、continuous usage stats、灵活 `arguments`。
- `src/types/responses/mod.rs` —— Responses API（Codex、Agents SDK）。输入链自有；输出链完全 upstream。
- `src/types/completion.rs` —— Legacy completions。基本 upstream。
- `src/types/anthropic.rs` —— Anthropic Messages API。完全自有（`async-openai` 中无对应类型）。
- `src/types/embeddings`、`src/types/images` —— 完全 upstream re-export（无 Dynamo 扩展）。

## Re-export 约定

当需要选择性 shadow 时，使用**显式 re-export**（`pub use foo::{A, B, C}`），而不是 glob。Glob（`pub use foo::*`）允许在模块顶部使用 —— Rust 允许本地的 `pub struct Foo` shadow 一个 glob 引入的 `Foo`（glob 仅会发出 `unused_imports` 警告）。但显式列表让所有权拆分对读者更显眼，并且当上游重命名或移除某类型时能在编译期捕获错误。

`src/types/responses/mod.rs` 使用 glob re-export，因为表面非常大（200+ 类型）。`src/types/chat.rs` 使用显式列表，因为表面可控且 Dynamo 拥有更多。两种模式都可接受；按你需要枚举多少类型来排除自己拥有的来选择。

## 命名：避免双向冲突

**陷阱。** 上游有时会在请求输入与响应输出两侧重用同一类型。`OutputMessage` 是典型例子：它出现在 `MessageItem::Output(...)` 内（输入侧 —— 之前的 assistant 轮次被回放）以及 `OutputItem::Message(...)` 内（输出侧 —— 我们刚刚生成的 assistant 消息）。

如果我们放宽 `OutputMessage`（让 `id`/`status` 可选）并 shadow 上游名字，每处构造 `OutputItem::Message(OutputMessage { ... })` 的输出侧代码都会破坏：`OutputItem::Message` 变体持有的是上游类型，不是我们的，我们放宽的 struct 不匹配。

朴素的修复是同时拥有 `OutputItem`。但这会级联到 `Response`、流事件以及大量子类型。正确的修复更小：

**规则。** 如果某类型在输入与输出两侧都被上游复用，就给 Dynamo 拥有的输入侧变体一个*不同的名字*。输出侧通过 glob/显式 re-export 继续使用上游名字。

`responses/mod.rs` 中当前的命名：

- `InputOutputMessage` —— Dynamo 自有、放宽；用于输入侧 `MessageItem::Output(...)`。
- `OutputMessage` —— 上游、未变；用于输出侧 `OutputItem::Message(...)`。
- 同样的模式适用于 `InputOutputMessageContent`（输入）vs 上游 `OutputMessageContent`（输出）以及 `InputOutputTextContent`（输入）vs 上游 `OutputTextContent`（输出）。

仅输入的类型可以使用相同名称 shadow 上游 —— 没有冲突。当前的 shadow：`MessageItem`、`Item`、`InputItem`、`InputParam`、`CreateResponse`。

## 具体到 Responses 输入链

截至撰写本文，自有的输入链如下：

```
CreateResponse
└── input: InputParam            (shadow)
    └── InputItem                (shadow)
        ├── ItemReference        (upstream)
        ├── EasyInputMessage     (upstream)
        └── Item                 (shadow，与上游变体一一对应)
            ├── Message(MessageItem)  (shadow)
            │   ├── Input(InputMessage)  (upstream)
            │   └── Output(InputOutputMessage)  (新名字 —— 已放宽)
            │       └── content: Vec<InputOutputMessageContent>  (新名字)
            │           └── OutputText(InputOutputTextContent)   (新名字 —— 已放宽)
            └── ... 19 个其他上游变体（FunctionCall、Reasoning 等）
```

`Item` 必须与上游变体一一对应，因为它是 `#[serde(tag = "type")]` 枚举 —— 我们无法继承变体。如果上游为 `Item` 新增变体，我们这里也必须加，否则携带该类型的负载将反序列化失败。这是上游漂移咬到我们的唯一一处；接受它作为拥有此链的代价。

输出链（`Response`、`OutputItem`、`OutputMessage`、流事件等）完全 upstream。我们在输出时铸造合法的 id/status，无需放宽，也没有理由拥有它。

## 当上游最终合并放宽时

若上游 PR 落地，使某字段变为可选（与我们放宽的一致），检查清单：

1. 在 `Cargo.toml` 中升 `async-openai`。
2. 如果我们的自有覆盖现在与 upstream 完全相同则删除；如果上游只是部分放宽则收窄。
3. 更新消费点（如果上游仍保留必填字段，则将 `Option<T>` 转回 `T` 等）。
4. 跑完整测试套件；序列化形态测试应能捕捉任何回归。

不要把多余的 Dynamo 自有类型「以防万一」留在原地。死所有权就是技术债。

## 当上游重命名或重构我们 re-export 的类型

Glob re-export 会静默接受重命名。显式 re-export 则会编译失败 —— 这正是它的用意。更新显式列表与任何消费代码，确认无语义漂移，跑测试。

## 测试模式

- 序列化形态测试（`lib/llm` 中的 `test_response_wire_format_shape`）验证序列化 JSON 与 API 规范匹配。修改自有类型时主要依赖它们。
- 自有类型的反序列化测试应同时覆盖放宽形态（拥有它的原因）与严格形态（证明我们没有破坏规范一致客户端）。
- 当为自有类型添加新 Dynamo 字段时，加入一个省略它的测试，断言默认行为。

## 明确*不属于*本 crate 的事

- HTTP 传输（请求执行、重试、流帧解析）—— 那是 `lib/llm/src/http/`。
- API 类型间的语义转换（Responses → Chat、Anthropic → Chat 等）—— 在 `lib/llm/src/protocols/`，使用本处定义的类型。
- 模型特定分词或 prompt 模板。

让本 crate 保持声明式：类型、serde derive、builder、`From` 转换。业务逻辑放在下游。

## 常见错误

- 因为某类型*靠近* bug 而拥有它，而非因为 bug 本身。把修复收窄。
- shadow 双向类型时未检查输出侧构造点。重命名前先 `grep` 整个 workspace 上的构造调用。
- 通过本地包装 struct + `#[serde(default)]` 给 upstream re-export 类型加字段。这行不通 —— serde 无法为外部类型注入默认值，除非用 `#[serde(remote)]`，而它需要字段一一对应，并不能解决 optional-vs-required 的不匹配。
- 添加变体时忘记更新 `From` impl。编译器会捕捉穷举匹配，但不会检测非穷举枚举上 `From<Ours> for Upstream` 的变体数量。
