# 分片 KV Router 的 Benchmark

## Trace 数据

Benchmark 使用 JSONL trace 文件。每行是一个 JSON 对象，字段如下：

| 字段 | 类型 | 说明 |
|-------|------|-------------|
| `timestamp` | float (ms) | 绝对到达时间；也可省略改用 `delay` |
| `delay` | float (ms) | 距离上一个请求的时间（与 `timestamp` 二选一） |
| `hash_ids` | `u64[]` | 块级 KV 缓存（KV cache）hash ID |
| `output_length` | `u64` | 输出 token 数 |
| `input_length` | `u64`（可选） | 输入 token 数 |

### 公开 trace（Mooncake FAST25）

除非你刻意要测试次要工作负载，否则把 Mooncake FAST25 arxiv trace 作为主要 benchmark trace。它是与外部 benchmark 讨论最匹配的 trace：

```bash
mkdir -p lib/kv-router/traces
curl -L https://raw.githubusercontent.com/kvcache-ai/Mooncake/main/FAST25-release/arxiv-trace/mooncake_trace.jsonl \
  -o lib/kv-router/traces/mooncake_trace.jsonl
```

次要 trace：

```bash
curl -L https://raw.githubusercontent.com/kvcache-ai/Mooncake/main/FAST25-release/traces/conversation_trace.jsonl \
  -o lib/kv-router/traces/conversation_trace.jsonl
curl -L https://raw.githubusercontent.com/kvcache-ai/Mooncake/main/FAST25-release/traces/synthetic_trace.jsonl \
  -o lib/kv-router/traces/synthetic_trace.jsonl
curl -L https://raw.githubusercontent.com/kvcache-ai/Mooncake/main/FAST25-release/traces/toolagent_trace.jsonl \
  -o lib/kv-router/traces/toolagent_trace.jsonl
```

| 文件 | 描述 |
|------|-------------|
| `mooncake_trace.jsonl` | 主要的 arxiv 工作负载：1 小时内 23,608 个请求，存在两个占主导的热前缀 |
| `conversation_trace.jsonl` | 较小的对话工作负载：12,031 个请求，hash 序列与 arxiv trace 不同 |
| `synthetic_trace.jsonl` | 合成工作负载：3,993 个请求，体量更小、结构差异更大 |
| `toolagent_trace.jsonl` | Agentic/工具使用工作负载：请求数与分布与 arxiv trace 类似，但 hash 序列不同 |

当前本地副本的 trace 形态汇总：

| 文件 | 请求数 | 跨度 | 平均块/请求 | 唯一 depth-2 前缀 | Top depth-2 前缀 |
|------|---------:|------|-------------------:|------------------------:|----------------------|
| `mooncake_trace.jsonl` | 23,608 | 3,600,000 ms | 17.3 | 6,557 | `[46,47]` × 9,203；`[74,75]` × 3,449 |
| `conversation_trace.jsonl` | 12,031 | 3,536,999 ms | 24.0 | 7,373 | top 前缀出现 43 次 |
| `synthetic_trace.jsonl` | 3,993 | 1,022,025 ms | 30.5 | 1,138 | top 前缀出现 24 次 |
| `toolagent_trace.jsonl` | 23,608 | 3,536,999 ms | 17.4 | 6,554 | `[46,47]` × 9,203；`[74,75]` × 3,449 |

---

## 两种 benchmark 模式

**稳态（Steady-state）**（`--benchmark-duration-ms 30000`）：按 trace 的自然到达速率回放。衡量真实场景下的 p99 延迟。吞吐受限于 trace 速率——两种 indexer 都应能跟得上，因此 ops/s 类似；有意义的对比是 p99。

**峰值吞吐扫描（Peak throughput sweep）**（`--sweep`）：逐步收紧 benchmark 时间窗口，把发起速率推过饱和点。用它来对比不同 indexer 的最大吞吐。

---

## 运行 benchmark

所有 benchmark 都通过 `dynamo-bench` 中的 `mooncake_bench` 运行。**从仓库根目录运行。** bench 二进制工作目录是 `lib/bench/`，所以 trace 路径必须是绝对路径——下面命令使用 `$(git rev-parse --show-toplevel)` 以增强可移植性。

