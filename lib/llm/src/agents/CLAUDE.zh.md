# Agent 运行时 Rust 模块

本目录包含 `dynamo-llm` 中面向 agent 的请求元数据与 trace 发送相关的 Rust 支持代码。

## 范围

- 在可能的情况下，将 `mod.rs` 文件保持为模块连线与公共再导出（re-export）层，
  把实现密集的逻辑放到具名模块中。
- 保持热路径请求处理的精简。Agent trace 相关工作应当由 trace 策略门控，并将
  受限大小的记录发布到异步 trace 总线，而不是在内联流程中做文件或网络 I/O。
- 将 agent trace 的 schema 放在 `trace/types.rs`、sink 行为放在 `trace/sink.rs`、
  事件面 / ZMQ 中继行为放在 `trace/relay.rs`、record 构造或校验放在
  `trace/record.rs`。
- 保留 trace 的一致性模型：request 与 tool record 共享同一个尽力而为
  （best-effort）的 trace 总线与 sink，但并不进行事务性提交。消费方必须按
  `event_time_unix_ms` 排序，而不是按 JSONL 行序。延迟到达的 tool record 是
  可接受的；记录被丢弃或缺失仍然有可能发生，因此不能将其表述为审计级别的完整性。
- 把外部支持的 tool-event 入口路径限定在 harness ZMQ relay。运行时事件面这一跳
  属于内部细节；如果同一次改动里没有具体的调用方，请不要新增一种独立的、直接接入
  事件面的 ingestion 模式。
- 除非同一次改动里就有具体的 Python 调用方需要，否则不要为 agent trace 的内部
  实现暴露 Python 绑定。

## 验证

对 `trace/` 下的改动，请运行：

```bash
cargo check -p dynamo-llm
cargo test -p dynamo-llm agents::trace --lib
```
