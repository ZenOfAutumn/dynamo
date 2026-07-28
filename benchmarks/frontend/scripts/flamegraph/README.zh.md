<!-- SPDX-License-Identifier: Apache-2.0 -->

# 火焰图（Flame Graph）脚本

用于从 Dynamo 前端（frontend）生成 CPU、off-CPU 以及差分火焰图 SVG 的
脚本。每个脚本会自动检测可用的剖析工具并选择最优的工具。

## 脚本

| 脚本 | 作用 | 是否需要 root？ |
|--------|-------------|----------------|
| `cpu_flamegraph.sh` | On-CPU 采样火焰图。依次尝试 cargo-flamegraph、samply，最后回退到 `perf record` + flamegraph.pl/inferno。 | 否（但 `perf` 需要 `CAP_PERFMON` 或 `perf_event_paranoid=-1`） |
| `offcpu_flamegraph.sh` | 通过 BPF 生成 off-CPU 火焰图。展示线程因何阻塞：互斥锁、I/O、futex、socket 等待等。 | 是（BPF 需要 root 或 `CAP_BPF`） |
| `diff_flamegraph.sh` | 比较两份剖析数据的差分火焰图。红色=回归，蓝色=改进。 | 否 |

## 快速开始

```bash
# 从一次正在运行的采集中获取 frontend PID
FRONTEND_PID=$(pgrep -f "dynamo.frontend" | head -1)

# CPU 火焰图（采样 30s）
./cpu_flamegraph.sh --pid $FRONTEND_PID --duration 30

# Off-CPU 火焰图（线程被什么阻塞）
sudo ./offcpu_flamegraph.sh --pid $FRONTEND_PID --duration 30

# 差分：比较优化前/后
./diff_flamegraph.sh before.perf.data after.perf.data
```

## 工具优先级

`cpu_flamegraph.sh` 按以下顺序尝试工具：

1. **cargo-flamegraph** —— 最简单，一步生成 SVG（仅用于启动新二进制，不支持 `--pid`）
2. **samply** —— 生成兼容 Firefox Profiler 的 JSON（支持 `--pid`）
3. **perf record** + **flamegraph.pl** 或 **inferno** —— 最常见的回退方式

`offcpu_flamegraph.sh` 尝试：

1. **bpftrace** —— 内联 BPF 脚本捕获 sched_switch 调用栈
2. **bcc offcputime-bpfcc** —— BCC 工具回退

## 选项

所有脚本共享统一的选项风格：

| 选项 | 说明 | 默认值 |
|--------|-------------|---------|
| `--pid PID` | 附加到运行中的进程 | — |
| `--duration N` | 采集时长（秒） | 30 |
| `--output-dir DIR` | 输出目录 | `.` |
| `--freq HZ` | 采样频率（仅 CPU） | 99 |
| `--min-us N` | 最小 off-CPU 时间（微秒，仅 off-CPU） | 1000 |

## 结果解读

### CPU 火焰图
- 较宽的塔状结构 = 占用最多 CPU 时间的函数
- 关注 `tokio-runtime-worker` 线程中的热点路径
- 窄而深的栈 = 正常调用链；宽而扁 = 优化目标

### Off-CPU 火焰图
- `futex_wait_queue` → 互斥锁/条件变量竞争
- `ep_poll` → epoll_wait（正常的 Tokio I/O 循环）
- `schedule_timeout` → 定时器/sleep
- `tcp_sendmsg` / `tcp_recvmsg` → socket I/O 阻塞

### 差分火焰图
- **红色**帧表示变慢（回归）
- **蓝色**帧表示变快（改进）
- 宽度差异表示变化幅度

## 与采集脚本的整合

主采集脚本会自动从 `perf record` 数据生成火焰图：
```bash
sudo bash benchmarks/frontend/scripts/run_perf.sh \
  --skip-nsys \
  --model Qwen/Qwen3-0.6B --concurrency 64 --num-requests 4096

# 火焰图 SVG 会出现在 artifacts/obs_<timestamp>/perf/
```

## 依赖

- **CPU**：`perf`（`apt install linux-tools-$(uname -r)`）或 `cargo install flamegraph` 或 `cargo install samply`
- **Off-CPU**：`bpftrace` >= 0.16 或 `bcc-tools`
- **生成 SVG**：`cargo install inferno`（提供 `inferno-collapse-perf`、`inferno-flamegraph`、`inferno-diff-folded`）或 Brendan Gregg 的 [FlameGraph](https://github.com/brendangregg/FlameGraph) 脚本