```bash
cargo bench --package dynamo-bench --bench mooncake_bench -- \
  <TRACE_PATH> [global options] <INDEXER_SUBCOMMAND> [indexer options]
```

### 自检（无需 trace）

```bash
cargo test --package dynamo-bench --test mooncake_trace
```

### 稳态（按真实请求速率测 p99）

**CRTC 基线（8 个 worker）：**
```bash
cargo bench --package dynamo-bench --bench mooncake_bench -- \
  $(git rev-parse --show-toplevel)/lib/kv-router/traces/mooncake_trace.jsonl \
  --trace-simulation-duration-ms 10000 --benchmark-duration-ms 30000 -d 7 \
  concurrent-radix-tree-compressed --num-event-workers 8
```

**Branch-sharded depth=2（2 shard × 4 worker）：**
```bash
cargo bench --package dynamo-bench --bench mooncake_bench -- \
  $(git rev-parse --show-toplevel)/lib/kv-router/traces/mooncake_trace.jsonl \
  --trace-simulation-duration-ms 10000 --benchmark-duration-ms 30000 -d 7 \
  branch-sharded-crtc --num-shards 2 --num-event-workers-per-shard 4 --prefix-depth 2
```

**Branch-sharded depth=4（2 shard × 4 worker）：**
```bash
cargo bench --package dynamo-bench --bench mooncake_bench -- \
  $(git rev-parse --show-toplevel)/lib/kv-router/traces/mooncake_trace.jsonl \
  --trace-simulation-duration-ms 10000 --benchmark-duration-ms 30000 -d 7 \
  branch-sharded-crtc --num-shards 2 --num-event-workers-per-shard 4 --prefix-depth 4
```

**Anchor-aware branch-sharded depth=2（2 shard × 4 worker）：**
```bash
cargo bench --package dynamo-bench --bench mooncake_bench -- \
  $(git rev-parse --show-toplevel)/lib/kv-router/traces/mooncake_trace.jsonl \
  --trace-simulation-duration-ms 10000 --benchmark-duration-ms 30000 -d 7 \
  anchor-aware-branch-sharded-crtc --num-shards 2 --num-event-workers-per-shard 4 --prefix-depth 2
```

### 峰值吞吐扫描

**CRTC 基线 —— sweep：**
```bash
cargo bench --package dynamo-bench --bench mooncake_bench -- \
  $(git rev-parse --show-toplevel)/lib/kv-router/traces/mooncake_trace.jsonl \
  --trace-simulation-duration-ms 10000 -d 7 \
  --sweep --sweep-min-ms 1000 --sweep-max-ms 30000 --sweep-steps 8 \
  concurrent-radix-tree-compressed --num-event-workers 8
```

**Branch-sharded depth=2 —— sweep：**
```bash
cargo bench --package dynamo-bench --bench mooncake_bench -- \
  $(git rev-parse --show-toplevel)/lib/kv-router/traces/mooncake_trace.jsonl \
  --trace-simulation-duration-ms 10000 -d 7 \
  --sweep --sweep-min-ms 1000 --sweep-max-ms 30000 --sweep-steps 8 \
  branch-sharded-crtc --num-shards 2 --num-event-workers-per-shard 4 --prefix-depth 2
```

**Anchor-aware branch-sharded depth=2 —— sweep：**
```bash
cargo bench --package dynamo-bench --bench mooncake_bench -- \
  $(git rev-parse --show-toplevel)/lib/kv-router/traces/mooncake_trace.jsonl \
  --trace-simulation-duration-ms 10000 -d 7 \
  --sweep --sweep-min-ms 1000 --sweep-max-ms 30000 --sweep-steps 8 \
  anchor-aware-branch-sharded-crtc --num-shards 2 --num-event-workers-per-shard 4 --prefix-depth 2
```

### Worker 扩展

```bash
for factor in 1 2 4 8 16 32; do
  cargo bench --package dynamo-bench --bench mooncake_bench -- \
    $(git rev-parse --show-toplevel)/lib/kv-router/traces/mooncake_trace.jsonl \
    --trace-simulation-duration-ms 10000 --benchmark-duration-ms 30000 \
    --num-unique-inference-workers 1000 \
    --trace-duplication-factor $factor \
    -d 7 \
    branch-sharded-crtc --num-shards 2 --num-event-workers-per-shard 4 --prefix-depth 2
done
```

