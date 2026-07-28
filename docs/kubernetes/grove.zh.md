---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Grove
---

Grove 是一个专为应对现代 AI 工作负载（尤其是分离式推理系统）的编排挑战而设计的 Kubernetes API。Grove 与 NVIDIA Dynamo 无缝集成，提供完整的 AI 基础设施管理能力。

## 概述

Grove 最初源于多节点分离式推理系统的编排挑战。它提供了一致且统一的 API，使用户能在单个自定义资源中定义、配置和扩缩 prefill、decode 以及任何其他组件（例如路由）。

### Grove 如何用于分离式服务

Grove 通过将大语言模型推理拆分为可独立扩缩与管理的专门组件，从而实现分离式服务。该架构带来多项优势：

- **组件专门化**：分别针对各自任务进行优化的 prefill、decode 与路由组件
- **独立扩缩**：每个组件可根据各自的资源需求与工作负载模式独立扩缩
- **资源优化**：通过专门的工作负载放置实现更佳的硬件资源利用率
- **故障隔离**：某一组件的问题不一定会影响其他组件

## 核心组件与 API 资源

Grove 通过若干自定义 Kubernetes 资源实现分离式服务，提供基于角色的 Pod 组的声明式编排：

### PodCliqueSet
最顶层的 Grove 对象，定义一组共同管理与共置的组件。主要特性包括：
- 支持自动扩缩容
- 拓扑感知的副本分布以提升可用性
- 多个分离式组件的统一管理

### PodClique
表示一组具有特定角色的 Pod（例如 leader、worker、frontend）。每个 clique 具备：
- 独立的配置选项
- 支持自定义扩缩逻辑
- 角色特定的资源分配

### PodCliqueScalingGroup
一组共同扩缩并一同调度的 PodClique，适用于紧密耦合的角色，例如 prefill leader 与 worker 组件，需要协调扩缩行为。

## 分离式服务的关键能力

Grove 提供若干专门特性，使其特别适合分离式服务：

### 灵活的 Gang 调度
PodCliques 与 PodCliqueScalingGroups 允许用户在 PodCliqueSet 内多个层级上指定灵活的 gang 调度需求，避免资源死锁，并确保分离式系统的所有组件能一起启动。

### 多级水平自动扩缩
支持可插拔的水平自动扩缩方案，根据各自的指标和需求独立扩缩 PodCliqueSet、PodClique 与 PodCliqueScalingGroup 自定义资源。

### 网络拓扑感知调度
允许指定网络拓扑的打包与分散约束，以优化网络性能和服务可用性，这对组件之间需要高效跨节点通信的分离式系统至关重要。Dynamo 通过 DynamoGraphDeployment 资源上的 `topologyConstraint` 字段暴露此能力，因此用户无需直接接触 Grove 内部即可启用拓扑感知放置。配置详情与示例参见[拓扑感知调度指南](./topology-aware-scheduling.md)。

### 自定义启动依赖
通过声明式规范，规定 PodCliques 必须按某种顺序启动，将 Pod 启动从 Pod 创建或调度中解耦。这确保了分离式组件能正确按序初始化。

## 使用场景与示例

Grove 特别支持：

- **多节点分离式推理**，适用于 DeepSeek-R1 与 Llama-4-Maverick 等大模型
- **单节点分离式推理**，用于优化资源利用率
- **agentic 模型流水线**，用于复杂的 AI 工作流
- **标准聚合式服务**模式，适用于单节点或单 GPU 推理

## 与 NVIDIA Dynamo 的集成

Grove 在战略上与 NVIDIA Dynamo 保持一致，以便在 AI 基础设施栈中无缝集成：

### 互补角色
- **Grove**：处理分离式 AI 工作负载的 Kubernetes 编排层
- **Dynamo**：提供完整的 AI 基础设施能力，包括服务后端、路由与资源管理

### 发布协调
Grove 正在与 NVIDIA Dynamo 协调其发布节奏，以确保无缝集成；最终的发布周期会反映在项目路线图中。

### 统一 AI 平台
该集成构建了一个完整平台，其中：
- Grove 管理分离式组件的复杂编排
- Dynamo 提供服务基础设施、路由能力以及后端集成
- 二者共同支撑复杂的 AI 服务架构，并简化管理

## 架构收益

Grove 在面向 AI 工作负载的 Kubernetes 编排上代表了显著进步：

1. **简化复杂部署**：提供统一 API，可在单个资源定义中管理多个组件（prefill、decode、路由）
2. **支撑复杂架构**：支持以前难以编排的高级分离式推理模式
3. **降低运维复杂度**：抽象掉协调多个相互依赖 AI 组件的复杂性
4. **优化资源利用**：实现对组件放置与扩缩的细粒度控制

## 入门

Grove 依赖 KAI Scheduler 进行资源分配与调度。

KAI Scheduler 请参阅 [KAI Scheduler 部署指南](https://github.com/NVIDIA/KAI-Scheduler)。

安装说明请参阅 [Grove 安装指南](https://github.com/NVIDIA/grove/blob/main/docs/installation.md)。

实际多节点部署示例请参阅[多节点部署指南](./deployment/multinode-deployment.md)，其中演示了多节点分离式服务场景。

Grove 的最新进展请关注 [GitHub 上的官方项目](https://github.com/NVIDIA/grove)。

Dynamo Kubernetes Platform 也允许在平台安装时一并安装 Grove 与 KAI Scheduler。详情参见 [Dynamo Kubernetes Platform 部署安装指南](./installation-guide.md)。
