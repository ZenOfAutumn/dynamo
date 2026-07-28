# dynamo-mocker

`dynamo-mocker` 是一个无需 GPU 的仿真 crate，用于模拟 Dynamo 的 LLM 调度（scheduler）与 KV 缓存行为。
它适用于在不运行真实推理（inference）引擎的情况下，需要仿真出真实调度器与缓存行为的测试、回放和基准测试工作流。

## 本 crate 提供的内容

- `MockEngineArgs`：用于配置仿真引擎
- `engine::create_engine`：用于构建 vLLM 风格或 SGLang 风格的 mock 调度器
- `KvEventPublishers` 钩子：用于发出对路由（router）可见的 KV 缓存事件
- `loadgen` 与 `replay` 模块：用于合成与基于 trace 的实验

## 基本 Rust 用法

```rust
use dynamo_mocker::common::protocols::{
    DirectRequest, KvEventPublishers, MockEngineArgs,
};
use dynamo_mocker::engine::create_engine;

let args = MockEngineArgs::builder()
    .block_size(16)
    .num_gpu_blocks(1024)
    .max_num_seqs(Some(32))
    .max_num_batched_tokens(Some(4096))
    .build()
    .unwrap();

let engine = create_engine(args, 0, None, KvEventPublishers::default(), None);

engine.receive(DirectRequest {
    tokens: vec![1, 2, 3, 4],
    max_output_tokens: 16,
    uuid: None,
    dp_rank: 0,
    arrival_timestamp_ms: None,
});
```

本 crate 也是 Dynamo 上层 mocker CLI 与回放工具的基础。在多数部署中，你将通过 Python 入口间接使用它，而非直接将其作为独立的 Rust 依赖嵌入。

## 延伸阅读

- Mocker 指南：
  <https://github.com/ai-dynamo/dynamo/blob/main/docs/mocker/mocker.md>
- Trace 回放指南：
  <https://github.com/ai-dynamo/dynamo/blob/main/docs/benchmarks/mocker-trace-replay.md>
- Python 组件 README：
  <https://github.com/ai-dynamo/dynamo/blob/main/components/src/dynamo/mocker/README.md>
