# Session 模块

session 模块管理 instance 之间的分布式 block 传输会话。会话协调 KV 缓存
（KV cache）block 在请求 instance（Prefill，预填充）与服务 instance
（Decode，解码）之间的搜索、staging 与 RDMA 传输。

## 协议概览

### Onboard 协议（InitiatorSession ↔ ResponderSession）

使用 `OnboardMessage` 进行多对端搜索与 staging：

```text
  Initiator (Prefill)              Responder (Decode)
        │                                │
        │──── CreateSession ────────────▶│
        │                                │  搜索本地 G2/G3
        │◀─── G2Results ────────────────│
        │◀─── G3Results ────────────────│  （若发现 G3 block）
        │◀─── SearchComplete ───────────│
        │                                │
        │──── HoldBlocks ───────────────▶│
        │◀─── Acknowledged ─────────────│
        │                                │
        │──── StageBlocks ──────────────▶│  G3→G2 staging（可选）
        │◀─── BlocksReady ──────────────│
        │                                │
        │     RDMA pull（远程 G2→本地 G2）
        │                                │
        │──── CloseSession ─────────────▶│
```

当配置了 G4（对象存储）时，initiator 还会通过内部 `G4Results`/
`G4LoadComplete` 消息（不通过网络发送）并行执行 G4 搜索。

### 统一会话协议（SessionHandle ↔ ServerSession）

使用 `SessionMessage` 的点到点会话：

```text
  Controller (Prefill)             ServerSession (Decode)
        │                                │
        │──── Attach ───────────────────▶│
        │◀─── StateResponse ────────────│  （当前状态快照）
        │                                │
        │──── TriggerStaging ───────────▶│  （若有待处理的 G3 block）
        │◀─── BlocksStaged ────────────│  （新 staged 的 block）
        │                                │
        │     RDMA pull（远程 G2→本地 G2）
        │                                │
        │──── BlocksPulled ─────────────▶│  （释放已拉取的 block）
        │──── Detach ───────────────────▶│
```

控制权可通过 `YieldControl`/`AcquireControl` 双向转移。对于逐层
（layerwise）传输，`BlocksStaged` 包含可选的 `layer_range`。

## 会话类型

| 会话 | 角色 | 协议 | 说明 |
|---------|------|----------|-------------|
| **ServerSession** | 持有 block，对外暴露以供拉取 | SessionMessage | 由原 EndpointSession + ControllableSession 合并而来 |
| **SessionHandle** | 客户端控制 | SessionMessage | attach/detach、状态查询、RDMA 拉取 |
| **InitiatorSession** | 多对端搜索编排者 | OnboardMessage | 由 InstanceLeader 创建以执行分布式搜索 |
| **ResponderSession** | 响应搜索请求 | OnboardMessage | 搜索本地 G2/G3、保留 block、按需 staging |

### ServerSession

服务端会话，持有 block 并对外暴露以供远程 RDMA 拉取。支持两种模式：

- **G2-only**：block 已经在 G2 中并已分配布局句柄
  （`BlockMetadataMap::Direct`）。`TriggerStaging` 为空操作。通过
  `ServerSession::new_g2_only()` 或 `create_server_session()` 工厂创建。
- **Staging**：G3 block 需要 staging 到 G2。布局句柄按 round-robin
  方式分配给 worker（`BlockMetadataMap::RoundRobin`）。通过
  `ServerSessionOptions` 支持 `auto_stage` 选项。通过
  `ServerSession::new_with_staging()` 创建。

可从 `InstanceLeader` 通过以下方法创建：
- `create_endpoint_session()` / `create_endpoint_session_for_blocks()` —— G2-only
- `create_controllable_session()` / `create_controllable_session_with_options()` —— 带 staging

**ServerSessionHandle** 提供本地控制：用于逐层传输通知的
`notify_layers_ready()`，以及优雅关闭的 `close()`。

### InitiatorSession

请求方。向一个或多个远程 instance 发送 `CreateSession`，收集结果，
应用「最先响应胜出」的去重，并编排 staging 与 RDMA 拉取。支持三种
staging 模式：
- **Hold**：查找并持有 block（G2+G3），不进行 staging
- **Prepare**：在所有节点 G3→G2 staging，会话保持活动
- **Full**：G3→G2 staging + 远程 G2→本地 G2 RDMA 拉取，会话完成

当 `search_remote == true` 时由 `InstanceLeader::find_matches_with_options()`
创建。

### ResponderSession

服务方。接收 `CreateSession`，搜索本地 block 管理器（先 G2，剩余项再 G3），
通过 `BlockHolder` 持有匹配的 block，并返回匹配结果。处理 staging 请求
并保持 block 存活直到会话结束。

## 核心构建块

### BlockHolder

会话期间用于持有 block 的 RAII 容器。层级无关
（`BlockHolder<G2>`、`BlockHolder<G3>`）。当 holder 被 drop 时 block 自动释放，
即便会话处理出现 panic 也不会泄漏。关键操作：`retain()`、`release()`、
`extend()`、`take_all()`。

### SessionEndpoint

带状态机的点到点会话原语。封装：
- 身份（`session_id`、`instance_id`）
- 状态机（`ControlRole` + `AttachmentState` + `SessionPhase`）
- 消息接收通道（`mpsc::Receiver<SessionMessage>`）
- 通过 watch 通道的状态发布
- 用于向对端发送消息的传输

由 `ServerSession` 内部使用。**不**处理 block 持有或 staging 逻辑 ——
那是调用方的责任。

### SessionHandle

