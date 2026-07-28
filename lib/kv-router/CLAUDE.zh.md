# lib/kv-router

KV-router 包含热路径路由（router）、索引、调度（scheduler）以及活跃序列状态。请将改动保持在小范围内，并阅读子目录中更具针对性的 `CLAUDE.md`（如果存在）。

## 哈希集合（Hash Collections）

- 对于内部数字键和热路径，尽可能使用 `FxHashMap` / `FxHashSet`。
- 不要将 `FxHashMap` / `FxHashSet` 用于文本键或外部可控的值（例如 `request_id`）；这些场景应使用标准哈希集合。
