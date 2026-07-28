# Backend Common (Rust)

为 Rust LLM 后端提供共享的运行时粘合层。两类抽象：
`Worker`（运行时生命周期）与 `LLMEngine`（针对引擎特定逻辑的 trait）。参考实现位于 `lib/backend-common/examples/mocker/`。

引擎直接使用 `PreprocessedRequest` 与 `LLMEngineOutput` —— 与 Rust 流水线其它部分使用的类型一致。不再需要 Python 风格的请求/响应包装类型。

## 引擎生命周期

```
construct (e.g. from_args)  ->  start()  ->  generate() / abort()  ->  cleanup()
         |                        |                |                        |
    parse args,            start engine,     serve requests           shutdown,
    return engine          return metadata   (concurrent)             release resources
```

trait 共有四个方法。`from_args` **不在** trait 上 —— 每个后端暴露自己的特定构造函数（通常是同步的固有方法 `from_args(argv) -> Result<(Self, WorkerConfig)>`）。这样可以在不引入 `where Self: Sized` 选退的前提下保持 trait 完全 object-safe，并让 `run.rs` 不需要泛型。

- `start(&self) -> Result<EngineConfig, DynamoError>` —— 在 `&self` 上使用内部可变性（interior mutability），以便 `Arc<dyn LLMEngine>` 可驱动生命周期。
- `generate(&self, request, ctx) -> Result<BoxStream<'static, LLMEngineOutput>, DynamoError>`
  —— 流式推理（streaming inference）。作者返回普通的 stream；框架负责将其包装为 `Annotated` 并接入取消机制。
- `abort(&self, ctx)` —— 可选，默认空实现。框架仅在活跃请求期间触发 `ctx.stopped()` 或 `ctx.killed()` 时调用 —— **不会** 在静默丢弃 stream（TCP reset、消费侧超时等）时调用。如果存在必须在所有丢弃路径上都执行的请求级清理（释放调度器槽、释放引擎句柄），请将释放逻辑通过 RAII 写在 `generate` 的 stream body 内；`abort` 仅用于带外通知（例如告诉远端调度器尽早取消计算）。
- `cleanup(&self) -> Result<(), DynamoError>` —— 在 shutdown 时被调用一次。只要 `start()` 成功，即使后续注册或 serve 失败，也保证会被运行。

## `generate` 的契约

stream item 的类型是 `Result<LLMEngineOutput, DynamoError>`。

必须恰好有一个**终止项（terminal item）** 作为最后一项被 yield。终止项是以下之一：

  * `Ok(chunk)` 且 `finish_reason = Some(...)`，或
  * `Err(dynamo_err)` 携带类型化的中途失败（mid-stream failure）。

非终止项是 `Ok(chunk)` 且 `finish_reason` 未设置。终止 `Ok` chunk 可以携带 token（completion 的最终若干 token）也可以为空 —— 契约规定 `finish_reason`（或 `Err`）标记结束。

终止 `Ok` chunk 上的 `completion_usage` 是可选的（但建议附带 —— OpenAI 前端在存在时会聚合它）。

在 debug 构建中，框架会用一个验证器（validator）包装 stream，违反规则时直接 panic —— 在开发与测试中提供响亮失败，在 release 构建中编译掉。

验证器强制的规则：

1. 终止项（终止 `Ok` chunk 或 `Err`）之后不得再 yield 任何项。

验证器**不**强制 “stream 必须以终止 chunk 结束” —— stream 可能因合理原因提前结束（适配器在引擎最终 yield 之前对取消做了 break）。一致性测试套件（conformance kit）通过 `ConformanceFailure::NoTerminalChunk` 捕获缺失终止项的情况，请用它对你的引擎做端到端验证。

引擎**必须**在 yield 之间轮询 `ctx.is_stopped()`，并在被取消时发出 `finish_reason` 为 `FinishReason::Cancelled` 的终止 chunk。一致性测试套件强制此约束 —— 取消之后的任何其它 `finish_reason`（`Length`、`Stop` 等）都被视为引擎忽略了取消信号。框架还会运行一个带外 monitor，在 `stopped()` 或 `killed()` 触发时调用 `engine.abort(ctx)` —— 用于释放引擎侧资源（KV slot、调度器条目），与流内取消检查并行运行。

## 输出构造

非终止 chunk 使用 `chunk::token(id)`。终止 chunk 使用上游 `LLMEngineOutput` 的构造函数，可链式使用 `LLMEngineOutputExt` 提供的链式 setter：