### 反复过载 benchmark

这种短窗口 benchmark 故意把发起负载推到每秒数百万 request/event 操作。该运行适合做压力形态对比，但出现警告是预期的，所获得的数值不应被解读为干净的稳态容量。

**CRTC 基线（8 个 worker）：**
```bash
cargo bench --package dynamo-bench --bench mooncake_bench -- \
  $(git rev-parse --show-toplevel)/lib/kv-router/traces/mooncake_trace.jsonl \
  --num-unique-inference-workers 128 \
  --trace-duplication-factor 20 \
  --trace-length-factor 4 \
  --benchmark-duration-ms 750 \
  --benchmark-runs 20 \
  concurrent-radix-tree-compressed \
  --num-event-workers 8
```

**Branch-sharded depth=2，相同事件 worker 总数（2 shard × 4 worker）：**
```bash
cargo bench --package dynamo-bench --bench mooncake_bench -- \
  $(git rev-parse --show-toplevel)/lib/kv-router/traces/mooncake_trace.jsonl \
  --num-unique-inference-workers 128 \
  --trace-duplication-factor 20 \
  --trace-length-factor 4 \
  --benchmark-duration-ms 750 \
  --benchmark-runs 20 \
  branch-sharded-crtc \
  --num-shards 2 \
  --num-event-workers-per-shard 4 \
  --prefix-depth 2
```

**Anchor-aware branch-sharded depth=2，相同事件 worker 总数（2 shard × 4 worker）：**
```bash
cargo bench --package dynamo-bench --bench mooncake_bench -- \
  $(git rev-parse --show-toplevel)/lib/kv-router/traces/mooncake_trace.jsonl \
  --num-unique-inference-workers 128 \
  --trace-duplication-factor 20 \
  --trace-length-factor 4 \
  --benchmark-duration-ms 750 \
  --benchmark-runs 20 \
  anchor-aware-branch-sharded-crtc \
  --num-shards 2 \
  --num-event-workers-per-shard 4 \
  --prefix-depth 2
```

要使用更大的 branch-sharded worker 池，设 `--num-event-workers-per-shard 8`（2 shard 共 16 个事件 worker）。

---

## 理解输出

### 标准字段（所有 indexer）

| 字段 | 含义 |
|-------|---------|
| Offered ops/s | 计划请求速率 = 总操作数 / benchmark 窗口 |
| Achieved ops/s | 实际完成的速率——未饱和时与 offered 相符 |
| p99 latency | `find_matches` 第 99 百分位时延 |

### Branch-sharded 额外字段

| 字段 | 含义 |
|-------|---------|
| Dispatched | 路由到某个 shard 做 lookup 的查询 |
| Early-exit % | 没有已注册前缀别名、无需 shard 派发即可解析的查询比例 |
| Avg routing | 前缀键路由表的平均查表时间 |
| Avg shard | 派发到的 shard 上 CRTC 遍历的平均时间 |

---

## 关键 CLI 参数

| 参数 | 默认值 | 作用 |
|------|---------|-------------|
| `-d N` | 1 | Worker 复制因子：用 N 份每个唯一 worker 回放 trace（越大写压力越大） |
| `--find-matches-concurrency N` | 0 | 在 trace 回放外额外开 N 个紧循环发起 `find_matches` 的 tokio 任务，压测读路径 |
| `--trace-simulation-duration-ms` | — | 把 trace 重缩放到该 wall-clock 时长（毫秒）；省略则保留原始 Mooncake 时间戳 |
| `--benchmark-duration-ms` | 60000 | 测量窗口；越短发起速率越高 |
| `--num-unique-inference-workers` | 1000 | 从 trace 划分出的 worker 数 |
| `--trace-length-factor` | 1 | 拉长每个请求的 hash 序列 |
| `--trace-duplication-factor` | 1 | 创建结构相同但 hash 空间不重叠的副本 |
| `--seed` | 42 | worker-to-trace 分配的 RNG 种子 |
| `--sweep` | off | 通过变化 benchmark 窗口寻找饱和点 |
| `--sweep-min-ms` | 1000 | sweep 中最短的 benchmark 窗口 |
| `--sweep-max-ms` | 50000 | sweep 中最长的 benchmark 窗口 |
| `--sweep-steps` | 10 | sweep 步数 |
| `--shard-metrics-csv FILE` | — | 随时间采样 shard block/node 数 → CSV + SVG |

