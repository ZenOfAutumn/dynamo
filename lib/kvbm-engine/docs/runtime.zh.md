# 运行时（Runtime）

`KvbmRuntime` 是 KVBM 操作所需的复合共享基础设施。它打包了所有下游
管理器和服务都需要的最小组件集合：

- **Tokio runtime** —— 异步执行上下文（拥有的或借用的句柄）
- **Messenger（Velo）** —— 用于 leader/worker 通信和对端发现的分布式 RPC
- **NixlAgent** —— RDMA/UCX 数据传输（可选；当 NixL 配置缺失时禁用）
- **EventManager** —— worker 协调与传输完成通知（通过 Messenger 访问）

## 构造

两个快速构造器涵盖常见场景：

```rust,ignore
// Leader 角色（读取 KVBM_* 环境变量 + TOML 文件）
let runtime = KvbmRuntime::from_env_leader().await?;

// Worker 角色
let runtime = KvbmRuntime::from_env_worker().await?;
```

对于测试或自定义配置，使用 builder：

```rust,ignore
let config = KvbmConfig::from_env()?;
let runtime = KvbmRuntime::builder(config)
    .with_runtime_handle(Handle::current())   // 注入已存在的 tokio runtime
    .with_messenger(messenger)                // 注入预构建的 Messenger
    .with_nixl_agent(agent)                   // 注入预构建的 NixlAgent
    .build_leader()
    .await?;
```

`KvbmRuntimeBuilder::from_json(json)` 是 vLLM `kv_connector_extra_config`
字典的主要入口 —— JSON 值的优先级最高，会覆盖环境变量、TOML 文件和
默认值。

## 组件访问

| 方法              | 返回                          | 备注                                   |
|---------------------|------------------------------|----------------------------------------|
| `handle()` / `tokio()` | `tokio::runtime::Handle`  | 借用或拥有的 runtime 句柄              |
| `messenger()`       | `&Arc<Messenger>`            | Velo RPC                               |
| `nixl_agent()`      | `Option<&NixlAgent>`        | 当配置中禁用 NixL 时为 `None`          |
| `event_system()`    | `Arc<velo::EventManager>`   | 来自 Messenger，用于传输通知           |
| `config()`          | `&KvbmConfig`               | 完整配置快照                            |

## RuntimeHandle

`RuntimeHandle` 是一个枚举，抽象了拥有的（`Arc<Runtime>`）和借用的
（`Handle`）tokio runtime。当未注入时，builder 会按配置创建一个拥有的
runtime。