```rust
use dynamo_backend_common::{chunk, LLMEngineOutput, LLMEngineOutputExt, usage};

// 非终止
yield Ok(chunk::token(id));

// 因长度终止，附带最终 token 与 usage 统计。
yield Ok(LLMEngineOutput::length()
    .with_tokens(vec![final_id])
    .with_usage(usage(prompt_len, n)));

// 因取消终止，附带部分 usage 统计。
yield Ok(LLMEngineOutput::cancelled()
    .with_usage(usage(prompt_len, generated)));

// 类型化的中途错误 —— 下游保留 BackendError 变体。
yield Err(DynamoError::builder()
    .error_type(ErrorType::Backend(BackendError::InvalidArgument))
    .message("bad request")
    .build());

// 上游也可用：LLMEngineOutput::stop()、LLMEngineOutput::error(msg)。
```

终止 chunk 上的 `completion_usage` 是可选的 —— 前端在存在时会聚合。`usage(prompt, completion)` 会自动计算 `total_tokens`（溢出时饱和）。

## 设计约束

- **跨引擎实现零重复（ZERO duplication）。** 在 `LLMEngine` impl 内编写逻辑前，先检查同样的逻辑是否已存在于另一个引擎。如果有，提取到 `Worker` 或共享工具中。

- **恰好两类类型。** `Worker` 拥有运行时生命周期，`LLMEngine` 拥有推理。不允许中间 trait 或 mixin。

- **Object-safe 的 trait。** `Arc<dyn LLMEngine>` 必须可用。所有方法接收 `&self`。构造函数与后端绑定，不在 trait 上。

- **非泛型的 `Worker`、`EngineAdapter` 与 `run()`。** 它们都持有 `Arc<dyn LLMEngine>`。这对第二阶段（PyO3 绑定）至关重要：Python 引擎将通过实现同一 trait 的 `PyLLMEngine` 适配器接入。

- **复用 `DynamoError`。** trait 方法返回 `DynamoError`（来自 `dynamo_runtime::error`），即工作区内统一的错误类型。引擎产生的失败使用 `ErrorType::Backend(BackendError::X)`，其中 `BackendError` 是运行时内嵌的分类枚举。backend-common 内部不引入自定义错误类型。

## 新增引擎

1. 创建一个依赖 `dynamo-backend-common` 的新 Rust crate。按仓库 Rust crate 约定将其放置于 `lib/` 下（如 `lib/<backend>-rs/`）。**不要**将 Rust crate 放在 `components/src/dynamo/` 下 —— 那是 Python 包命名空间。
2. 在 `src/<backend>_engine.rs` 中：`struct YourEngine; impl LLMEngine for YourEngine`。再加一个固有 `impl YourEngine { pub fn from_args(argv) -> Result<(Self, WorkerConfig), DynamoError> }`。
3. 实现 `start`、`generate`、`cleanup`（必需）以及 `abort`（可选）。
4. 编写 `src/main.rs`：
   ```rust
   use std::sync::Arc;
   mod your_engine;

   fn main() -> anyhow::Result<()> {
       let (engine, config) = your_engine::YourEngine::from_args(None)?;
       dynamo_backend_common::run(Arc::new(engine), config)
   }
   ```
5. 以 `lib/backend-common/examples/mocker/` 中的 `engine.rs` 为模板。
6. 对你的引擎运行一致性测试套件（见下方“测试”部分）。

## 错误处理

引擎在 `start`、`generate`、`cleanup` 与 `from_args` 中返回 `DynamoError`。使用 builder 模式构造：

```rust
use dynamo_backend_common::{BackendError, DynamoError, ErrorType};

return Err(DynamoError::builder()
    .error_type(ErrorType::Backend(BackendError::InvalidArgument))
    .message(format!("bad param: {reason}"))
    .build());
```

**约定：backend-common 与所有引擎实现发出的错误都使用 `ErrorType::Backend(BackendError::X)`。** 从前端/路由的角度看，所有经由后端层向上传递的错误都来自“后端” —— Worker 框架代码与引擎实现代码之间的内部划分对外不可见（也不相关）。顶层 `ErrorType::X` 变体保留给非后端代码路径（流水线传输、前端解析、路由调度）。

