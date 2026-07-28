<!-- # SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0 -->

# Router Benchmark 指南

本目录包含用于对带前缀缓存（prefix caching）的 Dynamo router 进行 benchmark 的脚本。Benchmark 衡量跨请求前缀共享带来的性能提升。

## 前置条件

- NVIDIA GPU（默认配置需要 8 张 GPU）
- （可选）H100 或更新的 GPU，用于 gpt-oss-120b 示例
- 已正确配置的 CUDA 环境
- 正在运行的 etcd 与 NATS（Dynamo 协调所需）
- 必备的 Python 包：
  - `dynamo` 包（含 vllm 与 frontend 模块）
  - 用于 benchmark 的 `aiperf`
  - 用于绘图的 `matplotlib`
  - `data-generator` 包（在 repo 根目录执行 `pip install -e ./benchmarks` 安装）

> [!Note]
> 在容器外运行时，请把 `DYNAMO_HOME` 设置为 Dynamo 仓库的根路径：
> ```bash
> export DYNAMO_HOME=/path/to/dynamo
> ```
> 在容器内运行时，默认值是 `/workspace`。

### 搭建 etcd 与 NATS

本 benchmark 需要 etcd 与 NATS。快速启动：

```bash
# From the repository root:
docker compose -f deploy/docker-compose.yml up -d
```

这会以所需配置在后台同时启动 etcd 与 NATS。

## 脚本概览

- **`run_engines.sh`** —— 启动多个 vLLM worker 实例
- **`ping.sh`** —— 简单测试脚本，用于验证环境是否搭建好
- **`prefix_ratio_benchmark.py`** —— 主 benchmark 脚本，扫描多个 prefix ratio
- **`real_data_benchmark.py`** —— 使用真实 Mooncake 风格 trace 数据的 benchmark 脚本
- **`agent_benchmark.py`** —— 面向多轮对话 trace 的并发驱动 benchmark
- **`mock_server.py`** —— 简单的 mock server，用于接收并记录 aiperf 发来的请求

## 使用说明

### 第 1 步：启动 worker

确保你有 8 张 GPU 可用（除非你用 mocker，见下文）。先在终端中启动 worker 引擎。

脚本支持三种模式：
- **`agg`（默认）**：聚合（aggregated）/单体 worker，同时处理预填充（prefill）与解码（decode）
- **`decode`**：仅负责 decode（token 生成）阶段的 worker
- **`prefill`**：仅负责 prefill（提示处理）阶段的 worker

```bash
# Default: 8 aggregated workers with DeepSeek model (handles both prefill and decode)
./run_engines.sh \
    --num-workers 8 \
    --model-path deepseek-ai/DeepSeek-R1-Distill-Llama-8B

# Example: 4 workers with larger model using tensor parallelism (2 GPUs per worker)
# NOTE: this requires having Hopper or later GPU SKUs to support MXFP4 precision.
./run_engines.sh \
    --num-workers 4 \
    --model-path openai/gpt-oss-120b \
    --tensor-parallel-size 2
```

#### 解耦服务（decode + prefill worker）

你可以为解耦服务分别启动 decode 与 prefill worker。这允许把特定 GPU 专门用于 prefill（提示处理）或 decode（token 生成）：

```bash
# Launch 4 decode workers (GPUs 0-3)
./run_engines.sh \
    --decode \
    --num-workers 4 \
    --model-path deepseek-ai/DeepSeek-R1-Distill-Llama-8B

# Launch 4 prefill workers (GPUs 4-7)
./run_engines.sh \
    --prefill \
    --num-workers 4 \
    --base-gpu-offset 4 \
    --model-path deepseek-ai/DeepSeek-R1-Distill-Llama-8B
```

#### 备选：启动 vLLM mocker worker

我们也支持运行轻量级的 mocker 引擎，它模拟 vLLM 行为但不真正进行模型推理。Mocker 引擎适合在没有 GPU 时测试 router 逻辑与性能。使用 `--mockers` 标志可以以 mocker 引擎替代真实 vLLM worker。

