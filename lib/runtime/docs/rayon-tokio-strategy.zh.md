# Rayon-Tokio 集成策略

## 概述

本文介绍在 Dynamo 运行时（runtime）中将 Tokio 异步运行时与 Rayon 数据并行计算
能力结合起来的集成策略。其核心思想很简单：

- **Tokio** 处理 I/O 受限操作、等待与协调
- **Rayon** 处理 CPU 密集型操作与数据并行
- 多个异步任务可以并发地把不同类型的工作提交到共享的 Rayon 线程池

## 架构

```text
+---------------------------------------------------------------+
|                     Tokio Runtime                             |
|  +-----------+    +-----------+    +-----------+             |
|  | Async     |    | Async     |    | Async     |             |
|  | Task 1    |    | Task 2    |    | Task 3    |             |
|  |           |    |           |    |           |             |
|  | Receives  |    | Processes |    | Handles   |             |
|  | requests  |    | streams   |    | batches   |             |
|  +-----+-----+    +-----+-----+    +-----+-----+             |
|        |                |                |                   |
|        +----------------+----------------+                   |
|                         |                                    |
|                  tokio_rayon::spawn                          |
|                         |                                    |
+-------------------------+------------------------------------+
                          |
                          v
+---------------------------------------------------------------+
|                    Rayon Thread Pool                         |
|                                                               |
|  +----------------------------------------------------------+ |
|  |         Work-Stealing Thread Pool (N threads)           | |
|  |                                                          | |
|  |  +---------+  +-----------+  +------------------+       | |
|  |  | scope() |  | par_iter()|  | join()           |       | |
|  |  | tasks   |  | chunks    |  | computations     |       | |
|  |  +---------+  +-----------+  +------------------+       | |
|  |                                                          | |
|  |  All patterns share the same thread pool                | |
|  +----------------------------------------------------------+ |
+---------------------------------------------------------------+
```

## 何时使用 Tokio，何时使用 Rayon

### 使用 Tokio（async/await）的场景
- **等待 I/O**：网络请求、文件 I/O、数据库查询
- **任务协调**：channel、同步、信号
- **流式处理**：随时间间断到达的条目
- **资源池**：连接池、async 锁
- **服务编排**：管理组件生命周期

### 使用 Rayon（计算池）的场景
- **批处理**：所有数据都已就绪可并行处理
- **CPU 密集型工作**：每个条目计算耗时 >1ms
- **数据变换**：tokenize、序列化、压缩
- **并行算法**：矩阵运算、排序、查找
- **map-reduce 模式**：在大数据集上做聚合

### 决策阈值
- 当并行处理 **≥10 个条目** 时使用 Rayon
- 当 CPU 工作耗时 **>1ms** 时使用 `spawn_blocking`
- 条目间等待 **>100μs** 的操作交给 Tokio
- 当工作能 **吃满多个 CPU 核** 时使用 Rayon

### 开销考量
基于基准测试，从 Tokio 到 Rayon 的 async 桥接开销大致为：
- 小任务约 **~25μs 开销**（来自 channel 通信）
- 耗时 >2ms 的任务约 **~4% 开销**
- 耗时 >10ms 的任务开销 **可忽略**

要从 async 上下文中以最小开销使用 Rayon：
- **小任务（<100μs）**：直接在 Tokio 上运行
- **中等任务（100μs - 1ms）**：使用 `spawn_blocking` + `pool.execute_sync()`
- **大任务（>1ms）**：使用 `pool.execute()`，更方便

## 并发使用模式

关键洞见在于：多个异步任务可以并发地、用不同的并行模式使用同一个 Rayon 线程池。
Rayon 的 work-stealing 调度器（scheduler）会高效地分发工作，无论使用何种模式。

### 模式 1：scope 与 ParIter 并发

```rust,ignore
use std::sync::Arc;
use dynamo_runtime::compute::ComputePool;

async fn concurrent_compute_tasks(pool: Arc<ComputePool>) {
    // 任务 1：使用 scope 动态生成任务
    let task1 = tokio::spawn({
        let pool = pool.clone();
        async move {
            pool.execute_scoped(|scope| {
                // 根据运行时条件动态 spawn 任务
                for i in 0..num_tasks {
                    scope.spawn(move |_| {
                        expensive_computation(i)
                    });
                }
            }).await
        }
    });

    // 任务 2：使用并行迭代器做批处理
    let task2 = tokio::spawn({
        let pool = pool.clone();
        async move {
            pool.install(|| {
                // 以并行块的方式处理数据
                data.par_chunks(100)
                    .map(|chunk| transform_chunk(chunk))
                    .collect::<Vec<_>>()
            }).await
        }
    });

    // 任务 3：使用 join 实现二元并行
    let task3 = tokio::spawn({
        let pool = pool.clone();
        async move {
            pool.join(
                || compute_left_branch(),
                || compute_right_branch(),
            ).await
        }
    });

    // 三个任务并发运行，共享 Rayon 线程池
    let (r1, r2, r3) = tokio::join!(task1, task2, task3);
}
```

