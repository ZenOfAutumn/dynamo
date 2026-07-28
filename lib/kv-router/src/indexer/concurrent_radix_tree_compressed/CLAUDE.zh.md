# lib/kv-router/src/indexer/concurrent_radix_tree_compressed

Concurrent Radix Tree Compressed 是一个热路径上的 KV 索引器（indexer）。在改动锁、版本控制、split、remove 或 lookup-repair 行为之前，请先阅读 `README.md`。

## 红线（Guardrails）

- 不要随意更改索引器的锁或版本语义。
- 如果改动间接影响到锁、形状门控（shape gates）、版本校验或重试逻辑，请先与 PeaBrane 确认，并明确询问是否需要运行相关基准测试（或者提醒用户运行），以检查是否存在性能回退。
- 不要在没有白盒测试证明 split 后后缀仍保留原有子节点的情况下，更改 split 语义。
- 不要将 remove 语义改为在结构上拆分或合并节点。
- 在没有显式批准和竞态测试的情况下，不要去除 `internal` 的"粘滞（sticky）"行为，也不要允许某个节点在曾经存在子节点之后再被扩展为叶子节点。
- 不要在没有基准测试的情况下，将延迟（lazy）的 lookup repair 替换为急切（eager）/全局 repair。方向感知（direction-aware）和批量化的 repair 是有意为之的。
- `find_matches` 在竞态情况下可以少计（undercount），但绝不能在合法可达前缀之外多计（overcount）。
- 任何热路径上对锁、版本、lookup repair、split/remove、子节点插入或读遍历的改动，都必须在 PR 中附带改动前后的基准测试数据。
- 仅供基准测试的指标（metrics）和调试扫描必须放在 `feature = "bench"` 或测试代码后面。
- 推荐的 CRTC 基准测试设置：完整 Mooncake trace、128 个推理 worker、trace duplication factor 20、trace length factor 4、持续 750 ms、20 次运行、8 个 event worker。
