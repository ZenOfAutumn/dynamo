# Agent Runtime Rust 模块

此目录包含 `dynamo-llm` 中面向 agent 的请求元数据与 trace 发射的 Rust 支持代码。

## 范围

- 尽量将 `mod.rs` 文件保持为模块装配与公共 re-export 表面，把实现重的逻辑放到具名模块中。
- 保持热路径请求处理精简。Agent trace 工作应受 trace 策略门控，并将有界记录发布到异步 trace 总线，而不是在内联中执行文件或网络 I/O。
- 将 agent trace schema 保留在 `trace/types.rs`，sink 行为在 `trace/sink.rs`，event-plane/ZMQ 中继行为在 `trace/relay.rs`，记录构造或校验在 `trace/record.rs`。
- 保留 trace 一致性模型：请求记录与工具记录共享同一个尽力交付（best-effort）的 trace 总线和 sink，但并非以事务方式提交。消费者必须按 `event_time_unix_ms` 排序，而不是 JSONL 行序。延迟到达的工具记录是可接受的；丢弃或缺失的记录仍可能发生，且不得被表述为审计级完整性。
- 受支持的外部工具事件入口路径是 harness ZMQ 中继。运行时 event-plane 跳是内部的；不要在同一变更中没有具体调用方时，新增独立的直接 event-plane 摄取模式。
- 除非同一变更中有具体的 Python 调用方需要，否则不要为 agent trace 内部暴露 Python 绑定。

## 验证

对 `trace/` 下的修改，运行：

```bash
cargo check -p dynamo-llm
cargo test -p dynamo-llm agents::trace --lib
```