### `branch-sharded-crtc` 参数

| 参数 | 默认值 | 作用 |
|------|---------|-------------|
| `--num-shards` | 2 | 独立 CRTC shard 数 |
| `--num-event-workers-per-shard` | 4 | 每个 shard 用于 KV 事件处理的 OS 线程数 |
| `--prefix-depth` | 2 | 用于计算 branch 路由 key 的 hash 块数 |

### `anchor-aware-branch-sharded-crtc` 参数

| 参数 | 默认值 | 作用 |
|------|---------|-------------|
| `--num-shards` | 2 | 独立 CRTC shard 数 |
| `--num-event-workers-per-shard` | 4 | 每个 shard 用于 KV 事件处理的 OS 线程数 |
| `--prefix-depth` | 2 | 派发到某 shard 前的最大路由 trie 深度 |

注意：`anchor-aware-branch-sharded-crtc` 不支持近似剪枝（approximate pruning）。它对乱序事件提供更强的路由正确性保证，代价是更高的路由延迟，并且在主导前缀工作负载下存在热 branch shard 坍缩风险（见 Known Issues）。

---

## 结果

Trace：`mooncake_trace.jsonl`（Mooncake FAST25 arxiv trace）。配置：branch-sharded 2 shard × 4 worker；CRTC 基线 8 worker；`--trace-simulation-duration-ms 10000`、`--benchmark-duration-ms 30000`、`-d 7`。

### 稳态 —— 在 trace 请求速率（约 17,268 ops/s）下的 p99

| Indexer | Achieved ops/s | p99 | 路由结果 | Avg routing | Avg shard |
|---------|---------------|-----|-----------------|-------------|-----------|
| CRTC 基线（8w） | 17,201 | 1,093 µs | — | — | — |
| Branch-sharded depth=2（2×4w） | 17,182 | **933 µs** | 0.0% miss | 762 ns | 238 µs |
| Branch-sharded depth=4（2×4w） | 17,221 | **841 µs** | 0.0% miss | 717 ns | 237 µs |
| Anchor-aware BSI depth=2（2×4w） | 17,064 | 1,510 µs | 91.3% dispatched / 8.7% shallow | 388 µs | 161 µs |

所有 indexer 都跟得上 offered trace 速率。在该 arxiv trace 上，branch-sharded depth=2 的 p99 比 CRTC 低约 **15%**，depth=4 低约 **23%**。这是适度的延迟收益，并不像在更小的 `conversation_trace.jsonl` 工作负载上那样大；不要直接对比这两组结果。

这些运行中的真实 miss 率实际为零。

Anchor-aware BSI 在该稳态运行中比 CRTC 与 branch-sharded 都慢。它的路由 TRIE 提供了更强的结构化路由模型，但在该热前缀 trace 上路由工作量明显更大，且 shard 负载几乎完全坍缩到一个 shard。

> **Shard 不平衡警告（该 trace）：** `mooncake_trace.jsonl` 包含两个主导 depth-2 前缀：`[46,47]` 出现 9,203 次，`[74,75]` 出现 3,449 次。Branch-sharded 路由因此即便没有坍缩到单 shard，也仍然偏斜。这些数字对于正确性与热前缀性能很有用，但并不能展现理想的多 shard 扩展性。

Shard block 分布：
```text
branch depth=2:  shard 0: 1,919,603 blocks (87.0%), 7,000 workers  shard 1: 286,468 blocks (13.0%), 7,000 workers
                 branches: shard[0]=819, shard[1]=3

branch depth=4:  shard 0: 1,919,603 blocks (87.0%), 7,000 workers  shard 1: 286,468 blocks (13.0%), 7,000 workers
                 branches: shard[0]=960, shard[1]=17

anchor-aware:    shard 0: 574 blocks (0.0%), 14 workers  shard 1: 2,094,155 blocks (100.0%), 7,000 workers
```

