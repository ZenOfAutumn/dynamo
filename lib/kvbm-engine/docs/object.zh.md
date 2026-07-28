# 对象存储模块

object 模块提供用于将 KV cache 块存储到对象存储系统（S3、MinIO）的 trait 与实现。它对应存储分层中的 G4（对象存储）层。

## ObjectBlockOps Trait

针对块级别对象存储操作的核心 trait：

| 方法 | 用途 |
|--------|---------|
| `has_blocks(keys)` | 检查块是否存在及其大小 |
| `put_blocks(keys, src_layout, block_ids)` | 通过逻辑布局句柄上传块 |
| `get_blocks(keys, dst_layout, block_ids)` | 通过逻辑布局句柄下载块 |
| `put_blocks_with_layout(keys, layout, block_ids)` | 使用已解析的物理布局上传 |
| `get_blocks_with_layout(keys, layout, block_ids)` | 使用已解析的物理布局下载 |

### 逻辑布局 vs 物理布局

该 trait 为 put/get 提供了两套 API：

- **逻辑** (`put_blocks` / `get_blocks`)：接收 `LogicalLayoutHandle`（G1、G2、G3）。worker 内部会将其解析为各自的物理布局。该 API 由 leader（不持有物理布局）以及 `CoordinatedWorker` 使用。
- **物理** (`put_blocks_with_layout` / `get_blocks_with_layout`)：直接接收已解析的 `PhysicalLayout`。由解析自身句柄后的 `PhysicalWorker` 以及实际执行 I/O 的 `S3ObjectBlockClient` 使用。

## Key 格式化

key 将 `SequenceHash` 值映射到对象存储路径：

- **`DefaultKeyFormatter`**：使用哈希的 Display 表示形式（例如 `0:abc123`）。适用于单 worker 场景。
- **`RankPrefixedKeyFormatter`**：以 worker rank 作为前缀（例如 `0/0:abc123`）。在 SPMD worker 场景下必需，因为多个 worker 会用不同的物理数据存储同一个逻辑块。

`create_key_formatter(rank)` 工厂函数会返回对应的 formatter。

## ObjectLockManager

用于协调式 offload 的分布式锁协议，避免重复上传：

```text
has_meta(hash)
  → true  → skip (already offloaded)
  → false → try_acquire_lock(hash)
              → true  → transfer → create_meta(hash) → release_lock(hash)
              → false → skip (another instance owns it)
```

锁获取使用条件 PUT (`If-None-Match: *`)，并通过基于截止时间的过期机制恢复陈旧锁。

## S3 实现

`s3` 子模块（通过 `s3` feature 开关启用）提供：

- **`S3ObjectBlockClient`**：为兼容 S3 的存储实现 `ObjectBlockOps`。通过 `rayon` 线程池支持并发上传/下载，并对对齐的块数据提供连续内存的快速路径。
- **`S3LockManager`**：使用 S3 条件写入实现 `ObjectLockManager`。

## 工厂函数

- **`create_object_client(config, rank)`**：根据配置创建 `Arc<dyn ObjectBlockOps>`。基于 `ObjectClientConfig` 选择后端（S3 或将来的其他实现）。
- **`create_lock_manager(config, instance_id)`**：创建用于分布式锁协调的 `Arc<dyn ObjectLockManager>`。