控制远程会话的句柄。支持：
- 状态观察：`current_state()`、`wait_for_ready()`、`wait_for_complete()`
- 控制命令：`trigger_staging()`、`mark_blocks_pulled()`、`detach()`
- 双向控制：`yield_control()`、`acquire_control()`
- RDMA 传输：`pull_blocks_rdma()`、`pull_blocks_rdma_with_options()`

由控制方（Prefill）使用以驱动远程 `ServerSession`（Decode）。

### SessionHandleStateTx

状态观察通道的发送端。由会话接收任务用于将 `StateResponse` 与
`BlocksStaged` 消息转发到 `SessionHandle` 观察的 watch 通道中。

### Staging

共享的 G3→G2 staging 逻辑被抽出到 `staging::stage_g3_to_g2()`。
核心 kernel：分配 G2 目的地 → 执行本地传输（G3→G2）→ 用源 sequence
hash 注册新 block。供 `InitiatorSession`、`ResponderSession` 与
`ServerSession` 共用以避免代码重复。

## 传输层

`MessageTransport` 是一个枚举，包含两种变体：

- **`VeloTransport`**：使用 Velo 主动消息（active messages）实现 instance
  间的分布式通信。
- **`LocalTransport`**：直接通过通道分发，用于进程内测试，无网络开销。

方法：
- `send()` —— 向目标 instance 发送 `OnboardMessage`
- `send_session()` —— 向目标 instance 发送 `SessionMessage`
- `request_metadata()` —— RPC 调用以获取远程 worker 用于 RDMA 的布局元数据

## 消息类型

| 类型 | 方向 | 用途 |
|------|-----------|---------|
| `OnboardMessage` | Initiator ↔ Responder | block 搜索、hold/drop、staging 请求 |
| `SessionMessage` | Controller ↔ ServerSession | attach/detach、控制权转移、block 操作、状态同步 |

### OnboardMessage 变体

| 变体 | 发送方 | 说明 |
|---------|--------|-------------|
| `CreateSession` | Initiator | 用 sequence hash 启动新会话 |
| `G2Results` | Responder | G2 搜索匹配（sequence hash + block ID） |
| `G3Results` | Responder | G3 搜索匹配（仅 sequence hash） |
| `SearchComplete` | Responder | 所有本地搜索完成 |
| `HoldBlocks` | Initiator | 哪些 block 要持有，哪些要 drop |
| `Acknowledged` | Responder | hold/drop 已处理 |
| `StageBlocks` | Initiator | 要 stage 到 G2 的 G3 hash |
| `BlocksReady` | Responder | 新 staged 的 G2 block 就绪 |
| `ReleaseBlocks` | Initiator | 释放特定 block |
| `CloseSession` | Initiator | 会话完成，进行清理 |
| `G4Results` | 内部 | 对象存储搜索结果（不通过网络发送） |
| `G4LoadComplete` | 内部 | 对象存储加载结果（不通过网络发送） |

### SessionMessage 变体

| 变体 | 类别 | 说明 |
|---------|----------|-------------|
| `Attach` | 连接 | 对端以某控制角色接入 |
| `Detach` | 连接 | 对端优雅断开 |
| `YieldControl` | 控制 | 发送方让出控制者角色 |
| `AcquireControl` | 控制 | 发送方获取控制者角色 |
| `TriggerStaging` | block 操作 | 请求 G3→G2 staging |
| `HoldBlocks` | block 操作 | 请求保留 block |
| `ReleaseBlocks` | block 操作 | 释放特定 block |
| `BlocksPulled` | block 操作 | 通知 block 已通过 RDMA 拉取完成 |
| `StateResponse` | 状态同步 | 完整状态快照（phase、role、block） |
| `BlocksStaged` | 状态同步 | 新 staged 的 block（可选 layer 范围） |
| `Close` | 生命周期 | 优雅关闭会话 |
| `Error` | 生命周期 | 报告错误 |

## 状态机

### SessionPhase

block 操作的生命周期。staging 是可选的 —— 已位于目标层（G2）的 block
会跳过：

```text
Searching → Holding ──────────────────── Ready → Complete
                    └── Staging ────────┘
                    └── Complete  （直接拉取，无需 staging）
                    └── Failed
```

### ControlRole

会话关系中的动态角色：
- `Neutral` —— 初始状态，可向任一方向转移
- `Controller` —— 向对端下达命令
- `Controllee` —— 执行来自对端的命令

通过 `YieldControl`/`AcquireControl` 支持双向转移。

### AttachmentState

对端连接状态：`Unattached`（等待中）或 `Attached { peer }`（已连接）。

## 分发函数

- **`dispatch_onboard_message`**：按 session ID 将 `OnboardMessage` 路由到
  各会话任务的通道。由 Velo onboard 处理器使用。
- **`dispatch_session_message`**：按 session ID 将 `SessionMessage` 路由到
  各会话任务的通道。由 Velo session 处理器使用。

## 文件结构

```text
session/
├── mod.rs              # 模块声明、分发函数、re-export
├── blocks.rs           # BlockHolder<T> —— RAII block 容器
├── endpoint.rs         # SessionEndpoint —— 状态机原语
├── handle.rs           # SessionHandle + SessionHandleStateTx
├── server_session.rs   # ServerSession + ServerSessionHandle + BlockMetadataMap
├── staging.rs          # 共享的 stage_g3_to_g2() 函数
├── state.rs            # SessionPhase、ControlRole、AttachmentState
├── messages.rs         # OnboardMessage、SessionMessage、BlockInfo 等
├── initiator.rs        # InitiatorSession（多对端编排者）
├── responder.rs        # ResponderSession（搜索 + 持有 + staging）
└── transport.rs        # MessageTransport（Velo + Local）
```
