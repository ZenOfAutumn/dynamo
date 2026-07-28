# 后端模块

两类抽象：`Worker`（运行时集成）与 `LLMEngine`（引擎特定逻辑的 ABC）。完整文档见 `README.md`。

## 引擎生命周期

```
from_args(argv) -> start() -> generate()/abort() -> drain() -> cleanup()
     |                |              |                  |          |
  parse args,    start engine,   serve requests    drain in-flight, shutdown,
  return config  return metadata (concurrent)      then cleanup    release resources
```

1. `from_args(argv)` —— classmethod 工厂方法。解析 CLI 参数，返回 `(engine, WorkerConfig)`。此时引擎尚未启动。
2. `start()` —— 启动引擎，返回 `EngineConfig`。该方法返回后，`generate()` 必须已经准备好接收调用。
3. `generate(request, context)` —— 流式推理，会被并发调用。
4. `abort(context)` —— 取消正在进行的请求（可选，默认空实现）。
5. `drain()` —— 清理前在后端侧执行 drain（可选，默认空实现）。会在服务发现注销 + 宽限期后被调用；用于在运行时仍存活时完成进行中的 NIXL 传输（issue #7319）。
6. `cleanup()` —— 在关闭时仅调用一次。

## 设计约束

- **跨引擎实现零重复。** 这是首要优先级。
  这个模块存在的全部理由就是消除 vllm、sglang 与 trtllm 之间不断滋生的代码重复。在 `LLMEngine` 子类中编写任何逻辑前，先确认相同的逻辑是否已经存在于另一个引擎中。如果存在，就把它抽取到 `Worker` 或公共工具中，让所有引擎调用共享版本。
  在新增功能时，永远先问：“这是引擎特有的还是通用的？”如果两个或更多引擎都需要相同代码，那它就是通用的。

- **严格保持两个类。** `Worker` 持有运行时生命周期；`LLMEngine` 持有推理逻辑。不要新增中间基类或 mixin。

- **`from_args()` 返回 `(engine, WorkerConfig)`。** 元组返回让契约可以静态检查 —— 子类如果忘了构造 `WorkerConfig`，会在类型检查阶段就报错，而非运行期才出 `AttributeError`。

- **`generate()` 委托给引擎并附带取消监听。**
  取消监听位于 Rust 侧（`dynamo_backend_common::EngineAdapter`）：它为每个请求生成一个任务，监听 `ctx.stopped()` / `ctx.killed()`，并在取消时调用 `engine.abort(context)`。Python 侧的 ABC `generate()` 只负责 yield chunks —— 跨流取消逻辑与异常 → `BackendError` 的映射由 bridge 负责。

- **`start()` 返回 `EngineConfig`。** model 类需要注册元数据（`context_length`、`block_size`、`total_kv_blocks`），但不应直接访问引擎内部。`start()` 返回这些元数据，使得边界保持干净。

- **不使用 hooks。** 如果需要在多个引擎间共享行为，请放在 `Worker` 或公共工具中，不要建立 hook 系统。

- **并行路径。** 现有的 `main.py` / `worker_factory.py` / `init_llm.py` 入口保持不动。`unified_main.py` 文件是另一条独立路径。修改本模块时不要破坏或修改现有后端。

## 请求 / 响应契约

`GenerateRequest` 与 `GenerateChunk`（`engine.py`）是 `TypedDict`，用于约束 `generate()` 的签名。`GenerateRequest` 包含必填的 `token_ids`，以及可选的 `sampling_options`、`stop_conditions` 与 `output_options`。`GenerateChunk` 包含必填的 `token_ids` 与 `index`（单选 chunk 时使用 `index=0`），以及可选的 `finish_reason` 与 `completion_usage`（最终 chunk 中均必填）。引擎可以读取后端特定的请求 key，但响应 chunk 的 key 在使用前应先纳入共享契约。

构造 `completion_usage` dict 时直接内联即可。Finish reason 的归一化（例如 `"abort"` → `"cancelled"`）由 Rust 层处理。

## 新增引擎

1. 创建 `<backend>/llm_engine.py` 并继承 `LLMEngine`
2. 实现必需的 `from_args()`、`start()`、`generate()`、`cleanup()`，以及可选的 `abort()`
3. `from_args()` 必须解析参数并返回 `(engine, WorkerConfig)`
4. 创建 `<backend>/unified_main.py` 调用 `run(<YourEngine>)`
5. 以 `sample_engine.py` 作为参考实现

## 错误处理

`Worker` 会将生命周期与 generate 错误封装为 `DynamoException` 子类（`dynamo.llm.exceptions`）。Rust bridge（`engine.rs`）会将其转换为有类型的 `DynamoError::Backend(...)`，从而获得完整的错误链可观测性。引擎可以直接从 `generate()` 抛出 `DynamoException` 子类 —— 这些异常会原样透传。非 `DynamoException` 的错误会被包装为 `Unknown`。

## 日志

请保持**三个引擎（vllm、sglang、trtllm）的日志规范一致**。
当你在某个 `llm_engine.py` 中新增或修改一条日志时，检查另外两个引擎是否也记录了相同的生命周期事件，并同步更新它们。目标是让运维无论使用哪个后端都能看到一致的日志形态，便于在混合部署中排障。

统一规范：
- `logger.info` 用于生命周期里程碑：引擎初始化完成、开始服务、引擎关闭。
- `logger.debug` 用于每条请求级别事件：请求中断、取消。
- `logger.warning` 用于可恢复问题：空输出、异常的 finish reason。
- `logger.error` 仅用于不可恢复的失败。

## 关键文件

| 文件 | 作用 |
|------|-------------|
| `engine.py` | `LLMEngine` ABC —— 引擎必须实现的唯一接口 |
| `worker.py` | `Worker` —— 对 `dynamo._core.backend.Worker` 的轻薄封装；生命周期状态机与信号处理在 Rust 侧（`lib/backend-common`） |
| `run.py` | 通用入口 —— 所有 `unified_main.py` 都调用 `run(engine_cls)` |
| `sample_engine.py` | 参考引擎 —— 既作模板也用于测试 |

Rust 侧的 `Worker`（位于 `lib/backend-common/src/worker.rs`）负责：
  - 生命周期状态机（Init → Running → Stopped）
  - SIGTERM/SIGINT 处理与优雅关闭编排
    （服务发现注销 → 宽限期 → drain → cleanup）
  - engine.cleanup() 返回后的 3 阶段分布式 runtime 关闭

状态机与编排器的不变式由同一 crate 内的 Rust 单元测试钉死。不要在 Python 侧重新实现它们。
