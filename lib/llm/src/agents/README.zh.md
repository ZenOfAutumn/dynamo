# 智能体支持模块（agents）

> 为 agentic（多步推理 / 工具调用）工作负载提供请求身份元数据与一套"尽力而为"的链路追踪（trace）能力。

## 这个模块解决什么问题

在 dynamo 的 LLM 推理链路里，一个 agent 会话往往由很多次 LLM 调用和工具调用串成一条"轨迹"。本模块做两件事：
1. 定义 `AgentContext`（会话/轨迹身份标识），它通过请求的 `nvext.agent_context` 字段进入系统，用来把分散的 LLM 调用归并到同一个 agent run。
2. 在请求结束时发出 request 级指标记录，并把外部 harness 进程上报的 tool 事件中继进来，统一落到本地 sink（stderr / JSONL / gzip JSONL）。注意：这套 trace 是**尽力而为**而非审计级——记录可能延迟、丢失，消费方必须按 `event_time_unix_ms` 排序，不能依赖落盘顺序。

## 目录结构

| 文件 / 子目录 | 职责 |
|---|---|
| `mod.rs` | 模块装配：声明 `context` 与 `trace` 两个子模块 |
| `context.rs` | 定义 `AgentContext`（会话/轨迹身份），含必填字段非空校验与 builder |
| `trace/mod.rs` | trace 子系统入口：持有全局追踪总线 `BUS`、`init_from_env*`、`publish`/`subscribe`，并 re-export 公共类型 |
| `trace/types.rs` | 全部 trace 数据结构：`AgentTraceRecord` 及其枚举/子结构 |
| `trace/config.rs` | 从环境变量加载 `AgentTracePolicy`（是否启用、sink 列表、容量、JSONL 滚动参数等） |
| `trace/record.rs` | 构造/校验记录：`emit_request_end`、`publish_tool_record`、`validate_tool_record` |
| `trace/sink.rs` | `TraceSink` trait 及 stderr / JSONL / JSONL-gzip 三种实现，并为每个 sink 启动后台 worker |
| `trace/relay.rs` | `AgentToolEventRelay`：本地 ZMQ PULL 套接字，把外部 harness 的工具事件中继到 Dynamo event plane |
| `trace/integration.rs` | 与主链路对接的辅助函数（从 `RequestTracker` 提取指标、启动工具事件 ingest/relay） |
| `trace/replay.rs` | 为 replay 场景计算输入序列哈希（`request_replay_metrics`） |
| `CLAUDE.md` / `AGENTS.zh.md` / `CLAUDE.zh.md` | 给协作工具/人的说明文档 |

## 核心概念（Java 视角）

- **`AgentContext`（context.rs）**：一个不可变的身份值对象（含 `session_type_id`、`session_id`、`trajectory_id`、可选 `parent_trajectory_id`）。类似 Java 里用 `@Builder` 生成的 DTO；`#[serde(deserialize_with = ...)]` 相当于在反序列化时做 Bean Validation，必填字段为空会直接抛出"反序列化异常"。

- **`AgentTraceRecord`（types.rs）**：trace 总线上流动的统一记录。它的 `event_type` 是带数据的枚举（`RequestEnd` / `ToolStart` / `ToolEnd` / `ToolError`），`request` 和 `tool` 两个字段二选一填充——类似 Java 的 sealed class + Optional 字段建模一条"要么是请求记录、要么是工具记录"的事件。

- **`TraceSink` trait（sink.rs）**：`async fn emit(&self, rec: &AgentTraceRecord)`，等价于一个带异步方法的 Java `interface`。stderr/JSONL/gzip 各是一个实现类；运行期用 `Arc<dyn TraceSink>`（≈ 持有接口引用的线程安全共享指针）持有。

- **追踪总线 `BUS`（mod.rs 里的 `TelemetryBus<AgentTraceRecord>`）**：一个 broadcast channel，发布端在请求热路径上只做一次 `publish`（≈ 往 `BlockingQueue` 投递），真正的文件/网络 I/O 由后台 worker 异步消费，从而不阻塞推理主流程。

- **`AgentTracePolicy`（config.rs）**：用 `OnceLock`（≈ 懒加载的线程安全单例）缓存从环境变量解析出的配置。所有 `DYN_AGENT_TRACE_*` 开关都在这里集中读取。

## 与其他模块的关系

- **入口被谁触发**：`entrypoint/input.rs` 在启动时调用 `agents::trace::init_from_env_with_shutdown` 和 `start_tool_event_ingest_from_policy`。
- **热路径调用**：`preprocessor.rs` 在请求处理过程中调用 `record_llm_metric_tokens`、`request_replay_metrics`、`request_metrics` 和 `emit_request_end`（全部由 trace 子系统提供）。
- **请求入口**：`AgentContext` 经 `protocols/openai/nvext.rs` 的 `NvExt.agent_context` 字段进入系统。
- **依赖**：`trace/sink.rs` 复用 `crate::telemetry`（jsonl / jsonl_gz / TelemetryBus）；`trace/relay.rs` 依赖 `dynamo_runtime` 的 event plane 与 `crate::utils::zmq`；`trace/replay.rs` 依赖 `dynamo_kv_router` 与 `dynamo_tokens` 的哈希工具。

## 阅读建议

1. 先读 `context.rs` 的 `AgentContext`——它是整个 agent 身份模型的根，最简单。
2. 再读 `trace/types.rs` 的 `AgentTraceRecord` 及其枚举，建立"一条 trace 记录长什么样"的认知。
3. 然后看 `trace/mod.rs`，理解 `BUS` 的 publish/subscribe 模型与 `init_from_env_with_shutdown` 的启动流程。
4. 想看数据怎么产生，读 `trace/record.rs::emit_request_end`；想看怎么落盘，读 `trace/sink.rs::spawn_workers`；想看外部工具事件如何进来，读 `trace/relay.rs::AgentToolEventRelay`。
