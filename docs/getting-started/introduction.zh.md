---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
sidebar-title: Introduction
---

# Dynamo 简介

Dynamo 是一个开源、高吞吐、低延迟的推理（inference）框架，旨在为分布式环境中的生成式 AI 工作负载提供服务。本页概述 Dynamo 的设计原则、性能优势以及生产级特性。

> [!TIP]
> 想立即上手？参见 [Quickstart](quickstart.mdx)，几分钟即可安装并运行 Dynamo。

## 为什么选择 Dynamo？

推理引擎优化 GPU；Dynamo 优化围绕 GPU 的整个系统。

- **在任意引擎之上做系统级优化** —— 推理引擎优化的是单 GPU 上的前向传播。Dynamo 增加分布式层：解耦（disaggregated）服务、智能路由（router）、跨内存层级的 KV 缓存（KV cache）管理，以及自动伸缩。
- **可组合的性能改进技术** —— 解耦服务、KV 缓存感知路由、KV 缓存卸载这些技术各自就能提升性能；同时使用还会获得复合收益。
- **与引擎无关** —— 可与 vLLM、SGLang 与 TensorRT-LLM 一起工作。无需更换服务基础设施即可切换引擎。Intel XPU 与 AMD 硬件的支持正在扩展中。
- **生产级、可大规模运行** —— Dynamo 覆盖完整部署生命周期：自动配置（AIConfigurator）、运行时（runtime）自动伸缩（Planner）、拓扑感知的 gang 调度（Grove）、容错与可观测性（observability）。
- **模块化采用** —— 可以从一个组件开始（例如只用 Router 在已有引擎之上做 KV 感知路由）。按需逐步采用更多组件。每个组件都可通过 pip 独立安装。

## 设计原则

### 为 AI 推理打下坚实基础

Dynamo 在推理引擎之上提供系统级优化。为做到这一点，Dynamo 采取操作系统式的思路，搭建调度（scheduling）、内存管理与数据传输的基础设施。这些基础使 Dynamo 能够随着新的系统级性能技术不断演进。

Dynamo 系统级设计的动机之一是支持解耦服务：在不同设备上运行 prefill 与 decode，使二者可独立伸缩与并行。解耦服务需要三种能力：(1) 调度，无干扰地分配 prefill 与 decode 阶段；(2) 内存管理，用于 KV 缓存的卸载与回填；(3) 低延迟数据传输，用于在节点（node）之间以及内存层级之间移动 KV 缓存。

![Dynamo 基础：调度、内存管理与数据传输](../assets/img/intro-foundations.svg)

Dynamo 的基础最初服务于解耦服务，随后扩展到多模态的 EPD 解耦，如今已支持扩散、RL 与 agent 等工作负载。

### 模块化但深度集成的生态

Dynamo 旨在降低替换生产中已有栈的负担。它以 Rust crate 与 pip wheel 的形式提供模块化、独立可用的组件。例如，Dynamo 的三大基础——调度（Dynamo）、内存管理（KV Block Manager）与数据传输（NIXL）——都可独立安装：

```bash
pip install ai-dynamo
pip install kvbm
pip install nixl
```

> [!NOTE]
> 也提供包含全部依赖的预构建容器。容器镜像参见 [Release Artifacts](../reference/release-artifacts.md)。

Dynamo 生态还包括以下其他模块化组件，并将持续扩展：

