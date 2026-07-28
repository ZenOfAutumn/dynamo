---
# SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Planner Replay Benchmarking
subtitle: Drive the planner in the loop against a saved trace to evaluate SLA behavior and scaling decisions
---

本指南说明如何通过在 mock replay 测试框架内运行已记录 trace，对 Dynamo Planner（调度器）进行基准测试。可用于对比 `agg`（聚合）与 `disagg`（解耦）拓扑、调优 SLA 目标，以及研究部署现实因素（引擎启动时间、worker 数量）如何影响 planner 行为 —— 整个过程都不需要拉起真实集群。

关于 trace replay 的通用机制（输入格式、到达加速比、router 模式、合成负载），请参见 [Mocker Offline Trace Replay](mocker-trace-replay.md)。本指南聚焦 `--planner-config` 这条路径。

## 1. 准备

### 构建

构建 Dynamo Python 绑定，使 `python -m dynamo.replay` 可用：

```bash
cd lib/bindings/python
.venv/bin/maturin develop --release
```

强烈推荐使用 `--release` 标志。replay 模拟主要是单线程且对 mocker 引擎核 CPU 密集；debug 构建可能慢 5–10 倍，多次扫描运行下成本会快速放大。

### 关键 Planner 配置参数

通过 `--planner-config` 以 JSON 形式传入，使用与线上 planner 相同的 schema。基准测试中最相关的字段：

| 字段 | 用途 |
|---|---|
| `mode` | `"agg"` 或 `"disagg"` —— 决定扩缩策略以及所需的引擎参数。 |
| `optimization_target` | `"sla"` 使用 TTFT/ITL 目标；`"throughput"` 使用静态队列/KV 阈值。 |
| `ttft_ms` / `itl_ms` | 以毫秒为单位的 SLA 目标，驱动负载扩缩决策。 |
| `enable_throughput_scaling` | 基于稳态负载预测的周期性扩缩。 |
| `enable_load_scaling` | 针对短期流量峰值的反应式扩缩。 |
| `throughput_adjustment_interval_seconds` | 吞吐扩缩决策之间的秒数间隔。 |
| `load_adjustment_interval_seconds` | 负载扩缩决策之间的秒数间隔。间隔越短反应越快，但会更易抖动。 |
| `pre_deployment_sweeping_mode` | `"rapid"` 使用 AIC 解析模型；不设置则回退到已记录的 profile 数据。 |
| `prefill_engine_num_gpu` / `decode_engine_num_gpu` | 每个引擎副本所占 GPU 数。**必须显式设置** —— 两者默认值都是 `None`，replay adapter 会静默将 `None` 当作 `0`，从而使报告中的累计 GPU 小时指标坍缩为 0。 |
| `report_filename` | 输出 HTML 文件名，位于 `./planner_reports/` 下。 |

### 关键引擎参数

agg 模式通过 `--extra-engine-args`，disagg 模式通过 `--prefill-engine-args` / `--decode-engine-args` 以 JSON 形式传入。replay 测试框架使用 mocker 引擎，因此这里的 "engine args" 实际上是解析性能模型的输入：

| 字段 | 用途 |
|---|---|
| `aic_backend` | 解析模型应仿真的后端，例如 `"vllm"`、`"trtllm"`、`"sglang"`。 |
| `aic_system` | 性能模型的 GPU SKU，例如 `"h200_sxm"`、`"h100_sxm"`、`"b200"`。 |
| `aic_model_path` | 性能模型使用的 HF 模型标识。 |
| `aic_tp_size` | 每个引擎副本的张量并行（TP）规模。 |
| `startup_time` | 模拟从 planner 决定扩容到新 worker 真正可用之间的秒数。未设置或为 `0` 表示 worker 立即生效。 |

其余字段遵循标准的 mocker 引擎协议（参见 [Mocker Offline Trace Replay](mocker-trace-replay.md)）。

## 2. 示例：在 Mooncake Agentic Trace 上对比 Agg 与 Disagg

下载 trace：

```bash
mkdir -p traces/mooncake_fast25 && cd traces/mooncake_fast25
curl -sLO https://raw.githubusercontent.com/kvcache-ai/Mooncake/main/FAST25-release/traces/toolagent_trace.jsonl
```

运行 agg（2 worker，TP=1）：

```bash
.venv/bin/python -m dynamo.replay traces/mooncake_fast25/toolagent_trace.jsonl \
  --planner-config '{
    "mode": "agg",
    "optimization_target": "sla",
    "ttft_ms": 1500, "itl_ms": 50,
    "enable_throughput_scaling": true, "enable_load_scaling": true,
    "pre_deployment_sweeping_mode": "rapid",
    "throughput_adjustment_interval_seconds": 300, "load_adjustment_interval_seconds": 10,
    "prefill_engine_num_gpu": 1, "decode_engine_num_gpu": 1,
    "report_filename": "replay_agg.html"
  }' \
  --extra-engine-args '{"aic_backend": "vllm", "aic_system": "h200_sxm", "aic_model_path": "nvidia/Llama-3.1-8B-Instruct-FP8", "aic_tp_size": 1}' \
  --num-workers 2 --arrival-speedup-ratio 1.0
```