把 `prefix_depth` 从 2 调到 4 会产生更多 branch 别名，但已存储 block 的拆分没有变化，因为最热 branch 的生命周期仍主导整个 trace。详见下文 Known Issues。

### 峰值吞吐扫描 —— `mooncake_trace.jsonl`

干净峰值定义为 benchmark 仍能跟上 offered 块吞吐时所达到的最高 ops/s。带 ⚠ 的行属于过载状态，不应作为干净吞吐上限。Sweep 对多线程 indexer 是先跑最短窗口，因此 trace 速率下的 p99 请使用上文独立的稳态运行。

| Indexer | 干净峰值 achieved ops/s | 干净峰值时的 p99 | 备注 |
|---------|--------------------------:|-------------------|-------|
| CRTC 基线（8w） | **118,157** | 1,087 µs | 在该热前缀 trace 上干净吞吐最佳 |
| Branch-sharded depth=2（2×4w） | 73,304 | 1,337 µs | 干净峰值低于 CRTC，因为流量偏斜到一个 shard |
| Anchor-aware BSI depth=2（2×4w） | 16,699 | 12,332 µs | sweep 在过载后明显劣化；正常 p99 请使用稳态运行 |

在该 arxiv trace 上，CRTC 的 sweep 干净吞吐最高。这与稳态表并不矛盾：branch-sharded 在 trace 速率下 p99 仍然更低，但热前缀分布限制了 shard 并行度，从而抬不起干净峰值吞吐。这也正是 arxiv trace 更适合作为主要 benchmark 的主要原因：它在更不利、更现实的热前缀工作负载下暴露准确性修复。

完整 sweep 数据：

**CRTC 基线：**

| Benchmark 窗口 | Offered ops/s | Achieved ops/s | p99 |
|-----------------|--------------|----------------|-----|
| 30,000 ms | 17,268 | 17,219 | 1,027 µs |
| 18,455 ms | 28,071 | 27,955 | 741 µs |
| 11,352 ms | 45,635 | 45,309 | 771 µs |
| 6,983 ms | 74,187 | 73,298 | 683 µs |
| 4,296 ms | 120,589 | 118,157 | 1,087 µs |
| 2,643 ms ⚠ | 196,008 | 160,615 | 1,281 µs |
| 1,626 ms ⚠ | 318,603 | 178,812 | 1,236 µs |
| 1,000 ms ⚠ | 518,049 | 164,820 | 1,511 µs |

**Branch-sharded depth=2：**

| Benchmark 窗口 | Offered ops/s | Achieved ops/s | p99 |
|-----------------|--------------|----------------|-----|
| 30,000 ms ⚠ | 17,268 | 13,732 | 17,772 µs |
| 18,455 ms | 28,071 | 27,967 | 1,967 µs |
| 11,352 ms | 45,635 | 45,089 | 814 µs |
| 6,983 ms | 74,187 | 73,304 | 1,337 µs |
| 4,296 ms ⚠ | 120,589 | 73,655 | 6,186 µs |
| 2,643 ms ⚠ | 196,008 | 113,628 | 2,663 µs |
| 1,626 ms ⚠ | 318,603 | 156,775 | 1,383 µs |
| 1,000 ms ⚠ | 518,049 | 117,445 | 2,390 µs |

**Anchor-aware BSI depth=2：**

| Benchmark 窗口 | Offered ops/s | Achieved ops/s | p99 |
|-----------------|--------------|----------------|-----|
| 30,000 ms | 17,268 | 16,699 | 12,332 µs |
| 18,455 ms ⚠ | 28,071 | 19,965 | 16,423 µs |
| 11,352 ms ⚠ | 45,635 | 23,088 | 14,922 µs |
| 6,983 ms ⚠ | 74,187 | 25,232 | 13,322 µs |
| 4,296 ms ⚠ | 120,589 | 26,185 | 11,384 µs |
| 2,643 ms ⚠ | 196,008 | 29,912 | 9,832 µs |
| 1,626 ms ⚠ | 318,603 | 35,720 | 6,970 µs |
| 1,000 ms ⚠ | 518,049 | 37,076 | 6,743 µs |

⚠ = bench 警告说跟不上 offered 速率。

### Worker 扩展 —— branch-sharded depth=2

