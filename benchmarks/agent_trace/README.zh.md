# Agent Trace 工具

用于处理由 `DYN_AGENT_TRACE_SINKS=jsonl` 或 `jsonl_gz` 输出的 Dynamo agent trace
文件的工具集。

## 转换为 Perfetto

```bash
python3 benchmarks/agent_trace/convert_to_perfetto.py \
  "/tmp/dynamo-agent-trace.*.jsonl.gz" \
  --output /tmp/dynamo-agent-trace.perfetto.json
```

在 [Perfetto UI](https://ui.perfetto.dev/) 中打开输出 JSON。

输入可以是 `.jsonl`、`.jsonl.gz`、包含 trace 分片的目录或一个 glob 模式。转换器
会输出 Chrome Trace Event JSON：

- 每个 session 对应一个 Perfetto 进程
- 每条轨迹（trajectory）通道对应一个 Perfetto 线程
- 每个 Dynamo `request_end` 对应一个 LLM 请求切片
- 默认情况下，prefill wait、prefill 与 decode 阶段切片堆叠在请求之下
- 每个 harness 的 `tool_end`/`tool_error` 对应一个 tool 切片；优先使用显式的
  `started_at_unix_ms`/`ended_at_unix_ms`，其次使用 `duration_ms`，再次使用同时
  存在 `tool_start` 记录时配对的 timing
- 通过 `--include-markers` 可选地启用首 token 标记

使用 `--no-stages` 获得仅请求级的紧凑视图。使用 `--separate-stage-tracks` 在
调试 Perfetto 嵌套或标签渲染时，将各阶段切片放置到相邻的阶段轨道上。

阶段切片边界会被规范化，以避免由独立指标取整造成的同线程重叠。原始 timing 字段
仍可在事件 args 中获取。

## 转换为 Mooncake Replay

使用 Dynamo agent tracing 捕获的 trace 默认包含 replay 哈希，可被转换为 Mooncake
JSONL 用于 Dynamo 的 replay/mocker 路径。仅当需要抑制 replay 元数据时，才设置
`DYN_AGENT_TRACE_REPLAY_HASHES=0`：

```bash
cargo run -p dynamo-bench --bin agent_trace_to_mooncake -- \
  --input-path /tmp/dynamo-agent-trace.*.jsonl.gz \
  --output-file /tmp/dynamo-agent-trace.mooncake.jsonl
```

转换器接受 `.jsonl`、`.jsonl.gz`、可重复的 `--input-path` 参数，以及形如
`{"timestamp": ..., "event": ...}` 的 recorder-envelope 记录。每个 Dynamo
`request_end` 会输出一行独立的 Mooncake 请求记录，其 `timestamp` 为 Dynamo 请求
到达时间的绝对值，且不带 `session_id`。在转换过程中，稳定的 trace
`input_sequence_hashes` 会被压缩为 Mooncake 的 `hash_ids`。

关于当前 replay 已覆盖的范围以及路线图上仍待完成的部分（缓存移动保真度、输出
token 重建、因果性的工具/轮次依赖、端到端的 agent 重跑），参见
[Replay Scope and Follow-ups](../../docs/agents/agent-tracing.md#replay-scope-and-follow-ups)。

回放（replay）输出时，使用与 trace 捕获时相同的 trace block size。转换器在写出
Mooncake JSONL 后会打印此值。如果希望 replay 哈希粒度与在线后端的 page size
对齐，请将该值用作 mock 引擎的 block size。

```bash
TRACE_BLOCK_SIZE=128
uv run --no-sync python -m dynamo.replay /tmp/dynamo-agent-trace.mooncake.jsonl \
  --trace-format mooncake \
  --trace-block-size "${TRACE_BLOCK_SIZE}" \
  --replay-mode offline \
  --router-mode kv_router \
  --num-workers 4 \
  --extra-engine-args "{\"block_size\":${TRACE_BLOCK_SIZE}}" \
  --report-json /tmp/dynamo-agent-trace.replay-report.json
```

`kv_router` 需要多个 mock worker。如需进行单一聚合 worker 的健全性检查，使用
`--router-mode round_robin --num-workers 1`。

## 校验转换器

转换器有一个本地自检脚本，故意未接入主 pytest 套件：

```bash
python3 benchmarks/agent_trace/validate_convert_to_perfetto.py
```
