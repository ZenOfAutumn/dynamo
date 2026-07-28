# 服务发现与模型注册表（discovery）

> 监听 dynamo discovery 平面上 worker 的上线/下线，动态维护“模型名 → 可服务的 worker 集合 → 推理引擎”的注册表，并跟踪各 worker 负载。

## 这个模块解决什么问题

dynamo 是分布式框架：后端 worker 会动态启动/停止，每个 worker 启动时会把自己的 `ModelDeploymentCard`（模型部署卡）注册到 discovery 平面。前端（HTTP/gRPC）需要实时知道“现在哪些模型可用、每个模型由哪些 worker 提供、该把请求路由到谁”。本模块就是这套动态注册表：它订阅 discovery 事件流，按模型名和 namespace 组织 worker，为每个模型构建好可直接调用的推理引擎（含 KV router / prefill router），并持续监控 worker 负载供路由决策使用。模块入口为同名父文件 `discovery.rs`，集中导出各公开类型。

## 目录结构

| 文件 / 子目录 | 职责 |
|---|---|
| `../discovery.rs` | 模块入口（父文件），导出 `Model`、`ModelManager`、`WorkerSet`、`ModelWatcher`、`KvWorkerMonitor` 等 |
| `model.rs` | `Model`：一个具名模型（如 "llama-3-70b"），背后挂一到多个 `WorkerSet`；按 worker 数量加权随机选择 WorkerSet |
| `worker_set.rs` | `WorkerSet`：同一 namespace/同一份配置部署出来的一组 worker，自带完整 pipeline（引擎、KV router、prefill router） |
| `model_manager.rs` | `ModelManager`：全局注册表，维护 模型名→`Model`、实例→card 的映射，并管理 prefill router 激活与各 endpoint 的运行时配置 watcher；错误类型 `ModelManagerError` |
| `watcher.rs` | `ModelWatcher`：订阅 discovery 事件流，把 worker 的上线/下线翻译成对 `ModelManager` 的增删；对外发出 `ModelUpdate`（`Added`/`Removed`）通知 |
| `worker_monitor.rs` | `KvWorkerMonitor`：订阅各 worker 的 KV/负载指标，维护 `WorkerLoadState`，提供 `LoadThresholdConfig` 与 prefill/decode worker 类型常量 |
| `endpoint_card.rs` | `wait_for_endpoint_model_card`：事件驱动地等待某 endpoint 上的 worker 发布其 `ModelDeploymentCard` |
| `runtime_configs.rs` | `runtime_config_watch`：把“实例可用性”和“运行时配置”两路发现 join 成单一 `watch`，只保留两者都齐备的 worker |

## 核心概念（Java 视角）

- `ModelManager`：类比一个线程安全的注册表单例。内部用 `DashMap`（线程安全并发 map，类似 `ConcurrentHashMap`）存放 `Arc<Model>`，无需自己加锁。
- `Model` / `WorkerSet`：两层组织结构。一个 `Model` 下可有多个 `WorkerSet`（每个对应一个 namespace），请求按 worker 数量做加权随机分发——类比按权重挑选下游服务实例的负载均衡。
- `ModelUpdate`（enum：`Added(card)` / `Removed(card)`）：带数据的枚举，类比只有两个子类的 sealed class，每个变体携带一张 `ModelDeploymentCard`。
- `ModelManagerError`（enum）：业务错误类型（`ModelNotFound` / `ModelUnavailable` / `ModelAlreadyExists`），配合 `Result<T, E>` 使用，相当于受检异常；`?` 自动向上抛。
- `ModelWatcher`：事件驱动的后台监听器。它消费 discovery 的 `DiscoveryStream`，可通过 `mpsc::Sender<ModelUpdate>`（类比 `BlockingQueue`）把变更推给订阅方；整个监听是 `async` 的，类比跑在虚拟线程里的事件循环。
- `RuntimeConfigWatch = watch::Receiver<...>`：`watch` channel 始终只保留“最新快照”，类比一个可订阅的 `AtomicReference`——读到的永远是最新值，旧值会被覆盖。

## 与其他模块的关系

- 依赖 `dynamo_runtime` 的 discovery API（`DiscoveryEvent`/`DiscoveryQuery`/`DiscoveryStream`、watch 工具）作为事件来源，依赖其 pipeline（`PushRouter`、`ServiceBackend` 等）来构建可调用引擎。
- 依赖 `crate::model_card::ModelDeploymentCard` 作为 worker 自描述、`crate::kv_router`（`KvRouter`/`PrefillRouter`/负载估计器）做 KV 亲和路由、`crate::backend::Backend` 装配后端、`crate::types::openai::*` 作为各类引擎类型。
- 被前端（HTTP/gRPC 服务）使用：前端持有 `ModelManager` 查询可用模型并取得引擎来处理请求；`worker_monitor` 的负载数据反哺路由决策。

## 阅读建议

1. 先读父文件 `../discovery.rs`，看清对外导出的几个核心类型及其归属。
2. 再读 `model.rs`（`Model` 如何挑 `WorkerSet`）和 `worker_set.rs`（一个 WorkerSet 持有哪些 pipeline 组件），建立数据模型。
3. 然后读 `watcher.rs` 的 `ModelWatcher` 与 `ModelUpdate`，理解“discovery 事件 → 注册表增删”的主循环。
4. 想看注册表细节看 `model_manager.rs`（`ModelManager` + `ModelManagerError`）；想看负载/路由相关看 `worker_monitor.rs`。