测试随 trace 量增长 p99 与 shard 平衡如何变化。配置：2 shard × 4 worker/shard、`--num-unique-inference-workers 1000`、`-d 7`（7 副本 × 1,000 worker = 7,000 个 worker 身份）、`--benchmark-duration-ms 30000`。

注意：`--trace-duplication-factor` 复制 request/hash 空间，但 worker 身份数始终保持 7,000。它增加 branch 多样性与事件量；不会扩大 worker 身份数量。

| Trace 复制 | Branches | Offered ops/s | Achieved ops/s | Avg routing | Avg shard | p99 | Block 拆分 |
|------------------:|---------:|--------------:|---------------:|-------------|-----------|-----|-------------|
| 1× | 822 | 17,268 | 16,942 | 4.3 µs | 419 µs | 9,069 µs | 87.1% / 12.9% |
| 2× | 1,640 | 34,536 | 33,623 | 4.3 µs | 531 µs | 9,989 µs | 45.5% / 54.5% |
| 4× ⚠ | 3,306 | 69,073 | 56,049 | 5.2 µs | 488 µs | 8,403 µs | 54.2% / 45.8% |
| 8× ⚠ | 6,587 | 138,147 | 61,306 | 4.8 µs | 439 µs | 6,563 µs | 29.4% / 70.6% |
| 16× ⚠ | 13,162 | 276,295 | 106,127 | 2.0 µs | 262 µs | 2,859 µs | 44.3% / 55.7% |
| 32× ⚠ | 26,328 | 552,590 | 99,795 | 4.9 µs | 263 µs | 2,549 µs | 55.4% / 44.6% |

**1× 与 2× 跟得上，但在这种长时间高压序列下 p99 噪声较大。** 普通的 trace 速率延迟对比请使用独立的稳态运行。

**复制提升 branch 多样性。** 2×、4×、16×、32× 下 block 拆分都比单副本 arxiv 运行更接近平衡。8× 仍偏斜，因为即便 branch 数变多，其生命周期仍不均匀。

**4× 及更高在该机器上、30s 窗口下属于过载。** 这些行有压力形态参考价值，不能作为干净吞吐上限。当 achieved 块吞吐落后于 offered 时，benchmark driver 会发出警告。

**实践含义：** 对该 arxiv trace 与 2 shard 配置，仅靠 branch 多样性不能保证干净扩展。热前缀分布与生命周期偏斜仍然重要；更高压运行需要更多 shard、更长 benchmark 窗口或更小 trace 量，才能把 indexer 的能力上限与 benchmark driver 的施压上限分开。

### 反复过载 benchmark

配置：`--num-unique-inference-workers 128`、`--trace-duplication-factor 20`、`--trace-length-factor 4`、`--benchmark-duration-ms 750`、`--benchmark-runs 20`。这些运行产生 2,163,500 个事件：1,007,958 个 `Stored`，1,155,542 个 `Removed`。每次运行都警告 benchmarker 跟不上，macOS 在事件生成期间还输出过 malloc range-group 警告。请把它视为饱和压力数据，而非干净容量。

| Indexer | 事件 worker | Offered ops/s | Achieved ops/s p50 | Achieved ops/s p90 | Achieved ops/s p99 | Achieved ops/s 均值 |
|---------|--------------:|--------------:|------------------:|------------------:|------------------:|-------------------:|
| CRTC 基线 | 8 | 3,514,213 | 1,897,506 | 2,712,728 | 2,773,248 | 1,897,265 |
| Branch-sharded depth=2 | 2×4 | 3,514,213 | 423,942 | 451,508 | 457,397 | 411,244 |
| Anchor-aware BSI depth=2 | 2×4 | 3,514,213 | 594,934 | 664,920 | 703,730 | 593,921 |

| Indexer | Offered block ops/s | Achieved block ops/s p50 | Achieved block ops/s p90 | Achieved block ops/s p99 | Achieved block ops/s 均值 |
|---------|--------------------:|------------------------:|------------------------:|------------------------:|-------------------------:|
| CRTC 基线 | 73,600,216 | 39,740,576 | 56,814,248 | 58,081,760 | 39,735,535 |
| Branch-sharded depth=2 | 73,600,216 | 8,878,863 | 9,456,199 | 9,579,526 | 8,612,927 |
| Anchor-aware BSI depth=2 | 73,600,216 | 12,460,051 | 13,925,808 | 14,738,639 | 12,438,834 |

