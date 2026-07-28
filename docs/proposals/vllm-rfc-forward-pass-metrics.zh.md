# vLLM RFC：通过 ZMQ 暴露每次迭代的前向传播（Forward Pass）指标

> 用于提交到 https://github.com/vllm-project/vllm/issues/new?template=750-RFC.yml

---

## 标题

`[RFC]: Per-iteration forward pass metrics with accurate engine-level timing`

---

## 动机

**问题：编排系统需要每次迭代的调度器（scheduler）遥测，但 vLLM 只对外暴露聚合后的 Prometheus 指标（metrics）。**

推理（inference）编排系统（自动扩缩容器、router、解耦（disaggregated）服务规划器）需要理解一个运行中的 vLLM 引擎*每次迭代*的开销结构：

- 每个 batch 中预填充（prefill）请求和解码（decode）请求各有多少？
- 解码请求的 KV 缓存（KV cache）深度分布如何？
- 计算了多少 token、又有多少命中了缓存？
- GPU 前向传播实际花了多久？
- 有多少请求在排队等待？

如今 vLLM 暴露的是 Prometheus gauge / histogram 指标，由外部 collector **异步抓取（scrape）**。这对于每次迭代级别的遥测有根本性的局限：

1. **有损**：Prometheus 抓取是按可配置间隔的拉取式。在迭代时长 10–100ms 时，scraper 会漏掉 90%+ 的迭代。Gauge 值仅反映抓取那一刻的最近状态，而不是完整分布。聚合指标必然丢信息。

2. **不同步**：scraper 与引擎循环跑在各自的定时器上。来自不同 gauge 的指标可能反映不同的迭代，因此无法把同一个 batch 的 prefill/decode 数与 wall time 对齐起来。

3. **没有按迭代的历史**：无法重建随时间变化的 batch 组成序列。自动扩缩容器无法基于 Prometheus 数据构建代价模型，因为它只看得到快照。

4. **延迟**：基于推送的 Prometheus（Pushgateway）使用 HTTP，按抓取频率叠加延迟和开销。对于每秒 100+ 次迭代的逐迭代发射来说，这是不可承受的。

**为什么这对生态系统重要：**

