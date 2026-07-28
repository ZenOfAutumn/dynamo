---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Request Migration
---

本文介绍 Dynamo 是如何实现请求迁移（request migration），以便在 LLM 文本生成
过程中优雅地处理 worker 故障。请求迁移允许进行中的请求在原 worker 不再可用时
继续在另一个 worker 上完成执行，从而提供容错能力并改善用户体验。

## 概述

请求迁移是通过位于 LLM 处理流水线中、Backend operator 与服务后端（service
backend）之间的 Migration operator 来实现的。当某个 worker 在请求处理过程中
故障时，迁移系统会保留部分已完成的生成状态，并在新的 worker 上重建该请求，
从上一个 worker 中断的位置继续生成。

## 架构组件

### Migrator

迁移系统集成在 LLM 处理流水线中前端预处理（preprocessing）与实际服务后端之间
的位置。这一定位使其能够拦截所有的通信流，并对故障场景做透明处理。

主要职责：
- 拦截流经流水线的所有请求与响应
- 通过错误模式匹配检测 worker 故障场景
- 管理重试逻辑，并支持可配置的迁移上限
- 跟踪部分响应的状态，确保无缝续接

### 迁移上限配置

迁移上限在**前端（frontend）**层面配置，并对该前端服务的所有模型全局生效。该
参数指定一个请求最多可被迁移到其他 worker 的次数：

- 默认行为：不允许迁移（migration_limit=0）
- 通过前端的 `--migration-limit` 标志设置
- 对该前端服务的所有模型生效

### 最大序列长度配置

最大序列长度配置控制迁移系统会为某个请求缓存 token 状态多久。一旦总序列长度
（prompt + 已生成 tokens）超过该上限，就会对该请求关闭迁移并停止 token 跟踪：

- 默认行为：无上限（`--migration-max-seq-len` 未设置）
- 在前端通过 `--migration-max-seq-len` 标志或 `DYN_MIGRATION_MAX_SEQ_LEN` 环境
  变量设置
- 防止因缓存过长序列而导致内存无界增长
- 边界：恰好等于上限时仍可迁移；只有严格超过该上限时才会关闭迁移
- 该检查会在请求初始化（prompt 长度）和生成过程中（prompt + 输出 tokens）同时
  进行

## Token 状态跟踪与请求迁移

迁移系统的核心是通过 token 状态管理来保存并续接部分生成的能力。这能保证当一个
worker 在生成中途故障时，新的 worker 可以从故障发生的精确位置无缝续接。

### Token 累积过程

当请求正在被处理、响应不断从 worker 流回时，迁移系统会跟踪每一个被成功生成
的 token：

1. **初始请求状态**：系统从原始的预处理请求开始，其中包含初始 prompt tokens。

2. **响应跟踪**：每当 worker 返回一个响应时，迁移系统会取出新生成的 tokens
   并追加到该请求的 token 序列中。这样就累积下了所有已经生成的 tokens。

3. **Token 配额管理**：系统也会更新剩余 token 配额，使其反映已经生成的 token
   数量，从而保证整体生成长度不会超出最初请求所限定的范围。

### 触发迁移的场景

迁移系统处理两种不同的故障场景：

#### 1. 新请求迁移（建立连接时失败）

**场景**：在创建初始连接时 worker 不可达。

**错误模式**：通信系统报告所选 worker 实例不可用。

**迁移过程**：
- 在初始建流时检测到连接失败
- 减少剩余迁移重试次数
- 用原始请求重新尝试建立新的流
- 由于尚未开始生成，因此不需要保留任何部分状态

#### 2. 进行中请求迁移（流中途断开）

**场景**：在已经接收到部分响应、正在生成的过程中连接丢失。

**错误模式**：在生成尚未完成时检测到流终止。

**迁移过程**：

1. **故障检测**：系统通过错误监控检测到流断开。

2. **状态保留**：此时该请求的 token 序列已包含原始 prompt tokens，以及来自
   故障 worker 已成功生成的所有 tokens。

3. **新流创建**：用累积下来的请求状态创建一个新的流，确保新 worker 拥有完整
   的上下文。

4. **续接**：新的 worker 收到带有完整 token 上下文的请求，并从前一个 worker
   中断的精确位置继续生成。

### 无缝的 token 流与请求状态演进

从客户端的视角看，token 流是连续且不被打断的。客户端先从第一个 worker 接收
tokens，故障发生后又会无缝地继续从备用 worker 接收 tokens，整个过程不会暴露
出底层迁移的痕迹。

请求状态在处理过程中是动态演进的。最初请求只包含原始的 prompt tokens；随着
生成推进，每一个被成功生成的 token 都会被追加到该请求的 token 序列中，逐步
形成完整对话上下文的累积记录。

当迁移发生时，这份累积下来的状态会被转移到新的 worker 上，新 worker 用它来
重建完整上下文。然后新 worker 会按照"自己一直在处理这个请求"的方式继续生成，
但起始位置就是该序列当前的进度。