| Indexer | p99 latency p50 | p99 latency p90 | p99 latency p99 | p99 latency 均值 |
|---------|----------------:|----------------:|----------------:|-----------------:|
| CRTC 基线 | 138 µs | 312 µs | 380 µs | 158 µs |
| Branch-sharded depth=2 | 137 µs | 161 µs | 198 µs | 137 µs |
| Anchor-aware BSI depth=2 | 352 µs | 484 µs | 560 µs | 370 µs |

在该过载运行中，branch-sharded 的 lookup 延迟与 CRTC 相当：路由平均约 1.2-2.6 µs，shard 遍历平均约 9-16 µs，块在两个 shard 间的放置仍较均衡。因此较低的 achieved ops/s 并非 lookup 延迟回退。该命令主要由事件处理（尤其是 `Removed` 事件）主导；branch-sharded 在该路径上多了路由表与 block-to-shard 簿记，而 CRTC 直接落事件。

Anchor-aware BSI 在该过载运行中 achieved ops/s 介于 CRTC 与 branch-sharded 之间，但 p99 lookup 延迟更高。路由平均约 18-27 µs，shard 遍历约 12-19 µs，已存储 block 仍严重坍缩到一个 shard（通常 shard 1 占约 93-100%），所以这并不是平衡分片的结果。

---

## 已知问题（Known Issues）

### 共享前缀的 shard 坍缩与生命周期 block 偏斜

`assign_shard` 把活跃 block 数作为主要负载指标，因此 branch 放置会随观测负载自适应。仍存在两种偏斜模式：

1. **共享前缀坍缩：** 若大量请求共享相同的前 `prefix_depth` 个块，它们会共享同样的路由别名，落到同一个 shard。在 `mooncake_trace.jsonl` 上，两个最热 depth-2 前缀占 23,608 个请求中的 12,652 个，branch-sharded depth=2/depth=4 把 87% 的 block 放在了同一个 shard。
2. **生命周期偏斜：** 即使 branch 数分布均匀，由于 branch 一旦放置就是永久的，部分对话会比其他对话累积更多 block。久而久之，少量"重 branch"会主导某个 shard。

热 shard 的更大树会拉高 p99。

**未来工作 —— 再平衡：** 周期性地把过载 shard 上的"重 branch"迁移到较轻的 shard。这能直接缓解整 branch 可整体迁移时的生命周期偏斜。但它无法完全解决共享前缀坍缩（见下文 `prefix_depth` 段落）——那种情况下路由 key 本身太粗，需要类似按节点深度路由这样的结构性修复。

### `prefix_depth` 必须按工作负载调优

如果大多数请求共享一个长 prompt 前缀，所有对话可能都 hash 到相同的前 `prefix_depth` 块 → 同一个 branch key → 一个 shard 拿走大部分流量。把 `prefix_depth` 设为覆盖共享前缀再加至少 1–2 个唯一块。

注意：`anchor-aware-branch-sharded-crtc` 避免了 FNV 路由 key 冲突（拥有相同前缀的不同对话仍会得到不同的 TRIE 路径），但它在主导前缀工作负载下有自己的热 branch 坍缩问题（见下文）。两个变体都不是无条件更优的——无论你用哪个，都需要调 `prefix_depth`。

### Anchor-aware BSI：热 branch shard 坍缩

`AnchorAwareBranchShardedIndexer` 使用静态 divergent-shard 分配：在新父节点下的第一个对话留在父节点的 shard 上；只有后续的 divergent 兄弟会被 hash 到不同 shard。在主导共享前缀的工作负载上，几乎所有流量都会成为同一个 TRIE 节点的"第一个子节点"，从而路由到同一个 shard。在 `mooncake_trace.jsonl` 上观察到：稳态运行中已存储 block 实际 100% 集中在 shard 1。

代码中有一条关于自适应热 branch 拆分的 open TODO。在解决之前，请在前缀多样性较窄的 trace 上对比两种 branch-sharded 变体后再选择一种用于生产。
