# 遥测落盘工具（telemetry）

> 一套通用的「进程内事件总线 + 落盘 sink」基础设施，供 audit（审计）和 agent trace（智能体追踪）等功能把结构化记录低开销地分发并写到文件。

## 这个模块解决什么问题

dynamo 的多个功能（审计日志、agent 调用追踪）都需要做同一件事：把结构化记录从热路径上「甩出去」，然后异步批量写盘，且不能拖慢推理。这个模块把这套通用能力抽象出来，避免每个功能各写一遍：

- 一个泛型的进程内广播总线（fanout bus），生产者发布、多个 sink 订阅。
- 几种落盘 sink：纯 JSONL 与滚动 gzip JSONL。
- 一个流式辅助器，在流跑完时发出完成信号。

注意它只提供「管道与落盘」机制，记录的具体内容由调用方（audit、agents）定义。

## 目录结构

| 文件 | 职责 |
|---|---|
| `mod.rs` | 模块入口。声明子模块，提供 `parse_sink_names`——从逗号分隔的环境变量解析 sink 名单（空则默认 `["stderr"]`） |
| `bus.rs` | `TelemetryBus<T>`：基于 `tokio::sync::broadcast` 的泛型进程内扇出总线，懒初始化 |
| `jsonl.rs` | channel 驱动的缓冲 JSONL sink（`JsonlSinkOptions`），内部包一个 `Recorder<T>` |
| `jsonl_gz.rs` | 滚动 gzip JSONL sink（`JsonlGzipSinkOptions`）：批量压缩追加，按未压缩字节数 / 行数阈值滚动分段 |
| `stream.rs` | `PassThroughWithCompletion`：透传一个 `Stream`，在流耗尽时通过 `oneshot` 通知完成 |

## 核心概念（Java 视角）

- **`TelemetryBus<T>`**（`bus.rs`）：≈ 进程内的发布订阅总线 / Guava `EventBus`。底层是 `tokio` 的 `broadcast` channel（≈ 多消费者的 `BlockingQueue`，每个订阅者都能收到全部消息）。用 `OnceLock` 懒初始化 ≈ Java 的 `volatile` + 双检锁单例。`T: Clone + Send` 约束 ≈ 泛型 bound，要求记录可跨线程克隆。
- **`JsonlSinkOptions` / `JsonlGzipSinkOptions`**：≈ 配置 POJO（缓冲字节、flush 间隔、滚动阈值），带 `Default` 实现 ≈ 默认构造器给出的缺省配置。
- **sink + `mpsc`**（`jsonl.rs` / `jsonl_gz.rs`）：记录经 `tokio::sync::mpsc`（≈ 单消费者 `BlockingQueue`）送给后台写盘任务，配合 `CancellationToken` 优雅停机 ≈ 协作式中断标志。
- **`parse_sink_names`**（`mod.rs`）：纯函数，做大小写归一化和去空，决定启用哪些 sink。

## 与其他模块的关系

- 被 `crate::audit`（`audit/sink.rs`、`audit/bus.rs`）使用，落地审计日志。
- 被 `crate::agents::trace`（`agents/trace/sink.rs`、`relay.rs`、`mod.rs`）使用，落地 agent 调用追踪。
- `jsonl.rs` 依赖 `crate::recorder::{Recorder, RecorderOptions}` 完成实际写入。
- 依赖外部 crate：`tokio`（broadcast/mpsc/oneshot）、`flate2`（gzip）、`serde`（序列化）、`tokio_util`（`CancellationToken`）。

## 阅读建议

1. 先读 `mod.rs`，看清四个子模块和 `parse_sink_names` 的取舍逻辑。
2. 再读 `bus.rs` 的 `TelemetryBus<T>`，理解「一处发布、多 sink 订阅」的中枢。
3. 想看落盘细节，从简单的 `jsonl.rs` 入手，再看 `jsonl_gz.rs` 的滚动分段逻辑。
4. 最后看 `stream.rs` 的 `PassThroughWithCompletion`，理解流结束时如何触发收尾。
