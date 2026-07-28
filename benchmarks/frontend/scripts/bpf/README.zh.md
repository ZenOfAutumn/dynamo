<!-- SPDX-License-Identifier: Apache-2.0 -->

# BPF Tracing 脚本

用于对 Dynamo frontend 进行低开销内核级追踪的 eBPF/bpftrace 脚本。这些脚本附加在内核 tracepoint 与 kprobe 上，无需修改应用就能测量调度、syscall、TCP 与上下文切换行为。

## 准备工作

```bash
# Full setup: install bpftrace, configure kernel, grant capabilities
sudo bash setup.sh

# Or step by step:
sudo bash setup.sh --install    # install bpftrace
sudo bash setup.sh --kernel     # set perf_event_paranoid=-1, kptr_restrict=0
sudo bash setup.sh --caps       # grant capabilities (run bpftrace without sudo)

# Check current state
sudo bash setup.sh --check

# Undo everything
sudo bash setup.sh --reset
```

授权 capability 之后，bpftrace **无需 sudo** 即可运行。

## 快速开始

```bash
# Get the frontend PID from a running capture
FRONTEND_PID=$(pgrep -f "dynamo.frontend" | head -1)

# Run a single script
./run.sh --pid $FRONTEND_PID offcputime

# List all available scripts
./run.sh --list

# Check if BPF environment is ready
./run.sh --check
```

## 脚本

| 脚本 | 测量内容 | 是否绑定 PID？ |
|--------|-----------------|----------------|
| `offcputime.bt` | 被切下 CPU（>1ms）的线程的内核栈。展示线程为何阻塞（futex、epoll、I/O）。 | 是 |
| `syscall_latency.bt` | 慢 syscall（>10us）按 syscall ID 统计，过滤到 Tokio worker。 | 是 |
| `runqlat.bt` | 调度器 run-queue 延迟 —— 唤醒后线程等待被调度的时长。 | 是 |
| `context_switches.bt` | 每个线程的上下文切换频率与开销。 | 是 |
| `cpudist.bt` | 每个线程的 on-CPU 时长分布（被抢占前持续运行的时间）。 | 是 |
| `funclatency.bt` | 特定内核/用户态函数的延迟直方图（模板 —— 需修改 probe）。 | 是 |
| `transport_latency.bt` | 通过 syscall tracepoint 测量 socket read/write 延迟。 | 是 |
| `tcplife.bt` | TCP 连接生命周期 —— 暴露浪费建连开销的短连接。 | 否（系统范围） |
| `tcpretrans.bt` | TCP 重传事件。 | 否（系统范围） |

## frontend 分析建议顺序

**1. 先做 off-CPU 分析** —— 找出 Tokio worker 为何阻塞：
```bash
bpftrace -p $FRONTEND_PID offcputime.bt
```
关注 `futex_wait_queue`（mutex 争用）、`ep_poll`（正常 I/O）、`schedule_timeout`（定时器）。

**2. syscall 延迟** —— 找出昂贵的 syscall：
```bash
bpftrace -p $FRONTEND_PID syscall_latency.bt
```
平均延迟较高的 `futex` = 锁争用。`writev` = TCP 发送开销。

**3. run-queue 延迟** —— 检查线程是否被 CPU 饿死：
```bash
bpftrace -p $FRONTEND_PID runqlat.bt
```
p99 > 1ms 表示 CPU 争用在拉高尾延迟。

**4. TCP 连接生命周期** —— 验证连接复用：
```bash
bpftrace tcplife.bt
```
大量到 localhost 的短连接（< 100ms）= 连接池优化机会（Part 3 瓶颈）。

**5. 上下文切换** —— 量化调度开销：
```bash
bpftrace -p $FRONTEND_PID context_switches.bt
```

## 结果解读

### Off-CPU 栈
```
@blocked_us[tokio-runtime-w,
    futex_wait_queue        ← mutex/condvar contention
    futex / do_futex
    __x64_sys_futex
    entry_SYSCALL_64
]: [1ms, 10ms) = 4812
```
- `futex_wait_queue` → mutex 阻塞（检查 Prometheus registry、TCP endpoint table）
- `ep_poll` → epoll_wait（正常 Tokio I/O 循环 —— 健康）
- `schedule_timeout` → 定时器/sleep
- `do_wait` → join handle 或 channel receive

### syscall 延迟
```
@slow[futex]: count=6504, avg=110ms, total=717s
```
`futex` 总耗时高 = 锁争用占主导。请与 off-CPU 栈交叉印证。

### TCP 生命周期
```
PID   COMM        LADDR     LPORT  RADDR     RPORT  TX_KB  RX_KB  MS
12345 tokio-run   127.0.0.1 43210  127.0.0.1 8081   0      128    45
```
大量到 mocker 端口、生命周期 < 100ms 的连接 = 没有连接池。

## 目录布局

```
bpf/
├── run.sh          # Script runner with capability detection
├── setup.sh        # Install bpftrace, configure kernel, grant caps
├── README.md
└── traces/         # bpftrace probe scripts (.bt files)
    ├── runqlat.bt
    ├── cpudist.bt
    ├── offcputime.bt
    ├── funclatency.bt
    ├── transport_latency.bt
    ├── tcplife.bt
    ├── tcpretrans.bt
    ├── syscall_latency.bt
    └── context_switches.bt
```

## 与 capture 脚本的集成

主 capture 脚本可以自动运行 BPF 追踪：
```bash
# Include BPF traces in the full capture (requires root)
sudo bash benchmarks/frontend/scripts/full_observability_run_perf.sh \
  --skip-nsys --skip-perf \
  --model Qwen/Qwen3-0.6B --concurrency 64 --num-requests 4096

# Compare event planes with BPF tracing:
sudo bash full_observability_run_perf.sh \
  --skip-nsys --skip-perf \
  --model Qwen/Qwen3-0.6B --concurrency 64 --num-requests 4096 \
  --event-plane zmq

sudo bash full_observability_run_perf.sh \
  --skip-nsys --skip-perf \
  --model Qwen/Qwen3-0.6B --concurrency 64 --num-requests 4096 \
  --event-plane nats

# BPF output appears in artifacts/obs_<timestamp>/bpf/
```

## 依赖要求

- Linux 内核 >= 4.18（支持 BPF CO-RE）
- `bpftrace` >= 0.16
- root 权限或 `CAP_BPF + CAP_PERFMON + CAP_NET_ADMIN + CAP_SYS_PTRACE` capability
- 部分 kprobe 脚本需要 kernel headers：`apt install linux-headers-$(uname -r)`
