---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 术语表
---

## B
**Block** - 用于 KV 缓存高效管理与内存分配的固定大小 token 块（通常 16 或 64 个 token），是 PagedAttention 等技术的基本单位。

## C
**Component（组件）** - Dynamo 中最基本的可部署单元。可被发现的服务实体，可承载多个端点，通常对应一个 Docker 容器（例如 VllmWorker、Router、Processor）。

**Conditional Disaggregation（条件式分离）** - Dynamo 在分离式服务中的智能决策机制：根据 prefill 长度与队列状态，决定请求在本地处理还是发送到远端 prefill 引擎。

## D
**Decode Phase（解码阶段）** - LLM 推理的第二阶段，逐个生成输出 token。

**Disaggregated Serving（分离式服务）** - Dynamo 的核心架构，将 prefill 与 decode 阶段拆分到专门的引擎中，以最大化 GPU 吞吐量并改善性能。

**Discovery Plane（发现平面）** - 服务发现层，组件（前端、路由器与 worker）在运行时使用 Kubernetes 或 etcd 后端注册服务、发现服务并监听新的服务生命周期事件。

**Distributed Runtime（分布式运行时）** - Dynamo 基于 Rust 的核心系统，跨分布式集群管理服务发现、通信与组件生命周期。

**Dynamo** - NVIDIA 面向大语言模型（LLM）和生成式 AI 模型的高性能分布式推理框架，专为多节点环境设计，支持分离式服务与缓存感知路由。

**Dynamo Kubernetes Platform** - 一个 Kubernetes 平台，为 Dynamo 推理图提供托管式部署体验。

## E
**Endpoint（端点）** - Dynamo 组件中具体的网络可访问 API，例如 `generate` 或 `load_metrics`。

**Event Plane（事件平面）** - 用于 KV 缓存更新、worker 指标与序列跟踪的发布 / 订阅层；支撑 KV 感知路由与分离式服务架构。

## F
**Frontend（前端）** - Dynamo 的 API 服务器组件，接收用户请求并提供 OpenAI 兼容的 HTTP 端点。

## G
**Graph（图）** - 一组相互连接的 Dynamo 组件，构成完整的推理流水线，具有请求路径（single-in）和响应路径（many-out 用于流式）。图可以打包为 Dynamo Artifact 用于部署。

## I
**Instance（实例）** - 拥有唯一 `instance_id` 的运行进程。多个实例可服务相同的 namespace、组件与端点，用于负载均衡。

**Inter-Token Latency (ITL)** - decode 阶段连续输出 token 之间的延迟；通常与 TTFT 一同定义性能 SLA。

## K
**KV Block Manager (KVBM)** - Dynamo 的可扩展运行时组件，跨异构与分布式环境处理 Key-Value block 的内存分配、管理与远程共享。

**KV Cache** - Key-Value 缓存，存储先前 token 的注意力状态，避免推理时重新计算。

**KV Router** - Dynamo 的智能路由系统，将请求引导至缓存重叠最多的 worker，最大化 KV 缓存复用。基于 KV 缓存命中率与 worker 指标决策路由。

**KVIndexer** - Dynamo 组件，使用前缀树结构维护所有 worker 的全局缓存 block 视图，用于计算缓存命中率。

**KVPublisher** - Dynamo 组件，将单个 worker 的 KV 缓存事件（存储 / 移除）发布给全局 KVIndexer。

## L

**LoRA (Low-Rank Adaptation)** - 微调技术，无需复制完整模型权重即可服务专门的模型变体。Dynamo 通过 worker API 支持 LoRA 适配器的运行时动态加载与服务（例如加载 / 卸载，或在 /v1/models 中发现）。

## M
**Model Deployment Card (MDC)** - 一种配置结构，包含分布式模型服务所需的所有信息。worker 加载模型时会创建一个 MDC，其中含有对 tokenizer、模板、运行时配置等组件的引用。Worker 发布其 MDC 以使该模型对前端可发现。前端使用 MDC 配置请求预处理（tokenization、prompt 格式化）。

## N
**Namespace（命名空间）** - Dynamo 的相关组件逻辑分组机制。类似文件系统中的目录，可避免不同部署间的冲突。

**NIXL (NVIDIA Inference tranXfer Library)** - 面向推理工作负载优化的高性能数据传输库，支持直接 GPU 到 GPU 传输与多种内存层级。

## P
**PagedAttention** - 来自 vLLM 的内存管理技术，通过将请求分块为 block 来高效管理 KV 缓存。

**Planner** - Dynamo 组件，根据实时需求信号与系统指标进行动态资源扩缩。

**Prefill Phase（预填充阶段）** - LLM 推理的第一阶段，处理输入 prompt 并生成 KV 缓存。

**Prefix Caching（前缀缓存）** - 优化技术，复用常见 prompt 前缀已计算的 KV 缓存。

**Processor** - Dynamo 组件，处理请求预处理、tokenization 与路由决策。

**Profiler** - Dynamo 组件，分析模型性能以确定最优引擎配置，包括 disagg/agg、并行映射（TP、TEP、DEP）以及其他引擎旋钮（batch size、max num tokens），为 Planner 提供 SLA 驱动的自动扩缩输入。

## R
**RadixAttention** - SGLang 的技术，使用前缀树结构以高效进行 KV 缓存匹配、插入与驱逐。

**RDMA (Remote Direct Memory Access)** - 允许分布式系统之间直接内存访问的技术，用于高效的 KV 缓存传输。

**Request Plane（请求平面）** - 在组件之间传输 RPC 的传输层（前端到 worker 或路由器到路由器），可使用 TCP、HTTP 或 NATS 中的一种协议。

## S
**SGLang** - 快速 LLM 推理框架，原生支持嵌入与 RadixAttention。

**Speculative Decoding（推测解码）** - 优化方法，由 draft 模型提议 token，主模型并行验证；可降低延迟（例如 vLLM with Eagle）。

## T
**Tensor Parallelism (TP)** - 模型并行技术，将模型权重分布到多个 GPU 上。

**TensorRT-LLM** - NVIDIA 优化的 LLM 推理引擎，支持多节点 MPI 分布式。

**Time-To-First-Token (TTFT)** - 从接收请求到生成首个输出 token 之间的延迟。

## V
**vLLM** - 高吞吐 LLM 服务引擎，具备分布式张量 / 流水线并行与 PagedAttention。

## W
**Wide Expert Parallelism (WideEP)** - 一种 MoE 部署策略，将专家分布到多个 GPU 上（例如 64-way EP），使每个 GPU 仅承载少量专家。

## X
**xPyD (x Prefill y Decode)** - Dynamo 用于描述分离式服务配置的记号，其中 x 个 prefill worker 服务 y 个 decode worker。Dynamo 支持运行时可重配置的 xPyD。
