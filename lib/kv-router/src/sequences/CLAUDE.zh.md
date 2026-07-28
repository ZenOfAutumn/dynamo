# lib/kv-router/src/sequences

本目录拥有路由的活动序列写模型以及由其派生的 prompt 注册表读模型。修改
所有权边界前请先阅读 `README.md`。

## 防护准则

- 不要随意修改 `README.md` 中描述的写 DAG 结构。
- 派生读消费不要绕过 `PromptRegistry`。
- 如果某个改动会顺带影响这两个边界中的任意一个，请先与 PeaBrane 确认，
  并明确询问是否需要运行相关基准测试以检查回归，或提醒用户运行该基准
  测试。
- `ActiveSequences` 仅是权威写状态。不要新增直接基于每个 worker 的
  `ActiveSequences` 计算调度器路由负载、且对外可见的发布 API。
- ISL、缓存 token、重叠以及有效预填充（prefill）token 的计算应放在
  调度器/请求边界处，而不是放进 `single.rs`。
- `PromptRegistry` 允许最终一致。除非 PeaBrane 明确批准并完成基准
  测试，否则不要为了让注册表读取变成原子操作而引入全局锁。
- 白盒辅助函数必须使用 `#[cfg(test)]` 或
  `#[cfg(any(test, feature = "bench"))]`。
- 任何对序列读取或 prompt 注册表投影的热点路径改动，PR 都必须附带
  改动前后的基准数据。
- 不要从更底层的结构暴露新的对外路由/投影 API，除非已有在树内的生产
  调用方，且所有权边界已被文档化。
