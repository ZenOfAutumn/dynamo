---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 路由概念
subtitle: Dynamo 路由器的成本模型、worker 选择与路由原语
---

本页解释 Dynamo 路由器如何评估 worker、选择目标 worker，以及它在请求路径中的位置。关于 CLI 参数和调优旋钮，参见[配置与调优](router-configuration.md)。

## KV 缓存路由

KV 缓存路由通过同时利用可复用的缓存状态和预估的活跃负载来分发请求，从而优化大语言模型推理。缓存复用减少了重复的 prompt 计算，而实时的 prefill 与 decode 负载核算则防止缓存丰富的 worker 过载。

KV 缓存复用给 LLM 服务的负载均衡带来了复杂性。虽然它能显著降低计算成本，但忽略 worker 各自 KV 状态的路由策略可能导致：
- 因 worker 选择不佳而错失缓存复用机会
- 因请求在各 worker 间分布不均而导致系统吞吐下降

## 成本计算

![请求 token 被哈希后，路由器利用 KV 索引器的前缀命中和 slot tracker 的活跃负载进行评估，随后选择成本最低的 worker。Worker 的 KV 事件通过 event plane 更新索引器。](../../assets/img/router-kv-routing-overview.jpg)

成本函数结合了两项与 worker 相关的预估：

- **Prefill 成本**：已分配给该 worker 的活跃 prompt 工作量，加上新到请求中未命中缓存的 prompt 工作量。设备（device）、主机（host）、磁盘（disk）以及共享缓存（shared-cache）的命中会按其配置的信用值降低此成本。
- **Decode 成本**：已分配给该 worker 的活跃 KV block，加上新到请求预计产生的 block。

```text
raw_prefill_blocks = active_prefill_blocks + incoming_prompt_blocks
adjusted_prefill_blocks = max(raw_prefill_blocks - overlap_credit_blocks, 0)
decode_blocks = active_decode_blocks + incoming_active_blocks
cost = prefill_load_scale * adjusted_prefill_blocks + decode_blocks
```

