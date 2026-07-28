---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: KV 缓存卸载
subtitle: Dynamo 中 vLLM 的 CPU 与磁盘卸载集成
---

# KV 缓存卸载

Dynamo 为 vLLM 提供多种 KV 缓存卸载后端，使你能够利用 CPU RAM 与磁盘存储将有效的 KV 缓存容量扩展到 GPU 内存之外。各后端通过 vLLM 的 connector 接口进行集成，并同时适用于聚合式与分离式服务。


| 后端                 | 来源                                           |
| ----------------------- | ------------------------------------------------ |
| **[KVBM](#kvbm)**       | [Dynamo](../../components/kvbm/README.md)        |
| **[LMCache](#lmcache)** | [GitHub](https://github.com/LMCache/LMCache)     |
| **[FlexKV](#flexkv)**   | [GitHub](https://github.com/taco-project/FlexKV) |


## KVBM

[KVBM](../../components/kvbm/README.md)（KV Block Manager）是 Dynamo 内置的 KV 缓存卸载系统。它提供三层架构（LLM 运行时、逻辑 block 管理、NIXL 传输），支持 CPU 与磁盘缓存层级，并与 Dynamo 的 KV 感知路由和分离式服务原生集成。


| 部署                 | 启动脚本                                                                           |
| -------------------------- | --------------------------------------------------------------------------------------- |
| 聚合式                 | [`agg_kvbm.sh`](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/launch/agg_kvbm.sh)                     |
| 聚合式 + KV 路由    | [`agg_kvbm_router.sh`](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/launch/agg_kvbm_router.sh)       |
| 分离式（1P1D）       | [`disagg_kvbm.sh`](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/launch/disagg_kvbm.sh)               |
| 分离式（2P2D）       | [`disagg_kvbm_2p2d.sh`](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/launch/disagg_kvbm_2p2d.sh)     |
| 分离式 + KV 路由 | [`disagg_kvbm_router.sh`](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/launch/disagg_kvbm_router.sh) |


配置详情请参阅 [KVBM 指南](../../components/kvbm/kvbm-guide.md)。

## LMCache

[LMCache](https://github.com/LMCache/LMCache) 是一个开源 KV 缓存引擎，提供"一次预填充、随处复用"的缓存能力，支持多级存储后端（CPU RAM、本地存储、Redis、GDS、InfiniStore/Mooncake）。


| 部署                        | 启动脚本                                                                                 |
| --------------------------------- | --------------------------------------------------------------------------------------------- |
| 聚合式                        | [`agg_lmcache.sh`](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/launch/agg_lmcache.sh)                     |
| 聚合式（多进程指标） | [`agg_lmcache_multiproc.sh`](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/launch/agg_lmcache_multiproc.sh) |
| 分离式                     | [`disagg_lmcache.sh`](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/launch/disagg_lmcache.sh)               |


配置详情请参阅 [LMCache 集成指南](../../integrations/lmcache-integration.md)。

## FlexKV

[FlexKV](https://github.com/taco-project/FlexKV) 是腾讯云 TACO 团队开发的可扩展、分布式 KV 缓存运行时。支持多级缓存（GPU、CPU、SSD）、跨节点的分布式 KV 缓存复用，以及通过 io_uring 与 GPUDirect Storage 实现的高性能 I/O。


| 部署              | 启动脚本                                                                         |
| ----------------------- | ------------------------------------------------------------------------------------- |
| 聚合式              | [`agg_flexkv.sh`](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/launch/agg_flexkv.sh)               |
| 聚合式 + KV 路由 | [`agg_flexkv_router.sh`](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/launch/agg_flexkv_router.sh) |
| 分离式           | [`disagg_flexkv.sh`](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/vllm/launch/disagg_flexkv.sh)         |


配置详情请参阅 [FlexKV 集成指南](../../integrations/flexkv-integration.md)。

## 另请参阅

- **[KVBM 设计](../../design-docs/kvbm-design.md)**：Dynamo 内置 KV 缓存卸载的架构与设计
- **[路由概念](../../components/router/router-concepts.md)**：基于 KV 缓存状态进行请求路由
- **[分离式服务](../../design-docs/disagg-serving.md)**：预填充 / 解码分离架构
