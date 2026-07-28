# 请求预处理器（preprocessor）

> 把外部 API 请求（OpenAI / Anthropic 等）翻译成推理引擎能直接消费的 token 序列，并在响应路径上把引擎产出的 token 反向还原成协议响应。

## 这个模块解决什么问题

在整个 dynamo LLM 推理链路里，前端 HTTP 收到的是「人类可读」的协议请求（聊天消息、工具定义、采样参数等），而后端推理引擎只认 token id 和数值参数。preprocessor 就是这两端之间的「翻译层」：

- **请求方向**：套用 chat 模板 → 收集多模态数据 → 分词（tokenize）→ 产出内部统一的 `PreprocessedRequest`。
- **响应方向**：把引擎流式吐出的 token 反分词（detokenize）、做推理内容（reasoning）解析、工具调用解析（tool calling jail），再组装成 `NvCreateChatCompletionStreamResponse` 等协议响应。

它以 pipeline `Operator` 的形式插在前端和后端之间（见下文）。

## 目录结构

| 文件 / 子目录 | 职责 |
|---|---|
| `../preprocessor.rs` | 模块入口（同名文件）。定义核心类型 `OpenAIPreprocessor`、`LLMMetricAnnotation`，以及请求/响应两个方向的全部转换逻辑 |
| `prompt.rs` + `prompt/` | chat 模板渲染。定义 `OAIChatLikeRequest`、`OAIPromptFormatter` trait 和 `PromptFormatter`；`prompt/template*` 是 minijinja 模板引擎；`prompt/deepseek_*` 是 DeepSeek 系列模型的专用格式化器 |
| `tools/` | 工具调用（function calling）的请求侧类型，如 `ToolType`、`ToolChoice`、`Function` |
| `media/` | 多模态数据（图片/音频/视频）的下载与解码，含 RDMA 传输。已有英文 `README.md` 与中文 `README.zh.md` |
| `speculative_prefill.rs` | 投机式 prefill 相关支持 |
| `lightseek_mm.rs` | 多模态 per-image token 计数（受 `lightseek-mm` feature 控制），用于 MM-aware KV 路由 |

## 核心概念（Java 视角）

- **`OpenAIPreprocessor`**（`preprocessor.rs`）：核心服务对象，持有 tokenizer、formatter、model_info 等不可变状态，用 `Arc<Self>` 共享（≈线程安全的单例 Service Bean）。它实现了 pipeline 的 `Operator` trait，签名是 `SingleIn<NvCreateChatCompletionRequest> → ManyOut<Annotated<...StreamResponse>>`，相当于一个把「请求 DTO」转成「流式响应」的过滤器。
- **`OAIChatLikeRequest` / `OAIPromptFormatter` trait**（`prompt.rs`）：≈ Java 的两个 interface。前者抽象「一个长得像 OpenAI 聊天请求的东西」（暴露 `messages()`、`tools()` 等方法），后者抽象「如何把这种请求渲染成 prompt 字符串」。这样不同模型家族（DeepSeek、通用 OAI）可以提供各自的实现而上层代码不变。
- **`PromptInput` / `TokenInput` / `TextInput` enum**（`prompt.rs`）：带数据的 enum ≈ Java 的 sealed class。表示输入既可能是已经分好的 token，也可能是待分词的纯文本，单条或批量。
- **`LLMMetricAnnotation` struct**（`preprocessor.rs`）：用 serde 序列化（≈ Jackson）的指标载体，记录 input/output/cached token 数、prefill/decode worker 归属、分词延迟等，挂在响应流的 `Annotated` 旁路上供观测。

## 与其他模块的关系

- 依赖 `crate::protocols`：输入类型来自 `protocols::openai`（`NvCreateChatCompletionRequest` 等），输出类型来自 `protocols::common::{llm_backend, preprocessor}`（`PreprocessedRequest`、`BackendOutput`）。
- 依赖 `crate::model_card::ModelDeploymentCard`（MDC）：构造时从 MDC 读取 tokenizer、模板、runtime 配置。
- 依赖 `crate::tokenizers`（分词）、`dynamo_parsers`（reasoning 解析）、`crate::preprocessor::media`（多模态）。
- 被 `crate::entrypoint` / 前端 pipeline 装配使用，作为请求链路中的一个 `Operator`。

## 阅读建议

1. 先读 `preprocessor.rs` 顶部的模块 doc 注释（translation/apply/prompt/tokenize 四阶段说明）。
2. 看 `OpenAIPreprocessor` 结构体定义（约 209 行）和 `new` / `preprocess_request`（约 406 行），理解请求方向。
3. 跳到文件底部的 `impl Operator ... for OpenAIPreprocessor`（约 2364 行）的 `generate`，看请求/响应如何串成 pipeline。
4. 响应方向重点看 `transform_postprocessor_stream`、`apply_tool_calling_jail`、`parse_reasoning_content_from_stream`。
5. 想理解模板渲染，再读 `prompt.rs` 的 `OAIChatLikeRequest` / `OAIPromptFormatter`。