| 类别 | 产品 | 说明 |
| :--- | :--- | :--- |
| **调度** | Dynamo | 面向 GenAI 工作负载的推理服务 |
| **路由** | Router | 基于 KV 缓存命中率与 KV 缓存负载的智能路由。后续会加入更多算法（如 agentic routing） |
| **数据传输** | [NIXL](https://github.com/ai-dynamo/nixl) | GPU 之间以及与分层存储之间的点对点数据传输（G1：GPU；G2：CPU；G3：SSD；G4：远程） |
| **内存** | KVBM（KV Block Manager） | 跨 G1–G4 内存层级管理 KV 缓存，支持自定义淘汰策略 |
| **伸缩 / 云** | Planner | 在给定 SLA 约束（TTFT 与 TPOT）下，对 prefill 与 decode 实时自动调优 |
| | [Grove](https://github.com/ai-dynamo/grove) | 为 Kubernetes 多节点解耦服务提供必需的 gang 调度与拓扑感知 |
| | [Model Express](https://github.com/ai-dynamo/model-express) | 通过缓存并经 NIXL 向其他 GPU 传输模型权重，加速权重加载。后续也将用于容错 |
| **性能** | [AIConfigurator](https://github.com/ai-dynamo/aiconfigurator) | 基于模型、ISL/OSL、硬件等估算聚合服务与解耦服务的性能。前称 LLMPet |
| | [AIPerf](https://github.com/ai-dynamo/aiperf) | 用 Python 重新设计的 GenAI-Perf，扩展性极强；支持分布式基准测试 |
| | AITune | 给定模型或流水线，搜索最佳后端进行部署（如 TensorRT、Torch.compile 等）（即将推出） |
| | Flex Tensor | 把权重从主机内存流式送到 GPU，让显存有限的 GPU 也能运行非常大的语言模型（即将推出） |

这些组件相互独立但被设计为统一家族协同工作。新组件也将遵循同样的设计原则。

### 厂商无关的生态启用

Dynamo ***不是为厂商锁定而设计***。Dynamo 旨在赋能更广泛的 AI 生态，提供开发者所需的功能，例如与第三方组件的集成。

Dynamo 从一开始就被设计为支持所有主流 LLM 推理引擎（vLLM、SGLang 与 TensorRT-LLM）。后续将支持更多引擎以满足开发者使用场景。

**对非 NVIDIA 硬件的支持**也已具备：Dynamo 正与 Intel、AMD 等硬件厂商合作扩展硬件支持。

支持的生态组件完整列表：

| **产品领域** | **支持的生态组件** |
| :--- | :--- |
| 推理引擎 | SGLang、TensorRT-LLM、vLLM |
| Kubernetes | Inference gateway |
| 内存管理 | Dynamo KV Block Manager、[LMCache](../integrations/lmcache-integration.md)、[SGLang HiCache](../integrations/sglang-hicache.md)、[FlexKV](../integrations/flexkv-integration.md) |
| 网络与存储 | Mooncake、DOCA NetIO、GDS、POSIX、S3、3FS（[通过 NIXL 支持](../design-docs/kvbm-design.md)） |
| 多硬件 | Intel XPU、AMD |

## 性能

Dynamo 通过组合三项核心技术——解耦服务、KV 缓存感知路由、KV 缓存卸载——实现业界领先的 LLM 性能。这些技术的底层支撑是 NIXL，一个低延迟数据传输层，能在节点之间无缝迁移 KV 缓存。

- [KV 缓存感知路由](../design-docs/router-design.md) 基于 worker 负载与已有缓存命中智能地路由请求。通过复用预先计算好的 KV 对，可绕过 prefill 计算，立即进入 decode 阶段。[Baseten](https://www.baseten.co/blog/how-baseten-achieved-2x-faster-inference-with-nvidia-dynamo/#how-baseten-uses-nvidia-dynamo) 使用 Dynamo KV 缓存感知路由后，在 Qwen3 Coder 480B A35B 上获得了 2 倍更快的 TTFT 与 1.6 倍吞吐。

- [KV 缓存卸载](../design-docs/kvbm-design.md) 将 KV 缓存从 HBM 移动到更便宜的存储层级（如主机内存、本地磁盘、远程存储），扩展可用上下文窗口。复用预计算状态可改善 TTFT，降低 TCO，并支持更长上下文处理。

- [解耦服务](../design-docs/disagg-serving.md) 在“设计原则”一节我们介绍了解耦服务的概念。其性能在 [InferenceX](https://newsletter.semianalysis.com/p/inferencex-v2-nvidia-blackwell-vs) 中得到了展示：DeepSeek V3 借助解耦服务与大规模 expert parallelism，实现了约 7 倍的单 GPU 吞吐。
此外，将这三项技术组合在一起会产生复合收益，如下图所示。

![解耦服务、KV 缓存感知路由与 KV 缓存卸载的性能可组合性](../assets/img/intro-perf.svg)

- **解耦服务 + KV 缓存感知路由** —— KV 感知路由同时为计算（在 prefill 上）与内存（在 decode 上）做负载均衡，同时优化延迟与吞吐。
- **解耦服务 + KV 缓存卸载** —— KV 缓存卸载带来更快的 TTFT，并可减少 prefill worker 数量以降低 TCO。
- **KV 缓存感知路由 + KV 缓存卸载** —— 卸载提升可寻址缓存总量，提高 KV 缓存命中率，进而加速 TTFT。

> [!TIP]
> 想动手试试这些技术？参见 [Dynamo recipes](https://github.com/ai-dynamo/dynamo/tree/main/recipes)，那里有逐步可用的部署示例，组合了解耦服务、路由与卸载。

## 从配置到生产级部署

### 用 AIConfigurator 在 30 秒内找到最佳配置

手动寻找解耦服务的最佳并行方案可能耗费数日穷举式扫描——大规模时这一挑战只会更突出。

Dynamo 的 [AIConfigurator](https://github.com/ai-dynamo/aiconfigurator/) 可在 30 秒内找出表现最佳的配置，并清晰展示相对标准聚合服务的性能提升预期。该逻辑已原生集成到 Kubernetes Custom Resource Definition（CRD）—— Dynamo Graph Deployment Request（DGDR）—— 用户可使用自动生成的优化配置进行部署。

### 用 Planner 基于 SLA 自适应调整部署

通过 AIConfigurator 或 DGDR 找到离线配置后，开发者就可以将所需模型部署到生产中。然而生产流量在线时变化很大，离线确定的静态配置不足以应对流量峰值。

Dynamo 提供 [Planner](../design-docs/planner-design.md) 来解决这一问题。开发者只需以 TTFT 与 Time Per Output Token（TPOT）的形式设置 SLA。Planner 会观察在线流量并自动决策伸缩 prefill 与 decode worker，从而在保持 SLA 的同时有效应对流量峰值。

最近 Planner 被扩展以处理更复杂的场景，例如在相同 SLA 下输入序列长度（ISL）剧烈变化。详见 [Planner 文档](../components/planner/planner-guide.md)。

### 用 Grove 应用拓扑感知的分层 gang 调度

当 Planner 决定自动伸缩时，开发者需要一种方式来独立地、分层地伸缩 worker。尤其在 prefill/decode 解耦中，prefill 与 decode worker 需要独立伸缩以满足指定 SLA，并且彼此之间需要在物理上就近调度以获得最佳性能。

Dynamo 提供 [Grove](https://github.com/ai-dynamo/grove)，它是一个 Kubernetes operator，提供单一声明式 API，能从简单的单 pod 部署到复杂的多节点解耦系统编排任意 AI 推理工作负载。

Grove 支持：

- 分层 gang 调度
- 拓扑感知放置
- 多级水平自动伸缩
- 显式启动顺序
- 带可配置替换策略的滚动更新

这些特性对于在数据中心规模部署与伸缩推理以获得最佳性能至关重要。

### 为 LLM 提供容错

Kubernetes 自带一些容错能力，但 LLM 部署需要专门的容错与韧性。Dynamo 在多个层面提供完善的容错机制，确保生产环境下的 LLM 推理可靠：

- **Router 与 Frontend** —— Dynamo 支持启动多个 frontend + router 副本，通过共享路由状态提升容错。
- **请求迁移** —— 当 worker 在处理请求中失败时，Dynamo 可以将进行中的请求迁移到健康 worker，并保留部分生成状态，向客户端维持无缝 token 流。
- **请求取消** —— Dynamo 支持通过 AsyncEngineContext trait 取消进行中的请求，提供优雅的停止信号以及沿请求链分层向下传播的取消机制。
- **请求拒绝（负载卸除）** —— 当 worker 过载时，Dynamo 会基于可配置的 KV 缓存利用率与 prefill token 阈值，对新请求返回 HTTP 503。

### 可观测性（Observability）

Dynamo 内置指标（metrics）、分布式 tracing 与日志，用于监控推理部署。搭建细节参见 [Observability Guide](../observability/README.md)。

## 接下来呢？

可以通过以下资源进一步了解：

- [Recipes](https://github.com/ai-dynamo/dynamo/tree/main/recipes) —— 组合解耦服务、路由与卸载
- [KV 缓存感知路由](../components/router/router-guide.md) —— 配置智能请求路由
- [KV 缓存卸载](../components/kvbm/kvbm-guide.md) —— 搭建多层内存管理
- [Planner](../components/planner/planner-guide.md) —— 配置基于 SLA 的自动伸缩
- [Kubernetes 部署](../kubernetes/README.md) —— 配合 Grove 大规模部署
- [整体架构](../design-docs/architecture.md) —— 完整技术设计
- [支持矩阵](../reference/support-matrix.md) —— 检查硬件与引擎兼容性

**延伸阅读：** [Dynamo Digest](../digest/index.mdx)。
