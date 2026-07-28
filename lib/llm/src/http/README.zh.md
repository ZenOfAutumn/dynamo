# HTTP 服务（http）

> dynamo-llm 的 OpenAI / Anthropic 兼容 HTTP 网关，是整个分布式运行时最主要的对外入口（ingress）。

## 这个模块解决什么问题

这是 dynamo LLM 分布式运行时的"前门"。它用 `axum` 搭起一套 OpenAI 兼容（以及可选的 Anthropic 兼容）的 HTTP endpoint，把进来的 Chat / Completion / Embedding 等请求转发给按模型挂载的引擎，并负责健康检查、Prometheus 指标、连接断开检测、OpenAPI 文档等横切能力。引擎可以通过 `ModelManager` 在运行期动态挂载/卸载（配合模型发现 watcher），所以无需重启就能上下线模型。一个关键设计：无论客户端是否要求 `stream=true`，下游一律按流式处理，非流式请求再由本层聚合成单个响应，从而下游只需处理一种请求-响应模式。

## 目录结构

| 文件 / 子目录 | 职责 |
|---|---|
| `http.rs`（同名模块入口，在上一级目录） | 仅 `pub mod service;` |
| `service.rs` | 子模块聚合 + 定义 `RouteDoc`（路由文档：method + path）；re-export `axum` 和 `Metrics` |
| `service/service_v2.rs` | 核心：`HttpService` / `HttpServiceConfig` + builder、共享状态 `State`、`StateFlags`（各类 endpoint 的开关）、把所有 router merge 成最终 `axum::Router` 并 `run` |
| `service/openai.rs` | 最大的文件：OpenAI 各 endpoint 的 handler 与 `*_router`（chat / completions / embeddings / images / videos / audios / responses / list_models 等） |
| `service/anthropic.rs` | Anthropic Messages API 兼容 endpoint（受 `DYN_ENABLE_ANTHROPIC_API` 控制） |
| `service/metrics.rs` | Prometheus 指标：`Metrics`、`InflightGuard`、`process_response_and_observe_metrics`、`/metrics` 的 router 等（HTTP 与 gRPC 共用） |
| `service/health.rs` | `/health`（就绪）与 live 检查 router |
| `service/disconnect.rs` | 客户端连接断开检测：`ConnectionHandle` / `create_connection_monitor`，用于及时取消已断开请求 |
| `service/busy_threshold.rs` | 繁忙阈值（busy/503）相关 router 与逻辑 |
| `service/realtime.rs` | 实验性 WebSocket endpoint `/v1/realtime`，引擎需通过 `install_engine` 显式注入 |
| `service/openapi_docs.rs` | 基于收集到的 `RouteDoc` 生成 OpenAPI 文档 router |
| `service/error.rs` | HTTP 错误类型/响应转换 |
| `service/clear_kv_blocks.rs` | `clear_kv_blocks_router`：POST `/clear_kv_blocks`，向 worker 广播清空 KV cache |

## 核心概念（Java 视角）

- **`struct HttpService` + `HttpServiceConfigBuilder`**：用 `derive_builder` 的 owned builder（`HttpService::builder().port(...).enable_tls(...).build()`）构造，类比 Spring 里配置好后 build 出的内嵌服务实例。`run(cancel_token)` 启动监听，`spawn` 则丢到后台 task。
- **`struct State`（`Arc<State>`）**：请求间共享的状态，含 `Metrics`、`ModelManager`、`Discovery`、`StateFlags`、`CancellationToken`。用 `Arc` 包裹后塞进每个 handler（axum 的 `State` extractor），类比 Spring 里被各 Controller 注入的单例 Bean。`Arc<T>` ≈ 线程安全的共享引用（这里内部多为原子/不可变，所以不需要 `Mutex`）。
- **`axum::Router` + `*_router()` 工厂**：每个能力模块导出一个 `xxx_router(state, path) -> (Vec<RouteDoc>, Router)`。`service_v2` 把 system 路由（health/metrics/models/busy）与 inference 路由（chat/cmpl/embed/...）分别 `.merge(...)` 再合并，并各自挂上 `TraceLayer` 中间件。axum router ≈ Spring MVC 的路由表，`.layer(...)` ≈ Servlet Filter / 拦截器；`echo_request_id_header` 中间件就是把请求头 `x-request-id` 回写到响应头。
- **`StateFlags`（`AtomicBool` 集合）**：每类 endpoint（Chat/Completion/Embedding/.../AnthropicMessages）一个原子开关，`enable_model_endpoint` 在运行期翻转。类比一组 `volatile boolean` 特性开关，配合模型发现实现 endpoint 的动态上下线。
- **`RouteDoc`**：`{ method, path }` 的小结构体，所有 router 顺手返回它，最后汇总喂给 `openapi_docs` 生成文档。可类比 Spring 启动时打印的 RequestMappingHandlerMapping 列表。

## 与其他模块的关系

- **依赖**：`axum` / `axum-server`（HTTP/TLS）、`tower-http`（`TraceLayer`）、`crate::discovery::ModelManager` 与 `Discovery`（模型挂载与实例发现）、`crate::kv_router::metrics`、`crate::request_template`、`dynamo_runtime`（`CancellationToken`、metrics registry、request span 等）。
- **被谁用**：`crate::entrypoint::input::http::run` 在 `Input::Http` 时构建并运行本服务；`crate::grpc`（gRPC 前端）大量复用本模块的 `Metrics` / `disconnect` / `service_v2` 等公共件——HTTP 与 gRPC 共享同一套指标与连接监控。
- **数据流**：handler 收到请求 → 经 `ModelManager` 找到对应模型引擎 → 走 `entrypoint::input::common` 装配的流水线 → 流式响应回传（非流式则聚合）。

## 阅读建议

1. 先看 `service.rs` 的模块清单和 `RouteDoc`，掌握模块版图。
2. 进 `service/service_v2.rs`：从 `struct HttpService` / `HttpServiceConfig`（约 201 / 216 行）和 `build` 里把各 `*_router` merge 成最终 router 的那段（约 520–592 行）看起，这是路由装配的总图。
3. 想看一个真实请求怎么处理，去 `service/openai.rs` 找 `chat_completions_router` 及其 handler（注意：这是个非常大的文件，按 router 名定位即可，不必通读）。
4. 横切能力按需点开：指标看 `metrics.rs`、健康检查看 `health.rs`、断连取消看 `disconnect.rs`。