### 模式 2：流处理 + 批量计算

```rust,no_run
# use futures::StreamExt;
# use rayon::prelude::*;
# use std::sync::Arc;
# struct Data;
# fn process_item(_: &Data) -> i32 { 0 }
# async fn send_results(_: Vec<i32>) {}
# use dynamo_runtime::compute::ComputePool;
# use futures::stream::Stream;

/// 示例：用 CPU 密集型批量操作处理 async stream
async fn stream_with_compute(
    pool: Arc<ComputePool>,
    stream: impl Stream<Item = Vec<Data>>,
) {
    // 使用 for_each_concurrent 来正确消费 stream
    stream.for_each_concurrent(4, |batch| {
        let pool = pool.clone();
        async move {
            // 用并行迭代器处理批
            let result = pool.install(move || {
                batch.par_iter()
                    .map(|item| process_item(item))
                    .collect::<Vec<_>>()
            }).await.unwrap();

            // async I/O 发送结果
            send_results(result).await;
        }
    }).await;
}
```

### 模式 3：混合负载服务

```rust,ignore
/// 真实示例：具备混合负载的 LLM 服务
struct LLMService {
    runtime: Arc<Runtime>,
    tokenizer: Arc<Tokenizer>,
}

impl LLMService {
    async fn run(&self) {
        let pool = self.runtime.compute_pool()
            .expect("Compute pool required");

        // Tokenize 服务 —— 使用并行迭代器
        let tokenization_task = {
            let pool = pool.clone();
            let tokenizer = self.tokenizer.clone();
            tokio::spawn(async move {
                loop {
                    // async I/O：从网络接收一批
                    let texts = receive_tokenization_batch().await;

                    // CPU 受限：并行 tokenize
                    let tokens = pool.install(move || {
                        texts.par_iter()
                            .map(|text| tokenizer.encode(text))
                            .collect::<Vec<_>>()
                    }).await.unwrap();

                    // async I/O：发送结果
                    send_tokens(tokens).await;
                }
            })
        };

        // Embedding 服务 —— 使用 scope 做多阶段计算
        let embedding_task = {
            let pool = pool.clone();
            tokio::spawn(async move {
                loop {
                    // async I/O：接收请求
                    let request = receive_embedding_request().await;

                    // CPU 受限：多阶段并行计算
                    let embeddings = pool.execute_scoped(|scope| {
                        let mut text_emb = None;
                        let mut context_emb = None;

                        scope.spawn(|_| {
                            text_emb = Some(compute_text_embedding(&request.text));
                        });

                        scope.spawn(|_| {
                            context_emb = Some(compute_context_embedding(&request.context));
                        });

                        // scope 等待两者都完成
                        combine_embeddings(text_emb.unwrap(), context_emb.unwrap())
                    }).await.unwrap();

                    // async I/O：发送结果
                    send_embeddings(embeddings).await;
                }
            })
        };

        // 批量推理服务 —— 使用嵌套并行
        let inference_task = {
            let pool = pool.clone();
            tokio::spawn(async move {
                loop {
                    let batch = receive_inference_batch().await;

                    let results = pool.execute_scoped(|scope| {
                        let mut results = Vec::with_capacity(batch.len());

                        // 为每个条目 spawn 一个任务
                        for item in batch {
                            scope.spawn(move |s2| {
                                // 在每个任务内部，再用并行迭代器
                                let preprocessed = item.data
                                    .par_chunks(10)
                                    .map(|chunk| preprocess(chunk))
                                    .collect::<Vec<_>>();

                                // 在嵌套 scope 内可继续 spawn 子任务
                                let mut stages = vec![];
                                for p in preprocessed {
                                    s2.spawn(move |_| {
                                        stages.push(run_inference(p));
                                    });
                                }

                                results.push(merge_stages(stages));
                            });
                        }

                        results
                    }).await.unwrap();

                    send_inference_results(results).await;
                }
            })
        };

        // 所有服务并发运行，共享同一个计算池
        tokio::join!(tokenization_task, embedding_task, inference_task);
    }
}
```

## 工作原理：线程池共享

即使不同的 async 任务提交了不同类型的工作，Rayon 的 work-stealing 调度器仍可
保证高效的资源利用：

