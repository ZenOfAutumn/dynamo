# 协议与消息类型（protocols）

> dynamo LLM 推理链路里所有「消息格式」的集中定义：既包括对外的 OpenAI / Anthropic 兼容 HTTP API，也包括组件之间内部传递的请求/响应类型。

## 这个模块解决什么问题

任何分布式系统都需要约定「线上传的是什么」。这个模块就是 dynamo LLM 的 schema 中心：

- **对外协议**：兼容 OpenAI（chat/completions/embeddings/responses）与 Anthropic Messages API 的请求/响应类型。
- **内部协议**：前端把多种外部 API「漏斗收敛」成统一内部表示，再经预处理流向后端的类型（`PreprocessedRequest`、`BackendOutput` 等）。
- **传输编解码**：SSE（Server-Sent Events）流式协议的编解码，以及张量（tensor）协议。

它本身基本只放数据类型 + serde 序列化（≈ Jackson 的 POJO/DTO 层）和少量转换 trait，不含业务执行逻辑。

## 目录结构

| 文件 / 子目录 | 职责 |
|---|---|
| `../protocols.rs` | 模块入口（同名文件）。声明各子模块，定义 `TokenIdType`、`ContentProvider` trait、`convert_sse_stream` 等公共 helper |
| `openai.rs` + `openai/` | OpenAI 兼容协议。`chat_completions`、`completions`、`embeddings`、`responses`、`images`、`audios`、`videos`、`models`、`tools`、`nvext`（NVIDIA 扩展字段）、`validate`（参数校验）、`stream_aggregator`（流式聚合） |
| `anthropic/` | Anthropic Messages API（`/v1/messages`）类型（`types.rs`）与转内部表示的流转换（`stream_converter.rs`） |
| `common.rs` + `common/` | 引擎/后端层「内部协议」。`common.rs` 含 `CompletionRequest`、`SamplingOptions`、`StopConditions` 等；`common/llm_backend` 含引擎输出，`common/preprocessor` 含 `PreprocessedRequest`，`common/postprocessor`、`common/timing`（`RequestTracker`） |
| `codec.rs` | SSE 流编解码（`SseLineCodec<T>`、`SseCodecError`） |
| `unified.rs` | `UnifiedRequest`：API 无关的内部统一请求包装，保留各 API 特有上下文以便响应侧忠实重建 |
| `tensor.rs` | 张量协议（`DataType` 等），用于非文本 IO |

## 核心概念（Java 视角）

- **`NvCreateChatCompletionRequest` 等 Nv* 类型**（`openai/chat_completions`）：≈ 用 Jackson 注解的 DTO。`Nv` 前缀表示在标准 OpenAI 字段之上加了 NVIDIA 扩展（见 `nvext`）。serde 的 `Serialize`/`Deserialize` 就是 Java 里 `@JsonProperty` 那套。
- **`ContentProvider` / `SamplingOptionsProvider` 等 trait**（`protocols.rs`、`common.rs`）：≈ Java interface，把「能从中取出文本/采样参数」的能力抽象出来，让不同请求类型统一对待。
- **`PromptType` / `FinishReason` / `DataType` 等 enum**：带数据或纯标签的 enum ≈ sealed class / Java enum，枚举出协议里所有合法取值。
- **`UnifiedRequest`**（`unified.rs`）：解决「漏斗架构」中信息丢失问题。多种 API 通过 `TryFrom`（≈带校验的转换构造器，失败返回 `Result` ≈ 受检异常）收敛到 `NvCreateChatCompletionRequest`，`UnifiedRequest` 额外保存原始 API 上下文，以便响应路径还原。
- **`SseLineCodec<T>`**（`codec.rs`）：基于 `tokio_util` 的流式编解码器，把字节流逐行解析成强类型 SSE 消息，再 deserialize 成 `T`。

## 与其他模块的关系

- 被 `crate::preprocessor` 大量依赖：预处理器的输入/输出类型都来自这里。
- 复用 `dynamo_runtime::protocols::annotated::Annotated` 作为统一响应信封，复用 `dynamo_protocols::types` 的底层消息类型（如 `ChatCompletionRequestMessage`、`StopReason`）。
- `common/` 下的内部类型流向 `crate::kv_router`（路由）、`crate::backend` 与各推理引擎。
- 几乎是整个 crate 的「公共词汇表」，被前端 HTTP、router、engine、recorder 等普遍引用。

## 阅读建议

1. 先读 `protocols.rs` 顶部 doc 与子模块声明，建立全局地图。
2. 对外协议从 `openai/chat_completions` 的 `NvCreateChatCompletionRequest` 看起（最常用的入口请求）。
3. 内部协议读 `common.rs`（`CompletionRequest`、`SamplingOptions`、`StopConditions`）和 `common/preprocessor.rs`（`PreprocessedRequest`，已有中文注释）。
4. 想理解多 API 收敛，读 `unified.rs` 顶部 doc 里的 hourglass 架构图。
5. 流式传输实现细节再看 `codec.rs`。
