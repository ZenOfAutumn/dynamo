# 通用工具（utils）

> LLM crate 内部的零散通用工具：LoRA 名字到 ID 的哈希、流式文本里的标记（marker）检测、ZMQ socket 封装。

## 这个模块解决什么问题

这是个「杂物抽屉」式的模块，收纳几个跨功能复用、但又不属于任何具体业务模块的小工具。三者互不相关，只是都被 LLM 链路的不同地方用到：

- **LoRA ID 派生**：把 LoRA 适配器名字稳定地映射成一个数值 ID。
- **流式标记检测**：在按 chunk 到达的文本流里识别标记串（如 reasoning / 工具调用的特殊 token），并正确处理跨 chunk 边界被切断的情况。
- **ZMQ 封装**：为 KV 事件等场景提供统一参数的 ZeroMQ socket 构造。

## 目录结构

| 文件 | 职责 |
|---|---|
| `mod.rs` | 模块入口。声明三个子模块，并 re-export `lora_name_to_id`、`MarkerMatcher`、`MatchResult` |
| `lora.rs` | `lora_name_to_id`：用 blake3 哈希把 LoRA 名字转成确定性的非零 signed int32 ID |
| `prefix_matcher.rs` | `MarkerMatcher` / `MatchResult`：基于 aho-corasick 的多模式标记检测，支持跨 chunk 的部分后缀匹配 |
| `zmq.rs` | ZMQ socket 类型别名与统一参数的构造 helper（pub/sub、pull、router） |

## 核心概念（Java 视角）

- **`lora_name_to_id(name) -> i32`**（`lora.rs`）：纯函数，≈ Java 里用 `MessageDigest` 算哈希再折叠成 int 的工具方法。取 blake3 前 8 字节、屏蔽符号位、保证非零，确保同名永远得到同一 ID（确定性）。
- **`MatchResult` enum**（`prefix_matcher.rs`）：带数据的 enum ≈ sealed class。区分「完整匹配到标记」`Complete`（含前缀/标记/位置/后缀）与「chunk 末尾出现部分匹配」`Partial`（需要保留等待下个 chunk），这是流式解析的关键状态。
- **`MarkerMatcher`**（`prefix_matcher.rs`）：基于 `aho_corasick` 的多模式匹配器 ≈ 预编译好的多关键词搜索器（类似一次性匹配多个正则但更快）。被工具调用 jail（`protocols/openai/chat_completions/jail.rs`）用来在输出流里定位特殊标记。
- **ZMQ socket 别名 + `configure_common_builder`**（`zmq.rs`）：`SharedPubSocket = Arc<Mutex<Publish>>` ≈ 线程安全共享的发布 socket（`Arc<Mutex<T>>` 就是「可多处持有 + 互斥访问」的共享对象）。helper 统一设置超时、重连、keepalive、linger 等参数。

## 与其他模块的关系

- `MarkerMatcher` 被 `protocols/openai/chat_completions/jail.rs`（工具调用流式拦截）使用。
- `zmq.rs` 的 socket 工具被 `crate::kv_router::publisher::zmq_listener`、`crate::fpm_publisher`、`crate::mocker`、`crate::block_manager::kv_consolidator` 等 KV 事件相关路径使用。
- `lora_name_to_id` 服务于 `crate::lora` / 预处理器中 LoRA 适配器的 ID 计算。
- 依赖外部 crate：`blake3`（哈希）、`aho-corasick`（多模式匹配）、`tmq` + `tokio`（异步 ZMQ）。

## 阅读建议

1. 先读 `mod.rs`，看清楚导出了哪三样东西。
2. 从最简单的 `lora.rs` 入手熟悉本模块风格。
3. 重点读 `prefix_matcher.rs` 的 `MatchResult` 与 `MarkerMatcher`——流式解析里 `Partial` 状态的处理是难点。
4. 需要时再看 `zmq.rs` 的类型别名和参数常量。
