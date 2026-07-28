# 流式响应性能记录与分析（perf）

> 以极低开销录制 LLM 的流式响应（带时间戳），事后再对录制数据做性能与 logprob 分析。

## 这个模块解决什么问题

LLM 推理的输出是一个 token 流（streaming）。要做性能分析（首 token 时延、token 间隔、吞吐）或质量/不确定性分析（每个位置候选 token 的概率分布），需要在不显著拖慢主链路的前提下，把流里每个响应连同时间戳记录下来。`perf` 提供一个包装在 `AsyncEngineStream` 外层的录制流：边转发边记录（或纯消费），流结束后通过一个 oneshot channel 把整段录制结果交出去供离线分析；`logprobs` 子模块则在录制结果之上做 logprob 提取与"模型不确定性/接近度"分析。

## 目录结构

| 文件 / 子目录 | 职责 |
|---|---|
| `perf.rs`（父模块，目录外） | 录制核心：`TimestampedResponse`、`RecordedStream`、`RecordingStream`、`RecordingMode`，以及 `record_stream*` 系列工厂函数 |
| `logprobs.rs` | logprob 提取与分析：`TokenLogprob` / `TokenLogProbs`、`LogprobExtractor` trait、`SensitivityAnalysis` 及 `analyze_logprob_sensitivity` 等 |

## 核心概念（Java 视角）

- **`TimestampedResponse<T>`**：给一条响应贴上 `Instant` 时间戳和 0 起序号的包装。`Instant` 是单调高精度时钟（≈ `System.nanoTime()` 的语义，专用于测耗时）。
- **`RecordingStream<R>`**：实现了 `Stream`（≈ 异步迭代器 / 响应式 `Publisher`）的装饰器，包住上游流。它在 `poll_next` 里根据 `RecordingMode` 决定行为，是整个模块的引擎。`impl Unpin` 让它可安全移动。
- **`RecordingMode`（enum）**：`Scan` = 边记录边把响应透传给下游（clone 一份留底）；`Sink` = 当作终点，只消费不下发（直接 move，不 clone，零拷贝开销）。带数据的判别在这里没有，但它是典型的 Rust 二选一枚举（≈ 只有两个实例的 enum）。
- **`RecordedStream<T>`**：录制完成后的不可变结果容器，含全部 `responses`、`start_time`、`end_time`，提供 `response_count()`、`total_duration()` 等。这是所有分析的输入对象。
- **oneshot channel（`RecordedStreamReceiver`）**：流结束（`Poll::Ready(None)`）时，`RecordingStream` 把 `RecordedStream` 通过 `oneshot::Sender` 发出，调用方持有 `oneshot::Receiver` 异步 `.await` 取结果。oneshot ≈ 一次性的 `CompletableFuture`，只能 send/await 一次。
- **`CapacityHint`（trait）**：让请求类型预估响应条数，从而对录制用的 `Vec` 预分配容量（`Vec::reserve`），减少扩容开销，类似给 `ArrayList` 传初始 capacity。
- **`LogprobExtractor`（trait，logprobs.rs）**：定义"从某种响应类型里抽取 logprob"的接口，已为 `NvCreateChatCompletionStreamResponse` 实现。`TokenLogProbs` 把"被选中 token + 候选 token"按概率排好序。`analyze_logprob_sensitivity` 接收 `Arc<RecordedStream<impl LogprobExtractor>>`（共享只读的录制结果），产出 `SensitivityAnalysis`，用于检测"前两名候选概率非常接近"的位置（模型在该处犹豫）或"是否近似贪心解码"。

## 与其他模块的关系

- `perf.rs` 依赖 `dynamo_runtime::engine` 的一组 trait/类型：`AsyncEngineStream`、`AsyncEngineContext`、`AsyncEngineContextProvider`、`ResponseStream`、`EngineStream`、`DataStream`、`Data`。`RecordingStream` 实现这些 trait，因此可无缝替换原始引擎流。
- `logprobs.rs` 依赖 `crate::perf::RecordedStream` 和 `crate::protocols::openai::chat_completions::NvCreateChatCompletionStreamResponse`。
- 录制流以装饰器形式插入到引擎/前端的响应管线中，对上下游透明（保留原 `AsyncEngineContext`）。

## 阅读建议

1. 从 `perf.rs` 的 `RecordingStream::poll_next` 看起，对照 `RecordingMode::Scan` 与 `Sink` 两个分支，理解"透传 + 录制"与"纯消费"的差异，以及流结束时如何经 oneshot 交付 `RecordedStream`。
2. 再看 `record_stream` 等工厂函数，了解如何把一个普通 `EngineStream` 包成录制流并拿到 receiver。
3. 分析侧：从 `logprobs.rs` 的 `LogprobExtractor::extract_logprobs_by_choice` 入手，然后看 `analyze_logprob_sensitivity` 如何把录制结果转成 `SensitivityAnalysis`。