1. **工作队列**：每个 Rayon 线程都有一个本地双端队列（deque）
2. **本地执行**：线程优先执行自己的任务（LIFO，利于缓存局部性）
3. **work stealing**：空闲线程会从忙碌线程偷任务（从另一端 FIFO）
4. **互不干扰**：不同的并行模式（scope、par_iter）可以和平共处

这意味着：
- spawn 大量小任务的 `scope` 任务可以与处理大批量的 `par_chunks` 并存
- 线程池会自动在不同类型工作之间均衡负载
- 在使用同一池的不同 async 任务之间，无需手工协调

## 性能考量

### 线程池大小

```toml
[runtime]
# Tokio 线程：针对并发 async 任务做优化
num_worker_threads = 8  # 通常等于核数

# Rayon 线程：针对吃满 CPU 做优化
compute_threads = 4     # 通常为 核数 / 2，避免过订阅
```

### 避免过订阅（oversubscription）

总线程数 = Tokio worker + Rayon 线程 + 系统线程

**建议：** 总数控制在 ≤ 1.5 × 物理核数

### 监控池利用率

```rust,ignore
// 查看池的指标
let metrics = pool.metrics();
println!("Active tasks: {}", metrics.tasks_active());
println!("Average duration: {:.2}ms", metrics.avg_task_duration_us() / 1000.0);
println!("Slow tasks (>100ms): {}", metrics.slow_tasks());

// 如果一直过载或过空，相应调整池大小
if metrics.tasks_active() > pool.num_threads() * 2 {
    // 考虑增大 compute_threads
}
```

## 常见模式与最佳实践

### 推荐：先收集成批，再处理

```rust,ignore
// ✅ 好：收齐 async 条目后并行处理
let items = stream.take(100).collect::<Vec<_>>().await;
let processed = pool.install(|| {
    items.par_iter().map(|item| process(item)).collect()
}).await?;
```

### 不推荐：在紧凑循环里交替 async 与计算

```rust,ignore
// ❌ 差：async 与计算反复交替
for item in items {
    let data = fetch_data(item).await;  // Async
    let result = pool.execute(|| compute(data)).await?;  // Compute
    store_result(result).await;  // Async
}

// ✅ 好：批量化操作
let all_data = futures::future::join_all(
    items.iter().map(|item| fetch_data(item))
).await;

let all_results = pool.install(|| {
    all_data.par_iter().map(|data| compute(data)).collect()
}).await?;

futures::future::join_all(
    all_results.iter().map(|result| store_result(result))
).await;
```

### 推荐：用 Scope 做动态并行

```rust,ignore
// ✅ 好：当并行度无法在运行前确定时
pool.execute_scoped(|scope| {
    while let Some(work) = find_more_work() {
        scope.spawn(move |_| {
            process_work(work);
        });
    }
}).await?;
```

### 推荐：用 ParIter 做数据并行

```rust,ignore
// ✅ 好：当处理集合时
pool.install(|| {
    data.par_chunks(optimal_chunk_size())
        .map(|chunk| process_chunk(chunk))
        .reduce(|| initial_value(), |a, b| combine(a, b))
}).await?;
```

## 故障排查

### 现象：CPU 利用率低但延迟高
**原因**：Rayon 线程数对负载来说过少
**方案**：增大 `compute_threads` 配置

### 现象：系统响应迟钝
**原因**：线程过订阅
**方案**：减小总线程数（Tokio + Rayon）

### 现象：工作分布不均匀
**原因**：chunk 切分不当
**方案**：使用更小的 chunk，或用 `scope` 做动态调度

### 现象：死锁或挂起
**原因**：嵌套 `install()`，或在 Rayon 线程上执行了阻塞操作
**方案**：对简单任务使用 `execute()` 取代 `install()`

## 配置示例

### 高吞吐服务
```toml
# 大量并发请求，每请求计算量适中
[runtime]
num_worker_threads = 16
compute_threads = 8
compute_stack_size = "4MB"
```

### 批处理系统
```toml
# 并发任务少，每批计算量大
[runtime]
num_worker_threads = 4
compute_threads = 12
compute_stack_size = "8MB"
```

### 混合负载
```toml
# async I/O 与计算之间取得平衡
[runtime]
num_worker_threads = 8
compute_threads = 6
compute_stack_size = "2MB"
```

## 总结

Rayon-Tokio 集成为处理混合负载提供了一套强大的模型：

1. **Tokio** 管理 async I/O 与协调
2. **Rayon** 提供共享的计算线程池
3. 多个 async 任务可并发使用不同的 Rayon 模式
4. work-stealing 保证资源高效利用
5. I/O 密集与 CPU 密集工作之间界限清晰

该架构使我们可以构建在网络 I/O 与 CPU 密集计算之间都高效运行的高性能服务，
而无需手工管理线程或编写复杂的同步代码。
