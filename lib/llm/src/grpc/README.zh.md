# gRPC 服务（grpc）

> 用 tonic 实现 KServe v2 兼容的 gRPC 推理入口，是 HTTP 网关之外的另一条 ingress。

## 这个模块解决什么问题

很多推理生态（尤其 Triton / KServe 体系）以 gRPC 的 KServe v2 协议为标准接口。`grpc` 模块在 dynamo-llm 里提供这条入口：对外暴露 `GrpcInferenceService`（KServe 定义的 `ModelInfer` / `ModelStreamInfer` / `ModelMetadata` / `ModelConfig` 等 RPC），对内复用与 HTTP 网关完全相同的引擎、`ModelManager` 和 metrics。它和 `http` 模块是"同级别的两种前端"，由 `entrypoint::input::grpc` 负责拉起。

## 目录结构

| 文件 / 子目录 | 职责 |
|---|---|
| `grpc.rs`（同名模块入口，在上一级目录） | 仅 `pub mod service;`，把 service 子模块挂上去 |
| `service.rs` | 子模块聚合：`pub mod kserve / openai / tensor` |
| `service/kserve.rs` | 核心：`KserveService` + builder，`impl GrpcInferenceService`，通过 `tonic::include_proto!("inference")` 引入生成代码，用 `tonic` 起 server；State 与 HTTP 共享 `Metrics` |
| `service/openai.rs` | gRPC 侧的 OpenAI Completions 请求处理（`/v1/completions` 语义在 gRPC 上的对应实现，`completion_response_stream` 等） |
| `service/tensor.rs` | 张量（Tensor）输入/输出的转换：KServe 的 `ModelInfer` tensor 与 dynamo 的 `NvCreateTensorRequest/Response` 互转，`tensor_response_stream` 等 |
| `protos/kserve.proto` | KServe v2 推理协议定义（被 tonic 在编译期生成为 `inference` 模块） |
| `protos/model_config.proto` | 模型配置协议定义 |

## 核心概念（Java 视角）

- **`struct KserveService` + `KserveServiceConfigBuilder`**：用 `derive_builder` 生成 builder，链式 `.port(...).http_cancel_token(...).build()`，类比 Java 的 Builder 模式构造服务对象。`run(cancel_token)` 启动 tonic server。
- **`impl GrpcInferenceService for KserveService`**：`GrpcInferenceService` 是 proto 生成的 trait（≈ Java 由 protoc 生成的 service interface / gRPC base class），`impl ... for` 就是实现这个 service 接口的各个 RPC 方法。每个方法返回 `Result<Response<...>, Status>`，`Status` ≈ gRPC 的异常/错误码。
- **`tonic::include_proto!("inference")`**：编译期把 `.proto` 生成的 Rust 代码内联进来（`pub mod inference { ... }`），类比 Maven/Gradle 的 protobuf 插件在 `target/generated-sources` 里生成 Java 类，只不过这里是直接 include 进当前模块。
- **流式响应 `impl Stream`**：`ModelStreamInfer` 返回一个 `Stream`（异步迭代器），类比 Java gRPC 的 `StreamObserver` / Reactor `Flux`。底层 token 流经 `tensor_response_stream` / `completion_response_stream` 转换成 KServe 响应帧。
- **`GrpcTuningConfig`**：从环境变量 `DYN_GRPC_INITIAL_*_WINDOW_SIZE` 读取 HTTP/2 流控窗口大小，未设置则用 tonic 默认值，是一个简单的可选配置结构体。

## 与其他模块的关系

- **复用 HTTP 模块**：直接 `use crate::http::service::Metrics`、`http::service::service_v2`、`http::service::disconnect`、`http::service::metrics`（`InflightGuard` / `process_response_and_observe_metrics` 等）。即 gRPC 前端与 HTTP 前端共享同一套 metrics 与连接监控工具（源码里多处 `[gluo NOTE]` 注明这些是本应在前端间共享的公共工具）。
- **依赖**：`tonic` / `prost`（gRPC + protobuf 运行时）、`crate::discovery::ModelManager`、`crate::protocols::{tensor, openai::completions}`、`dynamo_runtime`（`AsyncEngine` 系列 trait、`Context`）。
- **被谁用**：`crate::entrypoint::input::grpc::run` 在 `Input::Grpc` 时构建 `KserveService` 并运行；`grpc/service/kserve.rs` 反过来又调用 `entrypoint::input::common` 装配引擎流水线。

## 阅读建议

1. 先扫一眼 `protos/kserve.proto`，了解 KServe v2 暴露哪些 RPC 和消息类型——这是整个模块的契约。
2. 进 `service/kserve.rs`：从 `struct KserveService`（约 147 行）和 `impl GrpcInferenceService for KserveService`（约 318 行）看起，理解 RPC 方法如何接到内部引擎。
3. 想搞清楚数据如何在 KServe tensor 与 dynamo 内部类型间转换，再看 `service/tensor.rs` 的 `tensor_response_stream`。
4. 对照 `crate::entrypoint::input::grpc::run` 看服务是怎么被装配、启动的。
