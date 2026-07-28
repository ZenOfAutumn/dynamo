<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Coding Trace 导出工具

用于隐私保护的代码代理（coding-agent）轨迹（trace）的 Rust 原生导出器。

## Claude 导出器

Claude 导出器位于 `dynamo-bench`，调用方式：

```bash
cargo run -p dynamo-bench --bin claude_trace_export -- \
  --output-file /tmp/claude_trace.jsonl
```

该命令会写入两个文件：

- Mooncake JSONL：基准测试行，包含可选字段 `session_id`、`input_length`、`output_length`、`hash_ids` 以及 `timestamp` 或 `delay`
- Sidecar JSONL：不含原文的结构化元数据，例如上下文形状、顶层工具调用，以及由 progress 推导出的嵌套时序信息

Sidecar 路径由输出路径派生：在扩展名前插入 `.sidecar`。

## 默认发现逻辑

如果省略 `--input-path`，导出器会：

1. 从当前工作目录开始。
2. 沿着祖先目录向上遍历直到 `/`。
3. 对每个祖先目录，检查 `~/.claude/projects/<encoded-absolute-path>` 下的对应编码 Claude 项目目录。
4. 同时扫描 home 级 Claude 根目录 `~/.claude/projects`。

发现过程中会忽略：

- `history.jsonl`
- `subagents/` 下的所有内容

## 可选输入覆盖

使用 `--input-path` 将导出范围限制为特定文件或目录：

```bash
cargo run -p dynamo-bench --bin claude_trace_export -- \
  --input-path ~/.claude/projects/<encoded-path> \
  --output-file /tmp/claude_trace.jsonl
```

`--input-path` 可指向：

- 某个 Claude 会话 JSONL 文件
- `~/.claude/projects` 下的某个编码 Claude 项目目录
- 其编码 Claude 项目目录应被使用的某个仓库根目录
- 包含 Claude 会话 JSONL 文件的目录

## 重要参数

```bash
cargo run -p dynamo-bench --bin claude_trace_export -- \
  --input-path ~/.claude/projects/<encoded-path> \
  --tokenizer deepseek-ai/DeepSeek-R1-Distill-Llama-8B \
  --block-size 64 \
  --delta-overlap-words 50 \
  --tokenizer-workers 8 \
  --output-file /tmp/claude_trace.jsonl
```

- `--anonymize-session-id`：用稳定的匿名化 ID 替换 Claude 会话 ID
- `--delta-overlap-words`：通过仅对前一个 prompt 的最后 `N` 个词加上新增 delta 重新分词来近似分词；默认 `50`
- `--tokenizer-workers`：用于按会话并行分词的 Worker 线程数

## 解析语义

导出器会：

- 使用顶层非 sidechain 的 `user`、`assistant`、`system` 行作为主对话流
- 按 `requestId`、再按 `message.id` 对相邻的 assistant 片段进行分组
- 排除 `thinking` 与 `redacted_thinking`
- 在 `compact_boundary` 处重置对话状态
- 压缩后的轮次从注入的 `isCompactSummary` 行开始
- 跳过本地命令包装器噪声，如 `<local-command-caveat>`、`<local-command-stdout>` 以及命令包装行
- 以哈希文本形式保留顶层工具调用（tool-use）与工具结果（tool-result）的结构
- 仅从 `progress` 行中挖掘不含原文的 sidecar 指标

## 输出语义

- 每个会话的第一轮带 `timestamp`。
- 后续轮次带 `delay`。
- 源 Claude 时间戳按 UTC 解析并归一化为毫秒级回放时间。
- 每条 Mooncake 行表示该 assistant 轮次的完整 prompt 前缀。
- 压缩后，未来的行从 compact summary 起算，而非从压缩前的原始历史构造。
- 行会随着会话间轮次合并被增量写出。