```bash
# Example: Running mocker engines for testing (no GPU required)
./run_engines.sh --mockers \
    --num-workers 8 \
    --model-path deepseek-ai/DeepSeek-R1-Distill-Llama-8B \
    --block-size 64 \
    --speedup-ratio 2.0
```

**注意**：`--speedup-ratio` 控制 mocker 引擎模拟的推理速度。值越高（如 2.0），mocker 模拟的推理越快，benchmark 也能更快完成。这对在不等待真实推理时间的情况下测试 router 性能尤其有用。

#### Mocker 下的解耦服务（无需 GPU）

可以完全用 mocker 测试解耦服务：启动共享同一 namespace 的独立 prefill 与 decode mocker 组。这对在不需要任何 GPU 的情况下验证路由逻辑、指标以及 prefill-decode 交接非常有用。

```bash
NAMESPACE="test-disagg"
MODEL="Qwen/Qwen3-0.6B"

# Terminal 1: Decode mockers (2 workers)
python -m dynamo.mocker --model-path "$MODEL" \
    --endpoint "dyn://${NAMESPACE}.backend.generate" \
    --disaggregation-mode decode --num-workers 2 \
    --speedup-ratio 10 --block-size 16

# Terminal 2: Prefill mockers (2 workers)
python -m dynamo.mocker --model-path "$MODEL" \
    --endpoint "dyn://${NAMESPACE}.prefill.generate" \
    --disaggregation-mode prefill --num-workers 2 \
    --speedup-ratio 10 --block-size 16

# Terminal 3: Frontend with KV router
# --model-path must be the on-disk snapshot directory
MODEL_PATH=$(find ~/.cache/huggingface/hub/models--Qwen--Qwen3-0.6B/snapshots -mindepth 1 -maxdepth 1 -type d | head -1)
python -m dynamo.frontend --namespace "$NAMESPACE" \
    --model-name "$MODEL" --model-path "$MODEL_PATH" \
    --router-mode kv --http-port 8000 --kv-cache-block-size 16
```

验证：
```bash
# Send a request (should show prefill_worker_id and decode_worker_id in nvext)
curl -s localhost:8000/v1/chat/completions -H "Content-Type: application/json" \
    -d '{"model":"Qwen/Qwen3-0.6B","messages":[{"role":"user","content":"Hello"}],"max_tokens":10}' | python3 -m json.tool

# Check router metrics
curl -s localhost:8000/metrics | grep "^# HELP dynamo_component_router"
```

### 第 2 步：启动 router

在**新终端**中通过 Python CLI 启动 Dynamo router：

```bash
# Explicitly set NATS server for KV event publishing
export NATS_SERVER="${NATS_SERVER:-nats://localhost:4222}"

python -m dynamo.frontend \
    --router-mode kv \
    --router-reset-states \
    --http-port 8000
```

这会以以下方式启动 router：
- KV 缓存路由模式
- `--router-reset-states` 标志清除上次运行留下的事件缓存（JetStream），适合单 router benchmark
- HTTP 端口 8000

查看所有 router 参数：
```bash
python -m dynamo.frontend --help
```

router 参数（尤其是 KV 缓存路由参数）的详细解释，参见 [Router Guide](../../docs/components/router/router-guide.md)。

> [!Note]
> 如果不确定后端引擎能否正确发出 KV 事件（例如 gpt-oss、nemotron nano 2 这类混合模型），可以使用 `--no-kv-events` 标志关闭 KV 事件追踪，改用近似 KV 索引：
>
> ```bash
> python -m dynamo.frontend \
>     --router-mode kv \
>     --http-port 8000 \
>     --no-kv-events
> ```

#### 解耦服务下的自动 prefill 路由