运行 disagg（1P1D，TP=1）：

```bash
.venv/bin/python -m dynamo.replay traces/mooncake_fast25/toolagent_trace.jsonl \
  --planner-config '{
    "mode": "disagg",
    "optimization_target": "sla",
    "ttft_ms": 1500, "itl_ms": 50,
    "enable_throughput_scaling": true, "enable_load_scaling": true,
    "pre_deployment_sweeping_mode": "rapid",
    "throughput_adjustment_interval_seconds": 300, "load_adjustment_interval_seconds": 10,
    "prefill_engine_num_gpu": 1, "decode_engine_num_gpu": 1,
    "report_filename": "replay_disagg.html"
  }' \
  --prefill-engine-args '{"aic_backend": "vllm", "aic_system": "h200_sxm", "aic_model_path": "nvidia/Llama-3.1-8B-Instruct-FP8", "aic_tp_size": 1}' \
  --decode-engine-args  '{"aic_backend": "vllm", "aic_system": "h200_sxm", "aic_model_path": "nvidia/Llama-3.1-8B-Instruct-FP8", "aic_tp_size": 1}' \
  --num-prefill-workers 1 --num-decode-workers 1 --arrival-speedup-ratio 1.0
```

每次运行都会向 stdout 打印 AIPerf 摘要表，并向 `./planner_reports/<report_filename>` 写入 HTML 诊断报告。对于这条长 ISL、短 OSL 的 trace，agg 优于 disagg —— 后者只换来略好的 ITL，却付出了明显更多的 GPU 小时。

## 3. 示例：冷启动时间扫描

SLA 达成对引擎启动时间的敏感度有多高？将 `startup_time` 从 0 到 300 秒以 10 秒为步长进行扫描，并记录每次运行的 TTFT/ITL/GPU 小时。

```bash
#!/usr/bin/env bash
set -euo pipefail

TRACE=traces/mooncake_fast25/toolagent_trace.jsonl
OUT=planner_reports/sweep_startup
mkdir -p "$OUT"

run_one() {
  local s=$1
  local name=$(printf "replay_agg_startup_%03d.html" "$s")
  local extra
  if [[ "$s" -eq 0 ]]; then
    extra='{"aic_backend":"vllm","aic_system":"h200_sxm","aic_model_path":"nvidia/Llama-3.1-8B-Instruct-FP8","aic_tp_size":1}'
  else
    extra=$(printf '{"aic_backend":"vllm","aic_system":"h200_sxm","aic_model_path":"nvidia/Llama-3.1-8B-Instruct-FP8","aic_tp_size":1,"startup_time":%d}' "$s")
  fi
  .venv/bin/python -m dynamo.replay "$TRACE" \
    --planner-config "$(printf '{"mode":"agg","optimization_target":"sla","ttft_ms":1500,"itl_ms":50,"enable_throughput_scaling":true,"enable_load_scaling":true,"pre_deployment_sweeping_mode":"rapid","throughput_adjustment_interval_seconds":300,"load_adjustment_interval_seconds":10,"prefill_engine_num_gpu":1,"decode_engine_num_gpu":1,"report_filename":"%s"}' "$name")" \
    --extra-engine-args "$extra" \
    --num-workers 2 --arrival-speedup-ratio 1.0 \
    --report-json "$OUT/startup_$(printf '%03d' "$s").json" \
    >"$OUT/startup_$(printf '%03d' "$s").log" 2>&1
}

export -f run_one
# Run 12 sweeps in parallel; adjust -P for your machine.
seq 0 10 300 | xargs -n1 -P12 -I{} bash -c 'run_one "$@"' _ {}
```

每次运行都会输出 AIPerf 指标表（解析 TTFT / ITL avg / p90），以及它的 HTML 报告（grep `GPU hours: <float>`）。将这些指标对 `startup_time` 作图：

![Planner replay — startup time sweep](../assets/img/planner-replay-startup-sweep.png)

本扫描的观察结果（agg、TTFT SLA 1,500 ms、ITL SLA 50 ms、H200-SXM、Llama-3.1-8B-FP8、TP=1）：

- **SLA 在约 100–120 秒处出现悬崖。** 在这之下，planner 扩容速度足以维持 TTFT；超过此区间后，p99 TTFT 发散，系统持续处于 backlog 状态。
- **扩缩事件次数单调递减**（42 → 8）：启动时间越长，load planner 在做下一次扩缩决策前必须等待系统稳定。
- **ITL 在队列饱和前比 TTFT 更不敏感**。在悬崖之下，ITL 平均仅小幅上升（25 → 30 ms）；超过悬崖后，由于 decode 请求被饿死，p90 ITL 跃升到约 200 ms。
