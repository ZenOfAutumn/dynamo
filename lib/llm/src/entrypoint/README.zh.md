# 入口编排（entrypoint）

> 把"一个引擎（EngineConfig）"和"一种输入方式（Input）"拼装起来，并真正跑起来的装配层。

## 这个模块解决什么问题

dynamo 既能当 OpenAI 兼容的 HTTP 网关，也能当 KServe gRPC 服务、交互式终端、或纯后端 worker。这些场景的差异其实只有两件事：**引擎从哪来**（本地进程内 / 通过 etcd 发现的远程 worker）和**请求从哪进来**（HTTP / gRPC / stdin / 终端 / endpoint 拉取）。`entrypoint` 模块把这两个维度抽象成 `EngineConfig` 和 `Input` 两个 enum，再由 `run_input` 这一个函数做分发与启动。它处于整条推理链路的最外层，是 CLI / 上层 binary 的统一入口。

## 目录结构

| 文件 / 子目录 | 职责 |
|---|---|
| `entrypoint.rs`（同名模块入口，在上一级目录） | 定义 `EngineConfig`（引擎来源）、`RouterConfig`（路由配置）、`ChatEngineFactoryCallback` 等顶层类型，并 re-export `build_routed_pipeline*` |
| `input.rs` | 定义 `Input` enum 与唯一入口函数 `run_input`，按输入类型分发到各子模块 |
| `input/common.rs` | 公共装配逻辑：`prepare_engine`、`build_pipeline`、`build_routed_pipeline` —— 把引擎接上 preprocessor / backend / router / migration 等算子，组成完整流水线 |
| `input/http.rs` | `run`：构建并运行 OpenAI 兼容 HTTP 服务（含 TLS、模型发现 watcher、动态启用 endpoint） |
| `input/grpc.rs` | `run`：构建并运行 KServe 兼容 gRPC 服务 |
| `input/text.rs` | `run` + `main_loop`：终端交互式聊天 / 单条 stdin prompt |
| `input/endpoint.rs` | `run`：把本地引擎注册成一个可被远程拉取的 endpoint（worker 模式） |

## 核心概念（Java 视角）

- **`enum EngineConfig`**：带数据的 enum，类比 Java 的 sealed class（`Dynamic` / `InProcessText` / `InProcessTokens` 三个子类）。`Dynamic` 表示通过 etcd 发现的远程引擎；两个 `InProcess*` 表示进程内引擎，区别是输入是文本还是 token。`local_model()` 等方法用 `match` 在各分支上取公共字段，相当于在 sealed class 上做 switch。
- **`enum Input`**：同样是 sealed class 风格，列举 5 种输入入口（`Http` / `Stdin` / `Text` / `Endpoint(String)` / `Grpc`）。实现了 `FromStr` / `TryFrom<&str>`，相当于一个把命令行字符串解析成枚举的工厂方法；`Default` 会根据 stdin 是否是终端自动选 `Text` 或 `Stdin`。
- **`async fn run_input(...) -> anyhow::Result<()>`**：唯一总入口。`async` ≈ 返回 `CompletableFuture`；`anyhow::Result` ≈ 受检异常，函数体里大量的 `?` ≈ 自动 throw。它先初始化 trace / audit 等旁路设施（失败只告警、不致命），再 `match in_opt` 分发。
- **`build_routed_pipeline_with_preprocessor`（在 common.rs）**：用 builder 风格的 `.link(...)` 把 frontend → preprocessor → migration → backend → prefill → service_backend 一路串成双向流水线（forward edge 处理请求、backward edge 处理响应）。可类比把若干 Servlet Filter / 拦截器按顺序 link 成一条处理链。
- **`ChatEngineFactoryCallback`**：`Arc<dyn Fn(...) -> Pin<Box<dyn Future<...>>> + Send + Sync>`，即"线程安全、可跨线程共享的异步工厂函数对象"，类比 Java 里一个 `Function<..., CompletableFuture<Engine>>` 的单例。

## 与其他模块的关系

- **依赖**：`crate::http::service`（HTTP 服务 / Metrics）、`crate::grpc::service::kserve`（gRPC 服务）、`crate::discovery`（`ModelManager` / `ModelWatcher` 模型发现）、`crate::backend` / `crate::preprocessor` / `crate::kv_router` / `crate::migration`（流水线算子）、`dynamo_runtime`（`DistributedRuntime`、pipeline 的 `link` 机制、`PushRouter`）、`dynamo_kv_router`（KV 路由配置与 estimator）。
- **被谁用**：上层 binary / CLI（以及 Python binding）构造好一个 `EngineConfig` 和 `Input` 后，调用 `entrypoint::input::run_input` 启动整个服务。可以理解为整个 dynamo-llm 的 `main` 收口点。

## 阅读建议

1. 先看 `input.rs` 的 `enum Input` 和 `run_input`，建立"输入 → 分发"的整体心智模型。
2. 再看 `entrypoint.rs` 的 `enum EngineConfig`，理解三种引擎来源。
3. 然后进 `input/common.rs` 的 `prepare_engine` 和 `build_routed_pipeline_with_preprocessor`，这是流水线装配的核心，也是理解 dynamo 推理链路怎么"接线"的关键。
4. 最后按需挑一个具体入口（推荐先看 `input/http.rs` 的 `run`）看一种输入是如何落地成真实服务的。