常见的嵌套类别：`BackendError::InvalidArgument`（引擎或初始化拒绝输入）、`BackendError::CannotConnect`（无法连到发现 / 依赖）、`BackendError::EngineShutdown`（引擎启动失败 / 崩溃）、`BackendError::Unknown`（未分类）、`BackendError::StreamIncomplete`（引擎完成前 stream 已结束）、`BackendError::Cancelled`、`BackendError::ResponseTimeout`、`BackendError::Disconnected`、`BackendError::ConnectionTimeout`。

中途错误有两种等价的终止形式：

  * **类型化（推荐）**：从 stream 中 yield `Err(DynamoError)`。会作为 `Annotated::error` 转发并端到端保留 `ErrorType::Backend(...)` 变体。当失败类别对调用方有意义时（如区分 `BackendError::InvalidArgument` 与 `Disconnected`）使用此方式。
  * **字符串**：yield `Ok(LLMEngineOutput::error(msg))`。便于纯消息级失败。会丢失类型化的 `BackendError` 变体。

## 日志

让所有 Rust 引擎的日志风格保持统一。在某个引擎中新增或修改某条日志时，检查同样的生命周期事件在其它引擎中是否记录，并将其调整一致。

级别约定：
- `tracing::info!` 用于生命周期里程碑（引擎已启动、开始 serve、cleanup 完成）。`Worker` 已经发出 “Serving {model} on …” 与 “Engine cleanup complete” —— 引擎代码仅在这些消息未覆盖该事件时才补充。
- `tracing::debug!` 用于按请求事件（请求 abort、取消）。
- `tracing::warn!` 用于可恢复的问题。
- `tracing::error!` 仅用于不可恢复的失败。

## 测试

启用 `testing` cargo feature 以引入一致性测试套件：

```toml
[dev-dependencies]
dynamo-backend-common = { workspace = true, features = ["testing"] }
```

```rust
use dynamo_backend_common::testing;

#[tokio::test]
async fn my_engine_satisfies_contract() {
    let engine = MyEngine::new_for_test();
    testing::run_conformance(engine).await.expect("conformance");
}
```

测试套件断言：

- `start()` 返回非空的 `EngineConfig.model`。
- 单次 `generate()` 产出一个良构的 stream，且以终止 chunk 结束（`finish_reason` 已设置；`completion_suage` 可选）。
- 8 次交错的 `generate()` 调用全部成功完成（捕获并发轮询下的共享状态 bug）。
- 在中途触发 `stop_generating()` 后，stream 在 2 秒截止时间内终止（否则报告 `CancellationNotObserved`）。如果最后一个 chunk 不是 `FinishReason::Cancelled` 终止项 —— 任何其它终止原因，或没有终止项 —— 则报告 `ConformanceFailure::CancellationIgnored`。
- `cleanup()` 成功且幂等（连续两次调用都返回 Ok）。

还可使用：`testing::mock_context()` 与 `testing::cancelling_context(after)` 用于手写测试。

## 关键文件

| 文件 | 作用 |
|------|-------------|
| `engine.rs` | `LLMEngine` trait、`EngineConfig`、`chunk::token`、`LLMEngineOutputExt` setter、`usage()` 辅助。重导出 `PreprocessedRequest` / `LLMEngineOutput` / `FinishReason` 等。|
| `worker.rs` | `Worker` —— 运行时生命周期：创建 `DistributedRuntime`、注册模型、serve 端点、cleanup。`WorkerConfig` 在此。|
| `adapter.rs` | `EngineAdapter` —— 将 `LLMEngine` 桥接到 `AsyncEngine`。取消监控 + debug 构建下的 validator 包装。|
| `run.rs` | `pub fn run(engine, config)` —— 各后端 `main.rs` 使用的入口。非泛型。|
| `args.rs` | `CommonArgs` —— 共享 CLI 参数（`--namespace`、`--component` 等），每个引擎的 `Args` 都会扁平化注入。|
| `error.rs` | 重导出 `dynamo-runtime` 的 `DynamoError`、`ErrorType`、`BackendError`。无自定义错误类型。|
| `validate.rs` | debug 构建下的 stream validator。release 构建编译掉。|
| `testing.rs` | 一致性测试套件。受 `testing` feature 门控。|

## 第二阶段

后续计划提供 PyO3 绑定，使 Python 后端运行时变成本 crate 之上的薄包装。trait 与数据类型的设计已支持这一点，无需大重构 —— 不要在此处提前为第二阶段搭建脚手架。