通过 `run_engines.sh --prefill` 启动 prefill worker 时，前端会自动检测它们并激活内置的 prefill router。该 prefill router：
- 自动把初始 token 处理路由到专用的 prefill worker
- 与前端的 `--router-mode` 设置保持相同的路由模式
- 与你的 decode worker 无缝衔接以完成 token 生成

无需额外配置——只需同时启动 decode 与 prefill worker，剩下的由系统处理。详见 [Router Guide](../../docs/components/router/router-guide.md#disaggregated-serving)。

> [!Note]
> 自动 prefill 路由的统一前端目前对 vLLM 与 TensorRT-LLM 后端可用。SGLang 仍在推进中，需要单独启动一个独立 router 作为指向 prefill 端点的 prefill router。示例脚本：[`examples/backends/sglang/launch/disagg_router.sh`](../../examples/backends/sglang/launch/disagg_router.sh)

### 第 3 步：验证环境

在另一个终端中测试一切是否就绪：

```bash
./ping.sh
# Or specify a different port:
./ping.sh 8000
```

它会向 router 发送一个简单的测试请求。如果配置正确，你将看到流式响应。

### 第 4 步：运行 benchmark

环境验证完成后，运行 prefix ratio benchmark：

```bash
python prefix_ratio_benchmark.py
```

默认配置：
- 测试的 prefix ratio：0.5（可通过 `--prefix-ratios 0.1 0.3 0.5 0.7 0.9` 自定义）
- 输入序列长度：14000 tokens
- 输出序列长度：200 tokens
- 请求数：200
- 并发：20

可以自定义 benchmark：

```bash
# Test multiple prefix ratios
python prefix_ratio_benchmark.py --prefix-ratios 0.1 0.3 0.5 0.7 0.9

# Adjust input/output lengths
python prefix_ratio_benchmark.py --isl 10000 --osl 500

# Change request count and concurrency
python prefix_ratio_benchmark.py --requests 500 --concurrency 50

# Use multiple router endpoints for parallel benchmarking (for testing multiple Router replicas)
python prefix_ratio_benchmark.py --url http://localhost:8000 http://localhost:8001

# Specify output directory
python prefix_ratio_benchmark.py --output-dir results/experiment1
```

### 第 5 步（可选）：使用真实 trace 数据 benchmark

除了用受控 prefix ratio 的合成 benchmark，你也可以用真实 trace 数据 benchmark。这种方式使用来自生产 trace 的真实请求模式，并可通过合成参数做调整。

先下载 mooncake trace 数据集：

```bash
wget https://raw.githubusercontent.com/kvcache-ai/Mooncake/d21da178bae8db9651cf18a76824c084145fc725/mooncake_trace.jsonl
```

然后运行 benchmark：

```bash
python real_data_benchmark.py --input-dataset mooncake_trace.jsonl
```

脚本可以在原始 trace 数据集上做多种修改，模拟不同场景与负载状态。它接受与 [prefix data generator](../prefix_data_generator/README.md) 相同的合成参数：

**关键参数：**
- `--num-requests`：从 trace 中合成的请求数（默认全部）
- `--speedup-ratio`：加快请求到达时间（如 2.0 让请求到达速度是原来的 2 倍）
- `--prefix-len-multiplier`：缩放共享前缀的长度（如 2.0 把前缀长度翻倍）
- `--prefix-root-multiplier`：把前缀树结构以不同根复制 N 次
- `--prompt-len-multiplier`：缩放唯一用户 prompt 的长度（如 0.5 缩短）
- `--max-isl`：过滤掉输入序列长度超过该值的请求

示例：

```bash
# Use original trace dataset as-is (no synthesis parameters specified)
python real_data_benchmark.py --input-dataset trace.jsonl

# Speed up request rate by 2x and use only first 1000 requests
python real_data_benchmark.py --input-dataset trace.jsonl --num-requests 1000 --speedup-ratio 2.0

# Double prefix lengths to test cache efficiency with longer shared contexts
python real_data_benchmark.py --input-dataset trace.jsonl --prefix-len-multiplier 2.0

# Create more diverse workload by replicating prefix tree 3 times
python real_data_benchmark.py --input-dataset trace.jsonl --prefix-root-multiplier 3
```

> [!Note]
> 撰写本文档时，要在 trace 文件上做 loadgen 可能需要从 main 分支安装最新的 aiperf：
> ```bash
> pip install git+https://github.com/ai-dynamo/aiperf.git
> ```
> 不过到正式发布时，vLLM runtime 容器中包含的 aiperf 版本应已足够新可以直接用。

### 第 6 步（可选）：优先级队列 benchmark

`real_data_priority_benchmark.py` 测试 router 的优先级队列能否正确区分高/中/低优先级请求。它把 trace 切成三层，分别跑一次**基线**（无优先级标记）和一次**带优先级标记**的运行，然后画出对比 TTFT 的柱状图。

#### 工作方式

1. trace 通过与 `real_data_benchmark.py` 相同的参数合成，并按 `--priority-distribution` 分成低/中/高三层。
2. 每层作为一个并发流送到 aiperf。在带优先级的运行中，每个请求都带一个 OpenAI 兼容扩展头：
   ```json
   {"nvext": {"agent_hints": {"priority": <value>}}}
   ```
   `priority` 值会提升请求在 router 队列中的优先级——值越高，请求"看起来"到得越早，从而排在低值请求前面。
3. baseline 与 priority 两次运行使用不同的 aiperf 种子，确保生成 prompt 内容不同，避免 mocker KV 缓存交叉污染。

#### 前置条件：启用优先级队列

只有设置 `--router-queue-threshold` 时 router 队列才会激活。否则请求完全绕过队列，优先级也就无效。

```bash
# Launch the router with priority queue enabled.
# The fraction (e.g. 1.2) controls the busy threshold:
# workers are considered "busy" when active prefill tokens exceed
# threshold * max_num_batched_tokens. Values > 1.0 effectively make
# the queue always active.
python -m dynamo.frontend \
    --router-mode kv \
    --router-reset-states \
    --router-queue-threshold 1.2
```

#### 运行 benchmark

由于 mocker 默认 speedup ratio 是 1.0（实时），需要足够高的 `--speedup-ratio` 才能产生足够的并发负载让请求真正排队。建议 8 或更高：

```bash
python real_data_priority_benchmark.py \
    --input-dataset mooncake_trace.jsonl \
    --num-requests 5000 \
    --speedup-ratio 8 \
    --prefix-len-multiplier 4 \
    --prefix-root-multiplier 4
```

**优先级专属参数：**

| 参数 | 默认值 | 说明 |
|-----------|---------|-------------|
| `--priority-distribution` | `0.5,0.3,0.2` | 分配给低/中/高三层的请求比例（必须和为 1.0） |
| `--priority-values` | `0,1,2` | 三层（低/中/高）对应的 `priority` 值 |

示例：

```bash
# Equal tier sizes with aggressive priority differentiation.
# --priority-values sets the request priority per tier (low, medium, high).
# Higher values move the request further ahead in the router queue.
# Here low gets no boost, medium gets priority 2, and high gets priority 5.
python real_data_priority_benchmark.py \
    --input-dataset mooncake_trace.jsonl \
    --num-requests 5000 \
    --speedup-ratio 8 \
    --priority-distribution 0.33,0.34,0.33 \
    --priority-values 0,2,5
```

benchmark 会在结果目录输出 `ttft_comparison.png` 柱状图，显示每层 TTFT（p50 + p25–p75 误差棒），并对比 baseline 与 priority 两次运行。如果优先级队列工作正常，priority 运行中高优先级请求的 TTFT 应低于 baseline，低优先级请求的 TTFT 可能略高。

### 第 7 步（可选）：Agent benchmark（基于并发的多轮）

要使用基于并发（而非按时间戳重放）的方式 benchmark 多轮对话 trace，使用 `agent_benchmark.py`。它适合测试系统如何处理多个并发 agent 会话。

```bash
python agent_benchmark.py --input-dataset trace.jsonl --concurrency 10
```

**关键参数：**
- `--concurrency`：维持的并发会话数（默认 10）
- `--delay`：覆盖会话内多轮之间的间隔（毫秒）。设为 0 可去掉所有延迟。

示例：

```bash
# Run with 20 concurrent sessions using delays from trace file
python agent_benchmark.py --input-dataset trace.jsonl --concurrency 20

# Run with no delays between turns (stress test)
python agent_benchmark.py --input-dataset trace.jsonl --concurrency 10 --delay 0

# Run with fixed 1-second delay between turns
python agent_benchmark.py --input-dataset trace.jsonl --concurrency 10 --delay 1000
```

### Trace 数据集格式（JSONL）

`real_data_benchmark.py` 与 `agent_benchmark.py` 都接受 JSONL 格式的 trace（每行一个 JSON 对象）。该格式与 [Mooncake trace format](https://github.com/kvcache-ai/Mooncake) 兼容。

#### 字段

| 字段 | 类型 | 说明 |
|-------|------|-------------|
| `input_length` | int | 该请求的输入 token 数 |
| `output_length` | int | 要生成的输出 token 数 |
| `session_id` | string | 把多轮对话归到一起。`session_id` 相同的请求按顺序处理。 |
| `hash_ids` | list[int] | 该请求用于 KV 缓存路由的前缀块 hash ID 列表。共享 hash ID 表示前缀共享。 |
| `delay` | int | 在该轮发送前等待的毫秒数（在同会话上一轮完成后生效）。第一轮不应用。 |

#### Trace 文件示例

```jsonl
{"session_id": "conv_0", "input_length": 9176, "output_length": 152, "hash_ids": [0, 1, 2, 3, 4, 5]}
{"session_id": "conv_0", "input_length": 9368, "output_length": 104, "hash_ids": [0, 1, 2, 3, 4, 5, 6, 7], "delay": 500}
{"session_id": "conv_0", "input_length": 9516, "output_length": 164, "hash_ids": [0, 1, 2, 3, 4, 5, 6, 7, 8, 9], "delay": 500}
{"session_id": "conv_1", "input_length": 9445, "output_length": 143, "hash_ids": [0, 1, 2, 10, 11, 12, 13]}
{"session_id": "conv_1", "input_length": 9628, "output_length": 123, "hash_ids": [0, 1, 2, 10, 11, 12, 13, 14, 15], "delay": 500}
```

在该示例中：
- `conv_0` 与 `conv_1` 是两个可并发的对话
- 同一对话内的轮次按顺序处理
- 后续轮次在前一轮完成后等待 500ms
- `hash_ids` 体现前缀共享：两个对话都共享前缀块 `[0, 1, 2]`

## Benchmark 结果

我们在 8 张 L40S GPU 上以聚合模式部署 deepseek-ai/DeepSeek-R1-Distill-Llama-8B，把 Dynamo KV Router 与基线轮询路由策略做对比，评估缓存感知路由的收益。配置：

- **ISL/OSL**：14000/200
- **Prefix Ratio**：0.1、0.3、0.5、0.7、0.9
- **负载**：200 个请求被组织为 20 个前缀组
- **并发**：20

![Router Performance Comparison](results.png)

结果表明，Dynamo KV Router 在所有 prefix ratio 下都稳定优于轮询路由，且性能优势随 prefix ratio 增大而增大。这凸显了在多轮对话、文档问答以及 prompt 工程迭代等具有显著前缀共享的工作负载中，缓存感知路由的重要性。

## 故障排查

1. **worker 启动失败**：检查 CUDA_VISIBLE_DEVICES 与 GPU 可用性
2. **router 拒绝连接**：确认 router 在运行且端口正确
3. **benchmark 超时**：降低并发或减少请求数
4. **OOM**：在 run_engines.sh 中减小 max-num-batched-tokens 或 max-model-len