- **NVIDIA Dynamo** 目前以 out-of-tree 的 `--scheduler-cls` 子类方式实现这一功能（[InstrumentedScheduler](https://github.com/ai-dynamo/dynamo/blob/main/components/src/dynamo/vllm/instrumented_scheduler.py)），但是从调度器侧测量 wall time 在本质上不准确，因为调度器无法观察到 GPU 前向传播的边界（见下文 Proposed Change）。
- **自动扩缩容器**（Kubernetes HPA、自定义规划器）需要每迭代的吞吐信号才能在数秒内（而不是数分钟）做出扩缩决策。

---

## 提议方案

### 1. 在 EngineCore 中增加 `wall_time` 测量

在精确的边界处测量 GPU 前向传播时间——围绕 `EngineCore.step()` / `step_with_batch_queue()` 中的 `future.result()`：

```python
# In EngineCore.step():
scheduler_output = self.scheduler.schedule()
future = self.model_executor.execute_model(scheduler_output, non_block=True)
...
t_start = time.monotonic()
model_output = future.result()   # blocks until GPU finishes
wall_time = time.monotonic() - t_start
...
self.scheduler.update_from_output(scheduler_output, model_output, wall_time=wall_time)
```

这是代码库中**唯一**能同时直接访问 GPU 等待边界与调度器输出的位置。调度器无法准确测量这个时间，因为：
- 同步模式下：`schedule()` 在 `execute_model` 运行之前就返回了
- 异步模式下：`schedule(N+1)` 与 GPU 上的 batch N 并发运行，调度器侧的时间戳会混入相邻 batch 的重叠

把 `wall_time` 作为新的可选 kwarg 传给 `update_from_output()`，让调度器把它纳入指标。

### 2. 定义一个每迭代指标 struct

每次前向传播只发射一次的紧凑、带版本的 struct：

```python
class ForwardPassMetrics(msgspec.Struct, frozen=True):
    version: int = 1             # can include more info in later versions

    # Identity
    worker_id: str = ""          # unique engine instance identifier
    dp_rank: int = 0             # data parallel rank
    counter_id: int = 0          # monotonic sequence number

    # Timing (measured in EngineCore)
    wall_time: float = 0.0       # seconds, GPU forward pass time

    # Scheduled batch composition
    num_prefill_requests: int = 0
    sum_prefill_tokens: int = 0       # tokens being computed this iteration
    var_prefill_length: float = 0.0   # variance of total prompt lengths
    sum_prefill_kv_tokens: int = 0    # KV tokens read (cache hits + prior chunks)
    num_decode_requests: int = 0
    sum_decode_kv_tokens: int = 0     # total KV depth across decode requests
    var_decode_kv_tokens: float = 0.0

    # Queue state
    num_queued_prefill: int = 0
    sum_queued_prefill_tokens: int = 0
    num_queued_decode: int = 0        # preempted requests waiting
    sum_queued_decode_kv_tokens: int = 0
```

**为什么选这些字段：**
- 自动扩缩容器需要 `wall_time` + `num_prefill_requests` + `num_decode_requests` + token 计数，才能构建 `latency = f(prefill_tokens, decode_batch_size, kv_depth)` 形式的代价模型。
- 方差字段可用于检测异质 batch（短序列与长序列混合），它们会影响 padding 开销和 CUDA graph 效率。
- 队列指标可用于负载感知路由与背压（backpressure）信号。
- `msgspec.Struct` 支持零拷贝序列化，且 vLLM 的 KV 缓存事件已在使用它。

### 3. 通过 ZMQ PUB/SUB 发射（而非 Prometheus）

把该 struct 通过绑定到可配置 localhost 端口的 ZMQ PUB socket、用 msgpack 序列化发布出去：

```
ZMQ message: [topic_bytes, sequence_bytes, msgpack_payload]
```

**为什么用 ZMQ 而不是 Prometheus：**

| | ZMQ PUB/SUB | Prometheus |
|---|---|---|
| **投递** | 推送，每次迭代 | 拉取，按 scraper 间隔 |
| **完整性** | 每次迭代都被捕获 | 90%+ 迭代被漏掉 |
| **关联性** | 同一迭代的所有字段在一条消息中 | 各 gauge 可能对应不同迭代 |
| **延迟** | 每条消息约 10us（IPC） | 每次抓取一次 HTTP 往返 |
| **CPU 开销** | 后台线程，非阻塞发送 | metric registry 锁竞争 |
| **消费者** | 多个 SUB socket，零拷贝 | 单个 scraper 端点 |
| **格式** | 带版本、有类型、可扩展（msgspec） | 扁平的 key-value gauge |

ZMQ publisher 跑在后台守护线程里（与 vLLM 现有的 `ZmqEventPublisher`（用于 KV 缓存事件）相同的模式）。调度器热路径只需在有界队列上 `queue.put_nowait()`——没有序列化、没有 I/O。

**向后兼容：Prometheus "最近值" gauge。** 对于只想通过现有 Prometheus 基础设施获取近似指标的用户，可以选择把最新的 `ForwardPassMetrics` 暴露为 Prometheus gauge（每次迭代就地更新，按 collector 选用的间隔抓取）。这种方式严格弱于 ZMQ 流，但能保持与现有监控仪表盘的兼容。

### 4. Data parallel 支持

每个 DP rank 跑自己的 EngineCore 与调度器。每个 rank 在 `base_port + dp_rank` 上绑定自己的 ZMQ PUB socket，发射独立的、带 `dp_rank` 标签的 FPM 流。

**Attention DP（非 MoE）：** 每个 rank 完全独立（本地 `dp_size=1`）。每个 rank 发射自己的 FPM 流。无需跨 rank 协调——消费者（自动扩缩容器、planner）独立订阅每个 rank 的 ZMQ 端口，并按需聚合。

**DP+EP（MoE）：** 每个 rank 有自己的调度器并发射自己的 FPM。尽管 GPU 前向传播跨 rank 通过 collective（`coordinate_batch_across_dp`）同步，每个 rank 的 `wall_time` 仍在本地的 `future.result()` 边界处测量。这些测量在跨 rank 上几乎一致（collective 强制同步），所以任意 rank 的数据都具代表性。消费者可以平均，也可以直接取 rank 0 的数据。

这与今天的 **KV 缓存事件**采取的是同一种做法：每个 DP rank 发布到自己的 ZMQ 端口，跨 rank 的中继/消费层在引擎之外处理多 rank 聚合。

### 5. 启用方式

通过新的引擎参数控制：

```
--forward-pass-metrics-port PORT   # 0 = disabled (default), >0 = ZMQ PUB base port
```

对于 DP 部署，rank N 在 `PORT + N` 上绑定。启用后，调度器基类（或一个轻量 mixin）负责指标抽取与 ZMQ 发布。无需子类覆盖——这一机制应在任何调度器实现上都能工作。

### 6. 线上格式与版本控制

- **序列化**：msgpack 通过 `msgspec.msgpack.Encoder`（与 KV 缓存事件相同）
- **ZMQ multipart**：`[b"", seq.to_bytes(8, "big"), msgpack_payload]`
  - 空 topic 为未来基于 topic 的过滤预留空间
  - 8 字节大端序号用于排序 / 缺口检测
  - msgpack payload 为序列化后的 `ForwardPassMetrics`
- **版本控制**：struct 中的 `version` 字段。消费者在解释字段前必须校验版本。不兼容变更要 bump 版本号。

### 7. 实现范围

| 组件 | 改动 |
|-----------|--------|
| `EngineCore.step()` / `step_with_batch_queue()` | 围绕 `future.result()` 测量 `wall_time`，并传给 `update_from_output()` |
| `Scheduler.update_from_output()` | 接受可选 `wall_time` kwarg |
| `SchedulerInterface` | 新增可选方法 `get_forward_pass_metrics()` 或 mixin |
| 新：`ForwardPassMetrics` struct | 放在 `vllm/v1/metrics/` 或 `vllm/v1/core/sched/` |
| 新：`FpmPublisher`（ZMQ 后台线程） | 仿照现有 `ZmqEventPublisher` |
| `AsyncEngineArgs` | 新增 `--forward-pass-metrics-port` 参数 |
| 可选：Prometheus stat logger | 把最新 FPM 字段暴露为 gauge |

---

## 反馈期

2 周。

---

## CC 列表

@simon-mo @youkaichao @WoosukKwon @robertgshaw2-redhat

---

## 其他事项

**参考实现：** NVIDIA Dynamo 的 [InstrumentedScheduler](https://github.com/ai-dynamo/dynamo/blob/main/components/src/dynamo/vllm/instrumented_scheduler.py) 以 out-of-tree 的调度器子类方式、用调度器侧时间戳实现了这一能力。把计时下沉到 EngineCore、把 ZMQ publisher 放到 vLLM 核心后，可以：

1. 不再需要为指标功能覆盖 `--scheduler-cls`
2. 提供精确的 GPU 计时（而非调度器近似）
3. 让任何编排系统（不仅 Dynamo）都能消费每迭代指标
4. 复用 KV 缓存事件已有的 ZMQ 基础设施

**vLLM 中已有的 ZMQ 先例：** KV 缓存事件系统（`KVEventsConfig`、`ZmqEventPublisher`）已经使用了完全相同的模式——本地 ZMQ PUB、msgpack 序列化、后台线程。前向传播指标会沿用同一架构。

**不在范围内：** 消费者（Dynamo、自定义 autoscaler 等）如何订阅、中继、聚合这些指标。这是消费侧逻辑。本 RFC 只覆盖 vLLM 端的发射。
