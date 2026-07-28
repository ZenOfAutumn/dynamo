# 通用基础类型（common）

> 一组被整个 llm crate 复用的小而通用的基础类型：带校验和的文件引用、张量数据类型、版本号 trait。

## 这个模块解决什么问题

在分布式 LLM 推理里，很多东西需要跨进程/跨节点传递并保证一致性：模型文件要确认下载内容没被损坏、张量要知道元素是 FP16 还是 BF16、共享状态在 NATS 上更新时要做乐观并发控制。本模块把这些与具体业务无关的“工具型”类型集中放置，供 `block_manager`、`model_card`、`discovery` 等模块复用。模块入口为同名父文件 `common.rs`，只是简单地导出三个子模块。

## 目录结构

| 文件 / 子目录 | 职责 |
|---|---|
| `../common.rs` | 模块入口（父文件），导出 `checked_file`、`dtype`、`versioned` 三个子模块 |
| `checked_file.rs` | `CheckedFile`：带 BLAKE3 校验和与文件大小的文件引用，路径可以是本地磁盘或远程 URL，并自定义了 serde 序列化 |
| `dtype.rs` | `DType`：张量元素数据类型枚举（FP8/FP16/BF16/FP32 及各整型），可查每种类型占多少字节 |
| `versioned.rs` | `Versioned` trait：让对象带 revision 版本号，配合 NATS 实现原子更新 |

## 核心概念（Java 视角）

- `CheckedFile`：可类比一个不可变值对象，封装“文件路径 + 校验和 + 大小”。其内部路径字段类型是 `Either<PathBuf, Url>`——`Either` 类似一个只有两种取值的 sealed class（要么本地路径，要么 URL）。
  - `from_disk(path) -> anyhow::Result<Self>`：从磁盘读文件并算 BLAKE3 哈希；返回 `Result` 类比受检异常，调用处用 `?` 自动向上抛。
  - `checksum_matches(path) -> bool`：校验磁盘上某文件内容是否与记录的校验和一致，用于防篡改/防损坏。
  - 自定义 `Serialize`/`Deserialize`：序列化时 checksum 写成 `"blake3:<hash>"` 字符串；`size` 为 `Option<u64>` 以兼容旧版没有 size 字段的线格式（`Option` 类比可空值）。
- `DType`（enum）：带语义的数据类型枚举，类似 Java `enum`；`size_in_bytes()` 用 `match` 穷举每种类型返回字节数（`match` 类比 `switch`，但编译器强制覆盖所有分支）。
- `Versioned`（trait）：只有 `revision()` / `set_revision()` 两个方法的接口，类比 Java `interface`。NATS KV store 用它做乐观锁（revision 不一致则更新失败）。

## 与其他模块的关系

- 纯基础设施，几乎不依赖 crate 内其他业务模块，只依赖第三方库（`blake3`、`serde`、`url`、`either`、`anyhow`）。
- 被多处复用：`block_manager.rs` 通过 `pub use crate::common::dtype::DType` 重新导出 `DType`；`CheckedFile` 用于模型文件（如 `tokenizer.json`）的引用与校验；`Versioned` 用于需要在 NATS 上做原子更新的共享对象。

## 阅读建议

1. 先看 `dtype.rs`，最短、最直观，体会 Rust enum + `match` 的写法。
2. 再看 `versioned.rs`，理解一个极简 trait 的样子。
3. 最后读 `checked_file.rs`，重点看 `CheckedFile::from_disk`、`checksum_matches`，以及自定义 `Serialize`/`Deserialize` 如何兼容新旧线格式。
