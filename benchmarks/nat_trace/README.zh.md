# NAT Trace 转换器

将 NeMo Agent Toolkit（NAT）profiler trace 转换为 mooncake 格式，用于
aiperf 基准测试。

## 概览

该转换器将 NAT profiler 输出的 `all_requests_profiler_traces.json` 转换为
mooncake 风格的 JSONL，可与 aiperf 一起用于：
- 通过 `session_id` 串行化的多轮对话回放
- 使用 `hash_ids` 的前缀缓存基准测试

## 获取 trace 数据

trace 由 [NeMo Agent Toolkit](https://github.com/NVIDIA/NeMo-Agent-Toolkit)
profiler 在运行评测时生成。详细的开启 profiling 的 agent workflow
运行方式见 [NAT 文档](https://docs.nvidia.com/nemo/agent-toolkit/latest/)。

将很快开源一个示例 trace 文件，让基准测试更易上手。

## 输入格式

NAT profiler trace JSON（`all_requests_profiler_traces.json`）：
```json
[
  {
    "request_number": 0,
    "intermediate_steps": [
      {
        "payload": {
          "event_type": "LLM_START",
          "metadata": {"chat_inputs": [{"content": "..."}]},
          "name": "llama-3.3-70b",
          "UUID": "..."
        }
      },
      {
        "payload": {
          "event_type": "LLM_END",
          "usage_info": {"token_usage": {"prompt_tokens": 9176, "completion_tokens": 142}},
          "UUID": "..."
        }
      }
    ]
  }
]
```

## 输出格式

带会话串行化的 mooncake JSONL：
```json
{"session_id": "conv_0", "input_length": 9176, "output_length": 142, "hash_ids": [1, 2, 3]}
{"session_id": "conv_0", "input_length": 9500, "output_length": 98, "hash_ids": [1, 2, 3, 4]}
{"session_id": "conv_1", "input_length": 8234, "output_length": 156, "hash_ids": [5, 6]}
```

## 用法

基础转换：
```bash
python convert.py --input-file /path/to/all_requests_profiler_traces.json
```

使用自定义 tokenizer：
```bash
python convert.py \
    --input-file /path/to/all_requests_profiler_traces.json \
    --tokenizer meta-llama/Llama-3.3-70B-Instruct \
    --block-size 128
```

限制请求数量：
```bash
python convert.py \
    --input-file /path/to/all_requests_profiler_traces.json \
    --num-requests 100 \
    --skip-requests 10
```

## 参数

| 参数 | 说明 | 默认值 |
|----------|-------------|---------|
| `--input-file` | NAT profiler JSON 路径 | 必填 |
| `--output-file` | 输出 JSONL 路径 | `<input>_mooncake.jsonl` |
| `--tokenizer` | HuggingFace tokenizer 名称 | 自动从 trace 推断 |
| `--block-size` | 用于 hash 生成的 block 大小 | 128 |
| `--num-requests` | 处理的最大请求数 | 全部 |
| `--skip-requests` | 跳过前 N 条请求 | 0 |

---

# Telemetry Trace 转换器

将 OpenAI 风格的 telemetry JSONL（例如来自 agentic 研究流水线）转换为
mooncake 格式，用于 aiperf 基准测试。

## 概览

该转换器将包含 `llm_call` 与 `tool_call` 事件的 `telemetry.jsonl` 转换为
mooncake 风格 JSONL。它会从 telemetry 中识别 6 种 agent 类型并相应标记
每条记录。

## 输入格式

每行一个事件的 telemetry JSONL：
```json
{"event_type": "llm_call", "timestamp": "...", "session_id": "...", "latency_priority": "HIGH", "latency_ms": 738.22, "request_payload": {"messages": [...], "model": "gpt-5.2"}, "response_payload": {"usage": {"prompt_tokens": 256, "completion_tokens": 4}}}
{"event_type": "tool_call", "tool_name": "tavily_web_search", "session_id": "...", "start_time": "...", "end_time": "...", "duration_ms": 181.08}
```

仅处理 `llm_call` 事件；`tool_call` 事件会被丢弃。

## 输出格式

带 agent 类型与优先级的 mooncake JSONL：
```json
{"session_id": "082e33c7-...", "agent_type": "deep_coordinator", "input_length": 2426, "output_length": 33, "hash_ids": [1, 2, 3], "priority": "HIGH"}
{"session_id": "082e33c7-...", "agent_type": "research_worker", "input_length": 4800, "output_length": 154, "hash_ids": [1, 2, 3, 4], "priority": "LOW"}
```

## Agent 类型

agent 通过系统 prompt 前缀匹配进行识别：

| Agent 类型 | 系统 Prompt 前缀 |
|---|---|
| `deep_coordinator` | `You are a Deep Research agent` |
| `research_worker` | `Gather and synthesize comprehe` |
| `research_planner` | `For the given task, generate a` |
| `shallow_agent` | `Current date and time:` |
| `classifier` | （无系统消息）—— 用户消息中含 "Classify" |
| `complexity_analyzer` | （无系统消息）—— 用户消息中含 "complexity analyzer" |

## 用法

基础转换：
```bash
python convert_telemetry.py --input-file /path/to/telemetry.jsonl
```

使用自定义 tokenizer：
```bash
python convert_telemetry.py \
    --input-file /path/to/telemetry.jsonl \
    --tokenizer deepseek-ai/DeepSeek-R1-Distill-Llama-8B \
    --block-size 128
```

## 参数

| 参数 | 说明 | 默认值 |
|---|---|---|
| `--input-file` | telemetry JSONL 路径 | 必填 |
| `--output-file` | 输出 JSONL 路径 | `<input>_mooncake.jsonl` |
| `--tokenizer` | HuggingFace tokenizer 名称 | `deepseek-ai/DeepSeek-R1-Distill-Llama-8B` |
| `--block-size` | 用于 hash 生成的 block 大小 | 64 |

---

## 配合 aiperf 运行

转换完成后，配合 aiperf 使用：
```bash
aiperf profile \
    --model <model-name> \
    --tokenizer <tokenizer> \
    --endpoint-type chat \
    --streaming \
    --url http://localhost:8000 \
    --input-file output_mooncake.jsonl \
    --custom-dataset-type mooncake_trace \
    --concurrency 10
```

`session_id` 字段确保：
- 同一对话内的多轮被串行化（按顺序执行）
- 不同对话在 `--concurrency` 限度内并行执行
