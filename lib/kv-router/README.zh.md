# dynamo-kv-router

`dynamo-kv-router` 提供 Dynamo 进行 KV 感知路由所需的核心数据结构与调度原语，用于把请求引导到与之具有最佳 cache overlap 的 worker 上。

## 本 crate 提供的内容

- `RadixTree` 与 `ConcurrentRadixTree`：用于前缀重叠（prefix-overlap）索引
- `ThreadPoolIndexer` 与 `PositionalIndexer`：高吞吐的索引后端
- `KvRouterConfig`、`RouterQueuePolicy`、`LocalScheduler`：用于请求路由
- 协议与哈希辅助类型/函数，例如 `RouterEvent`、`WorkerId`、
  `compute_block_hash_for_seq`、`compute_seq_hash_for_block`

## Rust 基本用法

```rust
use dynamo_kv_router::{
    KvRouterConfig, RadixTree, compute_block_hash_for_seq, compute_seq_hash_for_block,
};
use dynamo_kv_router::protocols::BlockHashOptions;

let prompt_tokens = vec![1_u32, 2, 3, 4, 5, 6, 7, 8];
let local_hashes = compute_block_hash_for_seq(&prompt_tokens, 4, BlockHashOptions::default());
let seq_hashes = compute_seq_hash_for_block(&local_hashes);

let router_config = KvRouterConfig::default();
let index = RadixTree::new();
let scores = index.find_matches(local_hashes, false);

assert!(router_config.use_kv_events);
assert_eq!(seq_hashes.len(), 2);
assert!(scores.scores.is_empty());
```

要实现端到端路由，请将 indexer 与 `LocalScheduler` 以及从 crate 根重新导出的 worker/config 协议类型搭配使用。

## Features

- `metrics`：router 内部指标的 Prometheus 暴露
- `runtime-protocols`：与 `dynamo-runtime` 的集成点
- `standalone-indexer`：独立 indexer 服务支持
- `bench`：内部基准测试辅助

## 延伸阅读

- Router 指南：<https://docs.nvidia.com/dynamo/components/router>
- Indexer 内部实现：
  <https://github.com/ai-dynamo/dynamo/blob/main/lib/kv-router/src/indexer/README.md>
- 分片 KV indexer 的基准测试：[INDEXER_BENCH.md](../bench/kv_router/INDEXER_BENCH.md)
- Dynamo 仓库：<https://github.com/ai-dynamo/dynamo>

