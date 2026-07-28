---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Jail Stream
---

## 概述

`JailedStream` 是一个独立实现，用于在 token 流中处理"jail（禁锢）"检测。它提供基于 builder 模式的简洁 API：当检测到特定序列时累积 token，并在 jail 结束时将其作为单个 chunk 一次性释放。

## 主要特性

- **Builder 模式**：使用 builder 模式提供简洁的配置 API
- **可配置序列**：支持多组 jail 起止序列
- **Tool Call 解析**：内建工具调用（tool call）检测与解析
- **Stream 宏**：使用 `async-stream::stream!` 实现简洁的异步流
- **完全独立**：完全独立于已有代码
- **Annotations**：保留 annotation 以便观测

## 实现

### 位置
- 主实现：`lib/llm/src/protocols/openai/chat_completions/jail.rs`
- 示例：`lib/llm/src/protocols/openai/chat_completions/jail_example.rs`

### 用法

```rust
use crate::protocols::openai::chat_completions::jail::JailedStream;
use dynamo_runtime::engine::{AsyncEngineContextProvider, ResponseStream};

// 获取带 context 的 ResponseStream
let response_stream: Pin<Box<ResponseStream<_>>> = get_stream_from_engine();

// 在传入 apply 之前先把 context 抽出来
let context = response_stream.context();

// 应用 jail 转换（ResponseStream 实现了 Stream）
let jail = JailedStream::builder()
    .tool_call_parser("nemotron_deci")
    .build();

let jailed_stream = jail.apply(response_stream);

// 在交给 engine 消费前，把 context 重新包回去
let final_stream = ResponseStream::new(Box::pin(jailed_stream), context);
```

### 高级配置

```rust
// 使用自定义 jail 序列
let jail = JailedStream::builder()
    .jail_start_sequence("<TOOLCALL>")
    .jail_end_sequence("</TOOLCALL>")
    .tool_call_parser("nemotron_deci")
    .build();

// 使用多组序列
let jail = JailedStream::builder()
    .jail_start_sequences(vec!["<TOOLCALL>", "<FUNCTION>"])
    .jail_end_sequences(vec!["</TOOLCALL>", "</FUNCTION>"])
    .tool_call_parser("harmony")
    .build();
```

## 工作原理

1. **检测**：当检测到 jail 起始序列（或工具调用起始）时，流进入"jail"模式
2. **累积**：在 jail 期间，token 被累积在内存中而不直接 yield
3. **Annotations**：向下游发送带 annotation 的空 chunk，便于观测
4. **释放**：当检测到 jail 结束序列，或流自身结束时：
   - 累积的内容被解析为 tool call
   - yield 出一个包含解析结果的 chunk
5. **透传**：未进入 jail 的内容原样透传

## 测试

该实现包含完整的测试用例：

- `test_jailed_stream_with_start_end_sequences`：测试显式 jail 序列
- `test_jailed_stream_with_tool_calls`：测试工具调用检测与解析
- `test_jailed_stream_no_jailing`：测试普通透传行为

运行测试：
```bash
cargo test -p dynamo-llm jail --lib
```

## 优势

1. **独立**：无需修改现有代码
2. **API 简洁**：builder 模式使配置更直观
3. **灵活**：支持多种 jail 检测策略
4. **可维护**：使用 `stream!` 宏让异步代码更清晰
5. **易测试**：完善的测试套件并复用工具函数
6. **高效**：库内不做不必要的 boxing 与 context 处理
7. **可组合**：可在重新加回 context 之前串联多个流变换

## 性能优化

- **库内不做 Boxing**：返回 `impl Stream` 而非 `Pin<Box<ResponseStream>>`
- **栈上 Pin**：使用 `tokio::pin!()` 替代 `Box::pin()`，性能更优
- **无 Context 开销**：JailedStream 不管理 AsyncEngineContext
- **惰性求值**：仅处理必要的内容
- **高效状态管理**：仅在进入 jail 状态时才进行最小限度的 clone

## 集成方案

替换已有的 `apply_tool_calling_jail_internal` 函数：

```rust
// preprocessor.rs 中
pub fn apply_tool_calling_jail_with_parser(
    &self,
    stream: ManyOut<Annotated<NvCreateChatCompletionStreamResponse>>,
) -> ManyOut<Annotated<NvCreateChatCompletionStreamResponse>> {
    let jail = JailedStream::builder()
        .tool_call_parser(self.tool_call_parser.clone())
        .build();

    jail.apply(stream)
}
```

## 未来增强

- 支持 jail 序列使用正则表达式
- 为 jail 检测添加 metrics/telemetry
- 支持跨 chunk 边界的部分序列匹配
- 可配置的累积上限
- 支持嵌套 jail