`overlap_credit_blocks` 综合了配置的设备、主机、磁盘和共享缓存信用值。当缓存丰富的 worker 存在过多活跃 prefill 负载时，`overlap_score_credit_decay` 可以削减其设备本地那部分信用。路由器会选择成本最低的合格 worker。关于确切的调优行为，参见[配置与调优](router-configuration.md#tuning-guidelines)。

### 活跃负载建模

prefill 与 decode 的预估既包含新到请求的负载，也包含已分配给各 worker 的活跃负载。

#### Prefill 负载建模

对于 prefill 负载，路由器通过从请求的输入 token 中减去该 worker 已缓存的前缀 token，来估算每个候选 worker 未命中缓存的 prompt 工作量。

默认情况下，该有效 prefill 负载会一直按全额计费，直到第一个输出 token 标志着 prefill 完成。使用 `--router-prefill-load-model aic` 时，路由器还会基于有效 ISL 和已缓存前缀长度，向 [AIConfigurator (AIC)](../../features/disaggregated-serving/aiconfigurator.md) 请求一个预期的 prefill 时长。活跃负载追踪器以最早的活跃 prefill 作为时间锚点，并将已流逝的时间应用到该 worker 聚合建模的 prefill 积压上。当最早的请求完成时，下一个活跃 prefill 成为新锚点。如果某个被建模的非锚点请求先完成，追踪器会调整锚点以保持上报负载的连续性。

![时间线展示 AIC 如何在聚合建模的积压上衰减活跃 prefill 负载，并与"在第一个输出 token 前按静态方式核算"进行对比。](../../assets/img/router-active-prefill-timeline.jpg)

此模型仅改变路由器侧的 prompt 负载核算；它不改变后端的批处理或执行。

#### Decode 负载建模

对于 decode 负载，路由器追踪分配给每个 worker 的活跃 KV block。默认情况下，这涵盖已分配给活跃请求的 prompt 侧 block，并在每个请求完成时释放它们。

启用 `--router-track-output-blocks` 后，路由器还会在生成跨越 block 边界时添加占位的输出 block。如果请求包含 `nvext.agent_hints.osl`，这些输出 block 会根据朝预期输出长度的进度获得一个分数权重。这个预期 OSL 代理值使得接近完成的请求贡献更少的未来 decode 负载。若没有预期 OSL，被追踪的输出 block 会一直按全额计权，直到请求完成。

关于启用这些模型的参数，参见[配置与调优](router-configuration.md)。

## Worker 选择

路由器选择成本最低的 worker。当 `router_temperature` 设置为非零值时，路由器会对归一化的成本 logits 使用 softmax 采样，从而在选择中引入随机性，这有助于负载分布。

在打分之前，路由器会先按请求允许列表（allow-list）、精确 pin、DP-rank 范围、所需 taint 以及繁忙阈值过载状态过滤候选 worker。关于这些硬性合格规则，参见[路由器过滤](router-filtering.md)。

当请求在 policy-class 队列中等待时，加权的[缺额轮询队列调度（Deficit Round Robin Queue Scheduling）](deficit-round-robin.md)会在 worker 打分运行之前选出要分发的物理 class。

## 使用 KV 缓存路由器

要启用 KV 缓存感知路由，按如下方式启动 frontend 节点：

```bash
python -m dynamo.frontend --router-mode kv
```

当 KV block 被创建或移除时，引擎会通知 Dynamo 路由器，路由器随即识别出拥有最佳匹配 block 的 worker，并据此路由流量。

要评估 KV 感知路由的收益，可用 `--router-mode random|round-robin` 与 KV 感知路由对比你工作负载的性能。

关于详细的 CLI 参数和高级配置选项，参见[配置与调优](router-configuration.md)。

## 基础路由

Dynamo 在从一个组件向另一个组件的 endpoint 发送请求时，支持多种路由策略。

首先，创建一个绑定到某个组件 endpoint 的客户端。这里我们获取一个绑定到 `VllmWorker` 组件 `generate` endpoint 的客户端。

```python
client = runtime.endpoint("dynamo.VllmWorker.generate").client()
```

随后你可以使用客户端类暴露的默认路由方法向 `VllmWorker` 组件发送请求。

- **随机路由（Random routing）**：默认策略，通过 `client.generate()` 或 `client.random()` 使用
- **轮询路由（Round-robin routing）**：通过 `client.round_robin()` 在可用 worker 间循环
- **直接路由（Direct routing）**：通过 `client.direct(input, component_id)` 显式指向特定 worker
- **最小负载路由（Least-loaded routing）**：通过 `--router-mode least-loaded` 路由到活跃连接最少的 worker
- **设备感知加权路由（Device-aware weighted routing）**：通过 `--router-mode device-aware-weighted`，使用 CPU/非 CPU 比例预算，并在选中的设备组内进行最小负载选择

KV 缓存路由采用直接路由，并配合一种特殊的 worker 选择算法。

关于 KV 路由器性能的基准测试，参见 [KV 路由器 A/B 基准测试指南](../../benchmarks/kv-router-ab-testing.md)。
关于自定义路由逻辑和高级模式，参见[路由模式](router-examples.md#routing-patterns)。

## 设备感知加权路由

`device-aware-weighted` 专为异构集群设计，即 CPU 与非 CPU worker 共享同一 endpoint 的场景。路由器不比较原始的在途（in-flight）计数，而是比较 CPU 组与非 CPU 组之间经能力归一化的负载，然后在胜出的组内选择负载最小的 worker。

```text
normalized_load = total_inflight(group) / (instance_count(group) x throughput_weight)
```

吞吐权重（throughput weight）对 CPU worker 为 `1`，对非 CPU worker 为 `DYN_ENCODER_CUDA_TO_CPU_RATIO`。这使得路由器能够按设备能力成比例地路由，而不是永久性地饿死较慢的设备。

当只存在一类设备时，该行为退化为标准的最小负载路由。

