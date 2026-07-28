# 测试模块

kvbm-engine crate 的测试基础设施。核心 block 与 token 工具从 `kvbm_logical::testing` 与 `kvbm_physical::testing` 重新导出；本模块在此之上为传输层、会话、卸载（offload）流水线以及多实例场景增加 engine 特定的辅助工具。

## 测试辅助工具

### TestManagerBuilder / TestRegistryBuilder

使用合成的物理布局创建测试用的 block manager 与 registry。`TestManagerBuilder` 生成一个由 mock 内存支撑的 `BlockManager<T>`。`TestRegistryBuilder` 生成一个预填充哈希值的 `BlockRegistry`。

可使用 `populate_manager_with_blocks` 与 `create_and_populate_manager` 快速搭建带预分配 block 的 manager 用于测试。

### MessengerPair

创建一对相互连接的 Velo `Messenger` 实例，用于在没有真实网络的情况下进行传输测试。通过其中一个 messenger 发送的消息会被另一个收到，从而可以在单进程内进行端到端的会话测试。

```rust,ignore
let (messenger_a, messenger_b) = create_messenger_pair_tcp().await?;
```

### TestSession

用于测试分布式会话协议的辅助类。它会搭建完整的会话基础设施（dispatch map、传输、通道）以测试 `InitiatorSession` / `ResponderSession` / `ControllableSession` 之间的交互。

### EventsPipelineFixture

卸载流水线的测试夹具（fixture）。提供预先配置好的流水线阶段、event manager 与 block manager，用于隔离测试策略评估、batching 以及传输执行。

### MultiInstancePopulator

搭建带有多个 leader、worker 与 block manager 的多实例分布式测试场景。它会用可配置的 block 模式填充每个实例，用于测试跨实例的 onboarding。

```rust,ignore
let populated = MultiInstancePopulator::builder()
    .instance_count(3)
    .blocks_per_instance(100)
    .build()?
    .populate()
    .await?;
```

### 物理层测试工具

`TestAgent` 与 `TestAgentBuilder` 用于创建模拟的 `NixlAgent` 实例，从而在无需真实 RDMA 硬件的情况下测试 `TransferManager`。`TransferChecksums` 提供用于验证传输正确性的工具。

### Token Block 辅助工具

`token_blocks` 模块提供了创建带有已知 token 序列的测试 block 的工具，便于验证查找与匹配操作。

## 编写一个新测试

1. 选择适合测试范围的 fixture：
   - 单实例传输 → `TestManagerBuilder` + `TestAgent`
   - 会话协议 → `TestSession` + `MessengerPair`
   - 卸载流水线 → `EventsPipelineFixture`
   - 多实例 → `MultiInstancePopulator`
2. 构建 fixture 并填充测试数据
3. 触发被测代码
4. 对结果进行断言并验证清理（block 已释放、会话已关闭）
