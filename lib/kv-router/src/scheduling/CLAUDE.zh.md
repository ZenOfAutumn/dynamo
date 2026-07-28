# lib/kv-router/src/scheduling

调度（scheduling）负责请求准入（admission）、排队、worker 选择，以及从预测负载到
序列状态预订的交接。

## 约束（Guardrails）

- `SchedulerQueue::admit_one` 是规范的准入路径：先计算预测负载，再选择 worker、
  返回响应，最后预订状态。常规调度不要绕过此路径。
- 不要移除或削弱 `admission_gate`，除非能证明选择加预订仍然足够串行化，从而避免
  超额订阅（oversubscription）回归。
- 潜在负载预测必须通过
  `ActiveSequencesMultiWorker::potential_blocks_and_tokens_at(...)` 并配合
  `SchedulingRequest::prefill_token_deltas()` 进行。不要直接从调度处扫描每个
  worker 的 `ActiveSequences`。
- `SchedulingRequest` 的辅助方法是有效缓存 token、有效重叠、worker 配额、预填充
  （prefill）token 默认值与请求 block 数量的规范来源。不要在策略或选择器中重复
  实现这些逻辑。
- WSPT 必须保持对预填充提示（prefill-hint）的感知：被 pin 的请求使用所 pin 的
  worker 的有效缓存 token；未被 pin 的请求使用最佳允许 worker。除非跟踪被关闭
  或缓存数据缺失，否则不要静默地回退到原始 ISL。
- Pinned-worker 与 allowed-worker 约束必须在选择前完成校验，并被队列容量检查、
  选择器候选迭代以及 WSPT 优先级所遵守。
- 预填充负载提示在调度器/请求边界上，从所选 worker 的 `cached_tokens` 计算得出。
  不要把 ISL/cache-token 计算搬回 `ActiveSequences`。
- 选择器（selectors）应当无副作用：不进行预订、不修改队列，也不修改
  `PromptRegistry`。
- 在选择、调用 slot 状态、响应或 await 时不要持有 pending-heap 锁。该队列堆仅
  用于被 park 的请求。
- 不要跨 `.await` 持有 `workers_with_configs.borrow()`；可以拍一个短暂的同步快照，
  或仅在选择期间借用。
- 任何对队列排序、WSPT key、容量检查、准入串行化或选择器评分的修改，都应当包含
  针对性测试以及前后对比的路由或队列基准。
- 文本和外部 ID（例如 request ID）请保留在标准哈希集合上。仅对内部数值类的热路径
  key 使用 `FxHashMap` / `FxHashSet`。
