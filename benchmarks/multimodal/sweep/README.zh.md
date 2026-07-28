# 多模态基准测试 Sweep

由 YAML 驱动的基准测试编排器，可启动各服务后端（backend）、运行
[aiperf](https://github.com/triton-inference-server/perf_analyzer) 并发 sweep，
并可选地生成对比图表。

## 快速开始

```bash
# from the repo root
python -m benchmarks.multimodal.sweep \
  --config benchmarks/multimodal/sweep/experiments/embedding_cache/vllm_serve.yaml
```

## 工作原理

1. 解析 YAML 实验配置。
2. 对每个 **输入文件** × 每个 **基准配置** 进行：
   - 通过 workflow 脚本启动服务后端。
   - 在每个并发等级上运行 `aiperf profile`。
   - 停止服务（默认情况下，服务会在不同并发等级之间重启，避免热缓存
     带来的偏差 —— 由 `restart_server_every_benchmark` 控制）。
3. 针对每个输入文件，生成跨配置的对比图表。

## YAML 配置参考

```yaml
model: Qwen/Qwen3-VL-30B-A3B-Instruct-FP8
concurrencies: [16, 32, 64, 128, 256]
osl: 150                    # output sequence length
conversation_num: 10        # sessions per sweep value (optional; derived from
                            # input JSONL's unique session_id count if unset;
                            # flat JSONLs count each row as a 1-turn conversation)
warmup_count: 5
port: 8000
timeout: 900                # seconds to wait for server readiness
output_dir: benchmarks/multimodal/sweep/results/vllm_serve

# Optional env vars injected into the server process
env:
  ENABLE_ENCODER_CACHE: "0"

# JSONL files produced by benchmarks/multimodal/jsonl/
input_files:
  - benchmarks/multimodal/jsonl/1000req_1img_200pool_400word_http.jsonl
  - benchmarks/multimodal/jsonl/1000req_4img_200pool_400word_http.jsonl

# Each config launches the workflow with its own extra_args
configs:
  - label: cache-off
    workflow: benchmarks/multimodal/sweep/workflows/vllm_serve.sh
    extra_args: [--no-enable-prefix-caching, --multimodal-embedding-cache-capacity-gb, "0"]

  - label: cache-on
    workflow: benchmarks/multimodal/sweep/workflows/vllm_serve.sh
    extra_args: [--no-enable-prefix-caching, --multimodal-embedding-cache-capacity-gb, "10"]
```

## CLI 覆盖

YAML 中的任意顶层字段都可以通过命令行覆盖：

```bash
python -m benchmarks.multimodal.sweep \
  --config experiments/embedding_cache/vllm_serve.yaml \
  --concurrencies 1,2,4 \
  --osl 200 \
  --conversation-num 10 \
  --skip-plots
```

## 预热（Warmup）语义

`warmup_count: N` 是**请求（轮次）预算**，而不是会话（session）预算。对于
一个 10×10 的 JSONL，`warmup_count: 2` 意味着预热阶段总共发出 2 个请求 ——
两个都会落到 `user_0`（第 0 与第 1 轮），因为 aiperf 的"延续轮"优先策略
会持续向当前进行中的会话喂请求，直到其预算耗尽。预热并不会消耗 2 个完整
会话（20 个请求）。压测随后从 `user_1` 开始，依次跑完 `user_1..user_9`，
最后再回到一个全新的 `user_0` 实例作为第 10 个会话。建议保持
`warmup_count` 较小（≤ 每会话的轮次数），以确保预热停留在单个会话的前缀内。

## 输出目录结构

按上面的配置，使用两个输入文件、两个配置（`cache-off`、`cache-on`）以及
并发 `[16, 32]`，输出树结构如下：

```
<output_dir>/
├── 1000req_1img_200pool_400word_http/      # ← derived from input filename
│   ├── cache-off/                          # ← config label
│   │   ├── c16/                            # ← concurrency level
│   │   │   ├── profile_export.jsonl
│   │   │   ├── profile_export_aiperf.json
│   │   │   ├── profile_export_aiperf.csv
│   │   │   ├── gpu_telemetry_export.jsonl
│   │   │   ├── inputs.json
│   │   │   └── logs/
│   │   │       └── aiperf.log
│   │   └── c32/
│   │       └── ...
│   ├── cache-on/
│   │   ├── c16/
│   │   │   └── ...
│   │   └── c32/
│   │       └── ...
│   └── plots/                              # ← comparison plots across configs
│       └── ...
└── 1000req_4img_200pool_400word_http/
    ├── cache-off/
    │   └── ...
    ├── cache-on/
    │   └── ...
    └── plots/
        └── ...
```

## 已有实验

| 实验 | 配置 | 后端 |
|---|---|---|
| Embedding cache（vLLM serve） | `experiments/embedding_cache/vllm_serve.yaml` | 单节点 vLLM |
| Embedding cache（vLLM E+PD） | `experiments/embedding_cache/vllm_e_pd.yaml` | 解耦（disaggregated） vLLM E+PD |
| Embedding cache（TRT-LLM E+PD） | `experiments/embedding_cache/trtllm_e_pd.yaml` | 解耦 TRT-LLM E+PD |
