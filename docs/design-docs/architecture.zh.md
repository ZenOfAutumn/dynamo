---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 总体架构
subtitle: Dynamo 推理运行时的架构与组件
---

# Dynamo 架构

Dynamo 是面向生成式 AI 系统的分布式推理运行时，专为在变化的流量条件下提供高吞吐、低延迟与高可靠性而设计。它与具体后端无关（SGLang、TRT-LLM、vLLM 等均可），围绕三个相互配合的关注点构建：

- 一条用于 token 生成的快速 **请求路径（request path）**
- 一条用于扩缩容与放置的响应式 **控制路径（control path）**
- 一条用于 KV 复用与故障恢复的弹性 **状态路径（state path）**

本文将 Dynamo 作为一种架构来介绍，而不是一份功能清单：每个面（plane）各管什么、请求如何流动、系统如何自适应、以及在故障下如何保持正确性。

## 设计目标

Dynamo 的设计同时满足以下目标：

1. **延迟稳定性**：在突发流量与混合长度流量下，让 TTFT（首 token 时间）和 ITL（token 间隔）保持可预测。
2. **GPU 效率**：分离 prefill 与 decode，使两者能够各自独立扩缩容。
3. **计算复用**：通过 KV 感知路由和缓存生命周期管理，最小化 KV 重算。
4. **运维弹性**：把 worker 崩溃、重启与过载视为正常运行事件来处理。
5. **部署可移植性**：同时支持 Kubernetes 原生控制路径与非 Kubernetes 的运行模式。

## 为什么需要这套架构

现代 LLM 服务会反复遇到这几类瓶颈：

