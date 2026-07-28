---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: KVBM
---

Dynamo KV Block Manager（KVBM）是一个可扩展的运行时组件，用于在异构和分布式环境中处理 KV（Key-Value）block 的内存分配、管理与远程共享。它充当统一内存层和 vLLM、TensorRT-LLM 等框架的写穿（write-through）缓存。

KVBM 提供：
- 一个**统一内存 API**，覆盖 GPU 内存、固定主机内存、远程 RDMA 可访问内存、本地 / 分布式 SSD，以及远程文件 / 对象 / 云存储系统
- 支持**block 生命周期**（allocate → register → match）以及基于事件的状态转移
- 与 **[NIXL](https://github.com/ai-dynamo/nixl/blob/main/docs/nixl.md)** 集成，NIXL 是用于内存 block 远程注册、共享与访问的动态内存交换层

> **快速上手：** 请参阅 [KVBM 指南](kvbm-guide.md) 获取安装与部署说明。

## 何时使用 KV 缓存卸载

KV 缓存卸载可避免代价高昂的 KV 缓存重计算，从而带来更快的响应时间和更好的用户体验。服务提供方将获得更高的吞吐量和更低的单 token 成本，使推理服务更具可扩展性和效率。

当 KV 缓存超过 GPU 内存且缓存复用收益超过数据传输开销时，将 KV 缓存卸载到 CPU 或存储最为有效。它在以下场景尤其有价值：

| 场景 | 收益 |
|----------|---------|
| **长会话与多轮对话** | 保留较大的提示前缀，避免重计算，改善首 token 延迟和吞吐量 |
| **高并发** | 空闲或部分对话可被移出 GPU 内存，使活跃请求在不触及内存上限的情况下继续推进 |
| **共享或重复内容** | 在用户或会话之间复用（系统提示、模板）能提升缓存命中，特别是在远程或跨实例共享时 |
| **内存或成本受限的部署** | 卸载到 RAM 或 SSD 减少了 GPU 需求，可在不增加硬件的情况下支持更长的提示或更多用户 |

## 特性支持矩阵

|  | 特性 | 支持 |
|--|---------|---------|
| **后端** | 本地 | ✅ |
|  | Kubernetes | ✅ |
| **LLM 框架** | vLLM | ✅ |
|  | TensorRT-LLM | ✅ |
|  | SGLang | ❌ |
| **服务类型** | 聚合式 | ✅ |
|  | 分离式 | ✅ |

## 架构

![KVBM 架构](../../assets/img/kvbm-components.svg)
*Dynamo KV Block Manager 的高层分层架构视图，及其与 LLM 推理生态各组件的接口方式*

KVBM 包含三个主要逻辑层：

**LLM 推理运行时层** — 顶层包括推理运行时（TensorRT-LLM、vLLM），它们通过专门的 connector 模块与 Dynamo KVBM 集成。这些 connector 充当翻译层，将运行时特定的操作和事件映射到 KVBM 的面向 block 的内存接口。这样将内存管理与推理运行时解耦，实现了后端可移植性和内存分层。

**KVBM 逻辑层** — 中间层封装了核心 KV block 管理器逻辑，作为管理 block 内存的运行时基底。KVBM 适配器对来自不同运行时的入站请求做表示和数据布局的归一化，并将其转发给核心内存管理器。该层实现表查找、内存分配、block 布局管理、生命周期状态转移以及 block 复用 / 驱逐策略。

**NIXL 层** — 底层为所有数据与存储交易提供统一支持。NIXL 实现 P2P GPU 传输、RDMA 与 NVLink 远程内存共享、动态 block 注册与元数据交换，并为存储后端提供插件接口，包括 block 内存（GPU HBM、Host DRAM、远程 DRAM、本地 SSD）、本地 / 远程文件系统、对象存储和云存储。

> **了解更多：** 详细的架构、组件和数据流请参阅 [KVBM 设计文档](../../design-docs/kvbm-design.md)。

## 后续步骤

- **[KVBM 指南](kvbm-guide.md)** — 安装、配置与部署说明
- **[KVBM 设计](../../design-docs/kvbm-design.md)** — 架构深入剖析、组件与数据流
- **[LMCache 集成](../../integrations/lmcache-integration.md)** — 在 Dynamo vLLM 后端中使用 LMCache
- **[FlexKV 集成](../../integrations/flexkv-integration.md)** — 使用 FlexKV 进行 KV 缓存管理
- **[SGLang HiCache](../../integrations/sglang-hicache.md)** — 通过 NIXL 启用 SGLang 的层次化缓存
- **[NIXL 文档](https://github.com/ai-dynamo/nixl/blob/main/docs/nixl.md)** — NIXL 通信库详细信息
