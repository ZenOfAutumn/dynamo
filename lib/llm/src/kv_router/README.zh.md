# KV 感知路由（kv_router）

> 把纯算法库 `dynamo-kv-router` 接到真实分布式运行时的集成层：根据各 worker 的 KV cache 命中情况和负载，挑选「最该把这个请求发给谁」的 worker。

## 这个模块解决什么问题

在多 worker 的 LLM 推理集群里，如果某个 worker 的 GPU 上已经缓存了请求前缀对应的 KV block，把请求发给它就能跳过大段 prefill。本模块负责这个决策：维护每个 worker 缓存了哪些 block 的索引（indexer）、估算各 worker 当前负载（scheduler/metrics），综合算出最优 worker；并把这套逻辑接上真实运行时——用 `Client`/`Endpoint` 做服务发现与 RPC、用 `AsyncEngine` 接请求流、用 NATS/ZMQ 的 publisher/subscriber 接 KV 事件流。

注意它「只选不发」：`KvRouter` 文档明确写着 *"only decides which worker you should use. It doesn't send you there."* 真正的转发由 `push_router` / 上层 pipeline 完成。

## 目录结构

| 文件 / 子目录 | 职责 |
|---|---|
| `../kv_router.rs` | 模块入口。核心类型 `KvRouter<Sel>`（持有 indexer + scheduler + client，对外提供 `find_best_match`），各种常量（NATS subject、endpoint 名）、re-export `dynamo_kv_router` 的 `approx`/`protocols`/`scheduling`/`selector` |
| `push_router.rs` | `KvPushRouter` / `DirectRoutingRouter`：在 `KvRouter` 选出 worker 后，真正通过 `Client` 把请求 push 过去，并用 drop-guard 管理请求全生命周期（prefill、首 token、输出 block、清理） |
| `scheduler.rs` | `KvScheduler<Sel>`：薄封装 `dynamo_kv_router` 的 `LocalScheduler`，按负载/命中做 worker 打分调度 |
| `metrics.rs` | 路由侧 Prometheus 指标（`WorkerLoadMetrics` 等），注册到 frontend 的 registry |
| `sequence.rs` | `RuntimeSequencePublisher` / `Subscriber`：把 `dynamo_kv_router` 里运行时无关的 `ActiveSequencesMultiWorker` 业务逻辑接到 NATS 事件传输 + Prometheus |
| `shared_cache.rs` | HiCache 共享 KV cache 客户端（SGLang + Mooncake）：直查 Mooncake master HTTP 服务判断外部缓存命中 |
| `sticky_sessions.rs` | `StickySessionRouter` + 可插拔的 affinity 存储（默认 `InMemoryAffinityStore`）：让同一多轮会话固定路由到同一 worker |
| `agent_controller.rs` | `AgentController`：subagent 的 KV 隔离会话生命周期（open/close session RPC） |
| `indexer/` | block 命中索引。`Indexer` enum 有 `KvIndexer` / `Concurrent`（本地 radix tree）/ `Remote`（远端索引服务）/ `None` 四种形态；含 `subscriber`（订阅 KV 事件）、`remote`、`jetstream`（从 NATS object store 拉 radix 快照）、`worker_query`（worker-local 查询端点） |
| `prefill_router/` | `PrefillRouter`：在 Migration 与 decode 路由之间，可选地先调一个 prefill worker（disaggregated serving），把 `disaggregated_params` 注入 decode 请求 |
| `publisher/` | worker 侧把本机 KV 事件（block 增删）发布出去的发布器（`KvEventPublisher`，含 ZMQ listener、去重、批处理）。**已有英文 `README.md` 与 `README.zh.md`，本总览不覆盖** |

## 核心概念（Java 视角）

- `struct KvRouter<Sel>`：本模块的门面（facade）。`Sel` 是泛型选择策略，约束为 `WorkerSelector` trait（≈ Java 接口 + 泛型 `<Sel extends WorkerSelector>`），默认 `DefaultWorkerSelector`。它聚合 `indexer`、`scheduler`、`client`，对外暴露 `async fn find_best_match`（≈ 返回 `CompletableFuture` 的方法）。
- `enum Indexer`（`indexer/mod.rs`）：带数据的枚举（≈ sealed class），四个变体代表四种索引后端。调用方用 `match` 处理，编译器强制覆盖每种情况——比 Java 用 `instanceof` 分发更安全。
- `KvScheduler<Sel>`（`scheduler.rs`）：只是 `dynamo_kv_router::scheduling::LocalScheduler` 的 `Arc` 薄封装。体现本模块的定位——**算法在底层 crate，这里只做运行时接线**。
- `trait AffinityStore`（`sticky_sessions.rs`）：会话亲和存储的抽象接口（≈ Java interface），默认实现用 `DashMap`（≈ 线程安全 Map）+ 后台清理 task，也可换成 Redis/etcd/NATS KV 支持多 router 部署。
- publisher / subscriber 模式：worker 侧 `publisher/` 把「我缓存/淘汰了哪些 block」发到 NATS/ZMQ，router 侧 `indexer/subscriber.rs` 订阅并更新本地索引。事件流类比一个跨进程的 `BlockingQueue`，生产者和消费者在不同机器上。

## 与其他模块的关系（重点：与 lib/kv-router 的边界）

- **`lib/kv-router`（crate `dynamo-kv-router`）= 纯算法**：radix tree、overlap 打分、调度策略、协议结构体（`RouterEvent`/`WorkerId`/...）、`WorkerSelector` trait 等，**不依赖任何分布式运行时**，可单独测试。
- **本模块（`lib/llm/src/kv_router`）= 集成/接线层**：依赖 `dynamo-kv-router` + `dynamo-runtime`，把上面的算法接到真实世界：
  - 用 `dynamo_runtime` 的 `Client`/`Endpoint`/`DiscoveryQuery` 做服务发现与 RPC；
  - 用 `AsyncEngine`/`ResponseStream` 接入请求-响应 pipeline；
  - 用 NATS（`EventPublisher`、JetStream object store）/ZMQ 承载 KV 事件与 radix 快照；
  - 用 Prometheus 暴露指标。
  - 入口文件直接 `pub use dynamo_kv_router::{approx, protocols, scheduling, selector}` 把算法类型 re-export 出去，调用方常分不清边界——记住：**带 `Client`/`Endpoint`/NATS/ZMQ 的在这里，纯数据结构与算法在 `lib/kv-router`。**
- **被谁用**：frontend / router 组件（如 `python -m dynamo.router`、disaggregated serving 的 decode 路由）通过 `KvRouter` + `KvPushRouter` 选 worker 并转发。

## 阅读建议

1. 从入口 `../kv_router.rs` 的 `pub struct KvRouter<Sel>` 与 `KvRouter::new(...)` 看起——`new` 里依次构造 `Indexer`、`KvScheduler`、启动事件订阅，是理解整个模块装配关系的主线。
2. 想知道「怎么选 worker」，跟进 `KvRouter::find_best_match`（同文件，结合 `cache_hit_estimates_from_tiered_matches` 等打分辅助函数）。
3. 想知道「选完怎么发」，看 `push_router.rs::KvPushRouter` 及其 drop-guard 生命周期管理。
4. 想搞清索引后端差异，看 `indexer/mod.rs::Indexer` 这个 enum 的四个变体，再按需下钻 `indexer/subscriber.rs`（事件订阅）与 `indexer/remote.rs`（远端索引）。
5. 反复出现的 `dynamo_kv_router::...` 前缀就是「这里在用底层算法 crate」的信号，对照 `lib/kv-router` 阅读。
