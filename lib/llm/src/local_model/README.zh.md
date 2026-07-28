# 本地模型加载与注册（local_model）

> 把磁盘上（或从 Hugging Face 下载下来）的一个模型，包装成可被整个集群发现、调用的 `LocalModel` 实例。

## 这个模块解决什么问题

一个 worker 进程启动时，需要先确定"我要服务哪个模型、它在磁盘的哪里、配置参数是什么"，然后把这些元数据广播到集群（etcd / NATS），前端（frontend / ingress）才能发现并把请求路由过来。`local_model` 就负责这一段：解析模型路径、按需从 HF 下载、加载 `ModelDeploymentCard`（MDC，模型部署卡片）、命名、并通过 discovery 接口把模型注册（attach）到一个网络 endpoint 上。LoRA 适配器和基座模型都走这条注册路径。

## 目录结构

| 文件 / 子目录 | 职责 |
|---|---|
| `local_model.rs`（父模块，目录外） | 定义 `LocalModelBuilder`、`LocalModel`，负责构建、下载（`fetch`）、注册（`attach`）、注销（`detach_from_endpoint`）以及自托管元数据（`move_to_self_host`） |
| `runtime_config.rs` | 定义 `ModelRuntimeConfig` 与 `DisaggregatedEndpoint`，描述引擎运行期配置（KV block 数、最大并发序列、并行 rank、分离式部署 bootstrap 端点等） |

> 注意：模块入口 `local_model.rs` 位于本目录的上一级（`lib/llm/src/local_model.rs`），本目录只放它的子模块 `runtime_config`。

## 核心概念（Java 视角）

- **`LocalModelBuilder`**：典型的建造者模式（builder），等价于 Java 里链式 setter 的 `Builder`。每个方法 `&mut self -> &mut Self`，最后 `async fn build()` 产出 `LocalModel`。`build()` 返回 `anyhow::Result<LocalModel>`，相当于一个会抛受检异常的工厂方法，内部用 `?` 自动向上抛错。
- **`LocalModel`**：不可变的"已就绪模型"句柄。内部持有 `ModelDeploymentCard`（模型的全部元数据）、本地路径、`EndpointId`、HTTP/TLS 配置等。`#[derive(Clone)]` 让它可以低成本复制。
- **`ModelRuntimeConfig`**（`runtime_config.rs`）：一个可序列化的配置 POJO（`Serialize`/`Deserialize`，类似带 Jackson 注解的配置类）。其中带 `#[serde(default = "...")]` 的字段相当于反序列化时的默认值。它还实现了 `dynamo_kv_router::WorkerConfigLike` trait（≈ 实现了一个 router 侧需要的 interface），把并行 rank、KV block 总数等暴露给 KV router 做调度。
- **`runtime_data: HashMap<String, serde_json::Value>`**：引擎特定（vLLM / SGLang / TRT-LLM）的开放式配置袋，通过泛型方法 `set_engine_specific<T>` / `get_engine_specific<T>` 存取，类似 Java 里 `Map<String, Object>` + 泛型转换。
- **attach / detach 与自托管**：`attach` 通过 `endpoint.drt().discovery().register(spec)` 把 MDC 注册到服务发现；当开启 `DYN_SELF_HOST_METADATA` 时，`move_to_self_host` 会把磁盘上的元数据文件路径改写成 `http://<worker>/v1/metadata/<slug>/<suffix>/<file>` 这样的 URL，让其他节点能直接从本 worker 拉取，而不依赖共享文件系统。

## 与其他模块的关系

- 依赖 `crate::model_card::ModelDeploymentCard`（核心元数据载体）、`crate::model_type`（`ModelType` / `ModelInput`）、`crate::entrypoint::RouterConfig`、`crate::request_template::RequestTemplate`、`crate::preprocessor::media`（`MediaDecoder` / `MediaFetcher`）。
- 下载逻辑委托给 `crate::hub::from_hf`（`LocalModel::fetch`）。
- 依赖 `dynamo_runtime` 的 `Endpoint`、`discovery`（`DiscoverySpec` / `DiscoveryInstance`）、`metadata_registry`、`Slug` 等。
- `runtime_config.rs` 依赖 `crate::protocols::tensor::TensorModelConfig`，并实现 `dynamo_kv_router::WorkerConfigLike`。
- 被 `entrypoint`、`discovery::model_manager`、`http`/`grpc` service 层等使用（它们消费 `LocalModel` 或其产出的 MDC）。

## 阅读建议

1. 从 `local_model.rs` 的 `LocalModelBuilder::build()` 看起，理解一个模型从"路径/名字"到 `LocalModel` 的完整构建流程（含 frontend/echo 无路径的分支）。
2. 再看 `LocalModel::attach()`，理解模型如何注册进集群、LoRA 的 `model_suffix` 如何拼接。
3. 最后看 `runtime_config.rs` 的 `ModelRuntimeConfig`，对照各字段的 `serde` 默认值，理解 worker 向 router 暴露了哪些调度相关参数。
