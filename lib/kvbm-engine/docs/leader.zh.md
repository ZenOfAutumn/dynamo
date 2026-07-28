# Leader 模块

leader 模块为单个 KVBM instance 实现 block 协调。它拥有 block 元数据
（通过 `BlockManager<G2>` 与 `BlockManager<G3>`），处理缓存查找，并
编排在存储层级之间以及跨 instance 移动 block 的多阶段 onboarding 会话。

## Leader Trait

`Leader` trait 定义了核心协调接口：

```rust,ignore
pub trait Leader: Send + Sync {
    fn find_matches(&self, sequence_hashes: &[SequenceHash]) -> Result<FindMatchesResult>;
    fn find_matches_with_options(
        &self, sequence_hashes: &[SequenceHash], options: FindMatchesOptions,
    ) -> Result<FindMatchesResult>;
}
```

`find_matches` 搜索匹配给定 sequence hash 的 block，并根据 staging
模式与搜索范围返回立即结果或异步会话。

## InstanceLeader

`InstanceLeader` 是 `Leader` 的主要实现。它持有：
- 用于本地 block 注册的 `BlockManager<G2>` 与可选的 `BlockManager<G3>`
- 用于驱动传输执行的 `ParallelWorkers` 实例
- 处于活动状态的 onboarding 操作的会话状态
- 用于跨 instance 协调的远程 leader 连接

## FindMatchesResult

`find_matches` 的结果是以下两种之一：

- **`Ready`** —— 当 `search_remote == false` 且 `staging_mode == Hold`
  时返回。block 通过 RAII 原地保留，不创建会话。`ReadyResult` 直接
  拥有 `Vec<ImmutableBlock<G2>>`。

- **`AsyncSession`** —— 当需要远程搜索或 staging 时返回。包含一个
  `SessionId`、用于跟踪进度的 `watch::Receiver<OnboardingStatus>`，
  以及用于延迟控制的可选 `SessionHandle`。

## StagingMode

控制如何对匹配的 block 进行 staging 以及会话何时完成：

| 模式 | 行为 | 会话生命周期 |
|------|----------|-----------------|
| `Hold` | block 保留在原 instance 的当前层级（G2/G3） | 保持活动以支持后续操作 |
| `Prepare` | 在所有 instance 上执行 G3->G2 staging；不进行 RDMA 拉取 | staging 完成后保持活动 |
| `Full` | 各处 G3->G2，再将远程 G2 RDMA 拉取到本地 G2 | 当所有 block 都到达本地 G2 时完成 |

`Hold -> Prepare -> Full` 的渐进过程可以通过 `SessionHandle::prepare()`
与 `SessionHandle::pull()` 增量驱动。

## OnboardingStatus 状态机

```text
Searching
    |
    +---> Holding { local_g2, local_g3, remote_g2, remote_g3, pending_g4, ... }
    |         |
    |         +---> (prepare) ---> Preparing { matched, staging_local, staging_remote }
    |                                  |
    +---> Preparing ------------------>+
    |                                  |
    |                            Prepared { local_g2, remote_g2 }
    |                                  |
    |                                  +---> (pull) ---> Staging { matched, ..., pulling }
    |                                                        |
    +---> Staging ------------------------------------------>+
                                                             |
                                                        Complete { matched_blocks }
```

每个状态变体都携带用于进度跟踪和成本分析的计数器。`Holding` 包含 G4
加载跟踪（`pending_g4`、`loaded_g4`、`failed_g4`）。

## SessionHandle

`SessionHandle` 为 `Hold` 与 `Prepare` 会话提供延迟控制：

- `prepare()` —— 触发 G3->G2 staging（Hold -> Prepare 转换）
- `pull()` —— 触发将远程 G2 RDMA 拉取到本地 G2（Prepare -> Complete）
- `cancel()` —— 取消会话并释放所有保留的 block

不适用于 `StagingMode::Full`（其会自动运行至完成）。

## BlockAccessor

`BlockAccessor` 为基于策略的 block 扫描提供无状态、`Send + Sync` 的
接口。每次 `find()` 调用都会独立搜索 G2 然后 G3，并通过 RAII 获取
block。配套的 `PolicyContext` 通过 `yield_item()` 将扫描结果流式回传
给调用方。
