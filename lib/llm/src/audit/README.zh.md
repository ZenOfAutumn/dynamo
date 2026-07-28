# 请求审计模块（audit）

> 把"用户的 chat completion 请求 + 最终聚合后的响应"作为一条审计记录，异步落到可配置的 sink（stderr / NATS / JSONL / gzip JSONL）。

## 这个模块解决什么问题

在 dynamo 的 HTTP 前端处理 chat completion 时，业务上常需要留存完整的请求/响应用于审计、回放或离线分析。本模块负责：
1. 判断某条请求是否需要审计（依据 `store` 标志或 `DYN_AUDIT_FORCE_LOGGING`）。
2. 对**流式响应**做"旁路聚合"——边把 chunk 原样转发给客户端，边在后台把所有 chunk 聚合成一份完整响应，供审计使用。
3. 在请求结束时把 `AuditRecord` 投递到审计总线，由后台 worker 异步写到各 sink，不阻塞推理热路径。

## 目录结构

| 文件 | 职责 |
|---|---|
| `mod.rs` | 模块装配 + `init_from_env*`：根据 policy 初始化总线并启动各 sink 的 worker |
| `config.rs` | 从环境变量加载 `AuditPolicy`（是否启用、sink 列表、容量、JSONL 滚动参数、`force_logging`） |
| `handle.rs` | 核心数据 `AuditRecord` 与 `AuditHandle`；`create_handle` 决定是否审计，`emit` 投递到总线 |
| `bus.rs` | 审计总线封装（基于 `crate::telemetry::bus::TelemetryBus<AuditRecord>`）：`init`/`subscribe`/`publish` |
| `sink.rs` | `AuditSink` trait 及 stderr / NATS / JSONL / gzip JSONL 四种实现，并为每个 sink 启动后台 worker |
| `stream.rs` | 流式聚合工具：`scan_aggregate_with_future`、`fold_aggregate_with_future`、`PassThroughWithAgg` |

## 核心概念（Java 视角）

- **`AuditRecord`（handle.rs）**：可序列化的审计记录值对象，含 `request_id`、`model`、可选的完整 `request` / `response`（用 `Arc<...>` 共享，≈ 不可变的共享引用，避免深拷贝）。`#[serde(skip_serializing_if = "Option::is_none")]` 相当于 Jackson 的 `@JsonInclude(NON_NULL)`。

- **`AuditHandle`（handle.rs）**：一次请求生命周期内的"审计上下文句柄"。先 `create_handle` 创建，过程中 `set_request`/`set_response` 填充，最后调用一次 `emit`（消费 self，Rust 的 move 语义保证"只能 emit 一次"，类似 Java 里用完即弃的一次性对象）。

- **`AuditSink` trait（sink.rs）**：`async fn emit(&self, rec: &AuditRecord)`，等价于一个异步 Java `interface`。stderr / NATS / JSONL / gzip 各是实现类，运行期以 `Arc<dyn AuditSink>` 持有。

- **审计总线（bus.rs）**：基于 broadcast channel 的发布-订阅。热路径只 `publish`（≈ 往 `BlockingQueue` 投递），后台每个 sink 一个 worker `subscribe` 后异步消费。worker 在收到 shutdown 信号后会 drain 剩余记录再退出。

- **`PassThroughWithAgg<S>`（stream.rs）**：实现了 `Stream` trait 的包装器（≈ 装饰器模式包住上游 `Stream`），它把每个 chunk 原样向下游转发，同时累积起来；流结束时通过一个 `oneshot` channel（≈ 一次性的 `CompletableFuture`）把聚合后的完整响应交回给审计侧。

## 与其他模块的关系

- **被谁初始化**：`entrypoint/input.rs` 启动时调用 `audit::init_from_env_with_shutdown`。
- **被谁使用**：HTTP/预处理热路径（`preprocessor.rs`）调用 `audit::handle::create_handle` 创建句柄，并用 `audit::stream::scan_aggregate_with_future` / `fold_aggregate_with_future` 做流式/非流式聚合。
- **依赖**：`sink.rs` 复用 `crate::telemetry`（jsonl / jsonl_gz）和 `dynamo_runtime::transports::nats`；`handle.rs`、`stream.rs` 依赖 `crate::protocols::openai::chat_completions` 的请求/响应类型与 `DeltaAggregator`。
- 与 `agents` 模块同构：两者都用 `TelemetryBus` + sink + 后台 worker 的模式，但 audit 关注完整请求/响应，agents::trace 关注 agent 轨迹指标。

## 阅读建议

1. 先读 `handle.rs`：`AuditRecord`（落盘格式）+ `create_handle`（启用判定逻辑）+ `AuditHandle::emit`（投递点）。
2. 再读 `config.rs::AuditPolicy` 与 `mod.rs::init_from_env_with_shutdown`，理解配置与启动流程。
3. 然后读 `sink.rs`：先看 `AuditSink` trait 与四个实现，再看 `spawn_workers_from_env` 的 select/drain 循环。
4. 最后读 `stream.rs::scan_aggregate_with_future`，理解流式响应如何在不影响客户端的前提下被聚合用于审计。