迁移之所以是透明的，是因为：
1. 切换过程中没有 token 丢失或重复
2. 通过累积的 token 序列，新 worker 拥有完整的上下文
3. 生成从故障的精确位置继续
4. 响应流的格式与时序保持一致

这种 token 累积机制保证了迁移真正做到无缝衔接，在 worker 切换间保留了所有
计算成果，并保持生成质量。

## 收益

1. **容错能力**：在个别 worker 故障期间系统仍可继续工作
2. **资源效率**：保留部分生成成果而不是从头重启
3. **无缝用户体验**：用户在 worker 故障期间不会感知中断
4. **行为可配置**：迁移上限允许根据部署需求做调优
5. **零 token 丢失**：在迁移过程中完整保留生成状态

## 设计考量

迁移系统在架构层面有以下几点重要考量：

**多模型支持**：由于一个前端可同时服务多个模型，迁移上限被配置在前端层面并
对所有模型统一生效，从而简化了运维管理。

**状态管理**：系统不仅仔细跟踪 token 序列，还跟踪诸如剩余 token 配额、停止
条件、采样参数等元数据，以保证状态被完整保留。

**错误处理**：迁移系统会区分不同类型的故障，并对每种场景采用相应的恢复策略。

## 监控与指标

迁移系统通过 Prometheus 暴露指标（metrics），用于监控迁移活动。这些指标在
前端的 `/metrics` 端点（默认端口 8000）上提供：

- `dynamo_frontend_model_migration_total`：跟踪请求迁移总次数的 Counter
  - 标签：
    - `model`：所服务的模型名称
    - `migration_type`：取值为 `new_request`（初始连接失败）或
      `ongoing_request`（流中途断开）
- `dynamo_frontend_model_migration_max_seq_len_exceeded_total`：跟踪因序列
  长度超过 `--migration-max-seq-len` 而禁用迁移的次数的 Counter
  - 标签：
    - `model`：所服务的模型名称

**示例指标输出：**
```text
dynamo_frontend_model_migration_total{migration_type="ongoing_request",model="Qwen/Qwen3-0.6B"} 3
dynamo_frontend_model_migration_total{migration_type="new_request",model="Qwen/Qwen3-0.6B"} 1
dynamo_frontend_model_migration_max_seq_len_exceeded_total{model="Qwen/Qwen3-0.6B"} 2
```

这些指标可用于：
- 监控 worker 的可靠性与故障模式
- 在迁移率异常偏高时告警，提示基础设施问题
- 跟踪容错机制的有效性
- 监控 `--migration-max-seq-len` 被触达的频率，作为是否需要调整该上限的依据

关于 Dynamo 指标的更多信息，请参见
[Metrics 文档](../observability/metrics.md)。

## 已知限制

### 多个候选（`n > 1`）

对于要求生成多个候选（`n > 1`）的 OpenAI 兼容请求，请求迁移**不被支持**。
即使 `--migration-limit` 大于 0，Dynamo 也会对这类请求关闭迁移。

**原因：** 多候选生成会为每个候选维护独立的输出状态。如果想迁移一个部分完成的
请求，就需要为每个候选独立转移其已生成 token 状态、剩余 token 配额、终止状态
以及解码器状态。当前的迁移路径只能保留单一的续接状态，因此对一个交错的 `n > 1`
请求重试，可能导致候选相关输出被重复或丢弃。

该限制不影响常规的单候选请求（即省略 `n` 或将 `n` 设为 1 的情形）。

### 引导式解码（结构化输出）

对于使用引导式解码（guided decoding，结构化输出 / JSON schema）的请求，请求
迁移**不被支持**。当 worker 在引导式解码请求生成中途故障时，错误会被传播给
客户端，而不会尝试迁移。

**原因：** 推理后端为每个新请求都会重新初始化引导式解码的有限状态机（FSM），
并且只针对新生成的 tokens 推进 FSM，不会针对上下文 / prompt tokens 推进。当
一个部分完成的请求被迁移到新 worker 时，新 worker 会把已经生成的 tokens 当作
上下文进行 replay，但 FSM 仍然会从 schema 根节点开始 —— token 状态与 FSM 状态
之间的不一致会产生损坏的输出，通常表现为重复或嵌套的 JSON。

该限制对所有后端都同样适用（vLLM、SGLang、TRT-LLM）。

**未来方向：** 要支持引导式解码请求的迁移，需要在 worker 之间序列化并恢复 FSM
状态，或者在新 worker 上把先前的输出 tokens 通过 FSM replay 一遍。该工作作为
未来增强项进行跟踪。

## 运维影响

请求迁移从根本上改变了系统对故障的处理方式 —— 从 "fail-fast" 转向 "graceful
degradation"。这种架构上的转变在保持对外 API 契约不变的同时，带来了更高的可用性
与更高的资源利用率。
