# Flash Indexer 图表 -- 复现指南

构建 Flash Indexer Digest 帖子中所有图表的说明。

## 图表清单

所有输出位于 `../images/`。

| 文件 | 描述 |
|------|-------------|
| `fig-1-kv-event-density.{svg,png}` | KV 缓存事件密度热力图（Mooncake trace） |
| `fig-2-kv-event-flow.{svg,png}` | KV 事件流水线：engines → publishers → indexer → router |
| `fig-3-prefix-tree.{svg,png}` | 带 worker 跟踪的前缀感知 radix 树 |
| `fig-4-concurrency-model.{svg,png}` | 并发模型：粘性路由 + 并发读 |
| `fig-5-jump-search.{svg,png}` | 带回溯的位置跳跃搜索 |
| `fig-6-indexer-throughput.{svg,png}` | 基准：实际 vs 期望吞吐量（5 个后端） |

## 前置条件

```bash
pip3 install plotly kaleido numpy pyyaml
brew install librsvg   # for rsvg-convert (SVG -> PNG)
brew install d2        # only needed to re-render D2 sources
```

## 复现

### 一键构建（所有图表）

```bash
./build.sh          # figures 1-6 (D2 sources already processed)
./build.sh --d2     # re-render D2 sources first, then all figures
```

### 架构图（图 2-5）

```bash
# From this directory (tools/):

# 1. (Optional) Re-render D2 -> raw SVG (requires d2 CLI)
d2 --layout tala event-flow.d2      event-flow-raw.svg
d2 --layout elk  radix-tree.d2      radix-tree-raw.svg
d2 --layout tala write-read-path.d2 write-read-path-raw.svg

# 2. Inject legends + padding, write to ../images/
python3 inject_legends.py

# 3. Render SVGs to 2x PNGs
rsvg-convert -z 2 ../images/fig-2-kv-event-flow.svg    -o ../images/fig-2-kv-event-flow.png
rsvg-convert -z 2 ../images/fig-3-prefix-tree.svg       -o ../images/fig-3-prefix-tree.png
rsvg-convert -z 2 ../images/fig-4-concurrency-model.svg -o ../images/fig-4-concurrency-model.png
rsvg-convert -z 2 ../images/fig-5-jump-search.svg       -o ../images/fig-5-jump-search.png
```

### 性能图表（图 6）

```bash
python3 gen_throughput.py ../data/sweep_plot.json
```

### KV 缓存事件密度热力图（图 1）

```bash
# Synthetic data (no trace file needed):
python3 gen_heatmap.py

# Real Mooncake trace data (98 MB, download separately):
python3 gen_heatmap.py --real-data PATH_TO/kv_events_real.json
```

真实 trace 数据是
[Mooncake FAST'25 trace](https://github.com/kvcache-ai/Mooncake/blob/main/FAST25-release/arxiv-trace/mooncake_trace.jsonl)
的前 5%，在 16 个模拟 worker（每 worker 2,048 个 block，block_size 16）上回放。
该文件过大无法纳入仓库；可使用
[`mocker`](https://github.com/ai-dynamo/dynamo/tree/main/lib/mocker) 生成，或
下载 trace 后重放。

热力图脚本会读取 `design_tokens.yaml` 与 `plotly_dynamo.py` 以使用
Dynamo 暗色主题。

## 内容

```text
tools/
├── README.md                  # This file
├── build.sh                   # One-shot build for all figures
├── inject_legends.py          # SVG legend injection (Figures 2-5)
├── gen_heatmap.py             # KV event heatmap generator (Figure 1)
├── gen_throughput.py           # Throughput chart generator (Figure 6)
├── design_tokens.yaml         # Shared color/typography tokens
├── plotly_dynamo.py           # Plotly template builder
├── dynamo.d2                  # D2 theme file
├── theme.d2                   # Shared D2 theme
├── event-flow.d2              # D2 source for Figure 2
├── radix-tree.d2              # D2 source for Figure 3
└── write-read-path.d2         # D2 source for Figure 4
```