- **prefill/decode 不平衡**：当流量结构变化时，GPU 利用率会下降（[DistServe](https://arxiv.org/abs/2401.09670)）。
- **KV 重算**：当路由忽略缓存重叠时会拉高 TTFT 并浪费算力（[DeepSeek](https://arxiv.org/abs/2501.12948)）。
- **显存压力**：长上下文 + 高并发会超出 HBM 容量，必须依赖多层缓存管理（[KVBM](https://docs.nvidia.com/dynamo/components/kvbm)、[Mooncake](https://kvcache-ai.github.io/Mooncake/design/mooncake-store.html)、[AIBrix](https://blog.vllm.ai/2025/02/21/aibrix-release.html)、[FlexKV](https://github.com/taco-project/FlexKV)、[LMCache](https://lmcache.ai/)）。
- **动态需求**：会打破静态资源规划的假设（[AzureTrace](https://github.com/Azure/AzurePublicDataset)）。
- **真实世界故障**（pod 重启、网络分区、热点过载）：需要把恢复行为当作一等公民对待。

Dynamo 的应对方式是把服务（serving）、控制（control）、状态传播（state propagation）显式拆分到不同的面与控制循环中。

## 架构总览

![Dynamo architecture showing Request Plane (Client, Frontend, Router, Prefill/Decode workers), Control Plane (Planner, Dynamo Operator, Dynamo Graph, Grove, Model Express, Runtime Resources), and Storage &amp; Events Plane (KVBM, NIXL, Local SSD/NFS/Remote Storage)](../assets/img/dynamo-architecture.svg "Dynamo Architecture")

## 系统模型

### 请求面（关键数据路径）

请求面负责请求/响应的执行：

- **Frontend** 接受并标准化请求。
- **Router** 基于负载与 KV 重叠选择 worker。
- **Prefill worker** 计算提示词的 KV 状态。
- **Decode worker** 生成输出 token。

这条路径以低开销和持续 token 流式输出为优化目标。

### 控制面（自适应与编排路径）

控制面负责对“目标状态（desired state）”的管理：

- **Planner** 根据实时指标计算扩缩容目标。
- **Dynamo Operator** 基于 Dynamo CRD 调谐 Kubernetes 资源。
- **Discovery + Endpoints/CRD** 提供存活性与可发现性。
- **Grove/KAI Scheduler 路径** 在多节点 Kubernetes 部署中提供拓扑感知放置和分组扩缩容。
- **Model Express** 是可选的模型管理端点（按需启用）。

这条路径以正确性、向目标容量收敛为优化目标。

### 存储与事件面（状态传播路径）

存储/事件面负责缓存状态的可见性与流动：

- **KV Events** 发布缓存生命周期变化。
- **KVBM** 跨内存层级管理 block 复用、淘汰、卸载/回拉。
- **NIXL** 在 worker 之间、跨内存域执行高速 KV/数据传输。

这条路径以缓存复用、跨 worker 接力效率为优化目标。

## 端到端请求叙事（分离模式）

1. 客户端将请求发送到 **Frontend**。
2. Frontend 校验/预处理后转发给 **Router**。
3. Router 选择一个 **Prefill worker**。
4. Prefill 计算 KV 并返回传输元数据。
5. Router 选择一个 **Decode worker**。
6. Decode 接收 KV 状态（通常通过 **NIXL** 传输路径）。
7. Decode 通过 Frontend 流式返回 token。
8. **KV Events** 更新缓存可见性，供后续路由决策参考。
9. **KVBM** 可能根据压力与复用潜力决定卸载或回拉 KV block。

更细粒度的流程图请见 [Architecture Flow](dynamo-flow.md)。
请求传输方式的细节请见 [Request Plane](request-plane.md)。

## 控制循环

### 服务循环（Serving Loop）

在 frontend、router、prefill、decode worker 之间维持低延迟的请求执行。

### 规划循环（Planning Loop）

让容量与需求保持一致：

- Planner 消费运行时指标。
- Planner 计算 prefill/decode 目标。
- Connector 层把目标作用到运行时资源上。

Planner 同时支持基于吞吐与基于负载的策略。详见 [Planner Design](planner-design.md)。

### 韧性循环（Resilience Loop）

在故障下维持系统连续性：

- 健康检查识别不健康的 worker。
- 服务发现的存活性机制移除过期端点。
- 优雅关闭会排空在飞工作。
- 请求迁移/取消用于控制在飞行为。
- 过载时通过 load shedding 防止级联崩塌。

详见 [Fault Tolerance](../fault-tolerance/README.md)。

## Kubernetes 原生形态（CRD + Grove）

在 Kubernetes 部署中，同一套架构会映射为声明式资源：

- Dynamo Operator 调谐 `DynamoGraphDeployment`。
- 可发现性来自 `DynamoWorkerMetadata` + EndpointSlices。
- 由 Grove 支撑的多节点部署，会把 worker 组建模为 `PodCliqueSet` 与 `PodClique`。
- prefill/decode 的独立弹性通过 `PodCliqueScalingGroup` 表达，分别拥有各自的 `replicas` 和 `min` 目标。

图中标注的 `PodClique A/B`、`ScalingGroup "Prefill"`、`ScalingGroup "Decode"`、`(replicas, min)` 等就是这种分组扩缩容模型的体现。

## 容错架构

容错被嵌入到各个层面：

| 层面 | 机制 | 实际效果 |
|------|-----------|------------------|
| 请求 | 迁移、取消 | 在飞工作可以继续，也可以受控终止 |
| Worker | 健康检查、优雅关闭、端点排空 | 故障/正在终止的 worker 安全地停止接新流量 |
| 系统 | 请求拒绝 / load shedding | 防止过载在 worker 之间扩散 |
| 基础设施 | 服务发现租约过期、事件通路恢复 | 移除过期成员，流量重新路由 |

这一模型的前提是：故障是常态，不是异常。

## 性能依据

### 分离式服务

分离 prefill 与 decode 既能提升利用率，也能让两个阶段按各自需求扩缩容。

![Two scatter plots comparing the performance of disagg and baseline configurations on one node versus two nodes](../assets/img/disagg-perf-benefit.png)

*在 H100 上使用 R1 Distilled Llama 70B FP8（vLLM）测试，3K ISL / 150 OSL。*

### KV 感知路由

结合缓存重叠 + 负载信号进行路由，可减少 prefill 重算并降低延迟。
真实生产案例可参考：[How Baseten achieved 2x faster inference with NVIDIA Dynamo](https://www.baseten.co/blog/how-baseten-achieved-2x-faster-inference-with-nvidia-dynamo/#how-baseten-uses-nvidia-dynamo)。

![Two bar charts comparing Random routing and Dynamo with KV aware routing for Time To First Token (3x faster with Dynamo) and Avg request latency (2x faster with Dynamo).](../assets/img/kv-routing.png)

*在 2 台 H100 节点上使用 R1 Distilled Llama 70B FP8（vLLM）对 R1 发起 100K 次请求测试，平均 4K ISL / 800 OSL。*

### KV Block Manager（KVBM）

KVBM 通过多层内存的卸载/回拉来扩展有效缓存容量。

![Line graph comparing Pure GPU prefix caching with vLLM and KVBM host offloading for TTFT (Time To First Token)](../assets/img/kvbm-agg-performance.png)

*在 H100 上使用 Qwen3-8B 测试不同 QPS，平均 20K ISL / 100 OSL。*

### NIXL 数据传输

NIXL 通过优化跨 worker、跨异构内存域的传输行为，降低分布式服务下 KV 接力的成本。

## 实现模型

- **Rust** 用于对性能敏感的运行时组件。
- **Python** 用于后端集成与可扩展性。
- 子系统之间边界清晰，路由、规划、内存、传输等模块可独立演进。

## 相关文档

- [Architecture Flow](dynamo-flow.md)
- [Router Design](router-design.md)
- [Planner Design](planner-design.md)
- [Discovery Plane](discovery-plane.md)
- [Event Plane](event-plane.md)
- [Request Plane](request-plane.md)
- [Fault Tolerance](../fault-tolerance/README.md)
- [Grove](../kubernetes/grove.md)

## 致谢

Dynamo 受到了以下开源工作的启发：

- vLLM
- SGLang
- DistServe
- Mooncake
- AIBrix
- BentoML
