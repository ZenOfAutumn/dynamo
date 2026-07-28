---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Router
---

Dynamo KV Router 通过评估请求在不同 worker 上的计算成本，实现智能路由。它同时考虑解码成本（基于活跃 block）和预填充成本（基于新计算的 block），并利用 KV 缓存重叠来最小化冗余计算。优化 KV Router 对于在分布式推理部署中实现最大吞吐量与最低延迟至关重要。

## 快速开始

启动带 KV Router 的 Dynamo 前端：

```bash
python -m dynamo.frontend --router-mode kv --http-port 8000
```

在 Kubernetes 中，请在 Frontend 服务上设置 `DYN_ROUTER_MODE=kv`。Worker 会自动上报 KV 缓存事件——无需对 worker 端做配置变更。

| 参数 | 默认值 | 描述 |
|----------|---------|-------------|
| `--router-mode kv` | `round_robin` | 启用 KV 缓存感知路由 |
| `--router-kv-overlap-score-weight` | `1.0` | 在预填充与解码优化间取舍（值越高 TTFT 越好） |
| `--no-router-kv-events` | enabled | 回退到近似路由（不消费 worker 事件） |
| `--router-queue-threshold` | `4.0` | 反压队列阈值；通过 `nvext.agent_hints.priority` 启用优先级调度 |
| `--router-queue-policy` | `fcfs` | 队列调度策略：`fcfs`（尾部 TTFT）、`wspt`（平均 TTFT）或 `lcfs`（仅用于对比的反向排序） |
| `--no-router-track-prefill-tokens` | disabled | 在路由器负载计算中忽略提示侧的预填充 token；适用于仅解码的路由路径 |

### 独立 Router

也可以将 KV Router 作为独立服务运行（不依赖 Dynamo 前端）。详见 [Standalone Router 组件](https://github.com/ai-dynamo/dynamo/tree/main/components/src/dynamo/router/)。

部署模式与快速上手步骤请见 [Router 指南](router-guide.md)；CLI 参数与调优指南请见[配置与调优](router-configuration.md)；A/B 基准测试请见 [KV Router A/B 基准测试指南](../../benchmarks/kv-router-ab-testing.md)。

## 前置条件与限制

**要求：**
- **仅支持动态端点**：KV router 要求使用 `register_model()` 并设置 `model_input=ModelInput.Tokens`。后端 handler 接收的是带有 `token_ids` 的预先 tokenized 请求，而不是原始文本。
- 后端 worker 必须使用 `model_input=ModelInput.Tokens` 调用 `register_model()`（参见[后端指南](../../development/backend-guide.md)）
- 不能将 `--static-endpoint` 模式与 KV 路由一起使用（请改用动态发现）

**多模态支持：**
- **TRT-LLM 与 vLLM**：通过多模态哈希支持图像的多模态路由
- **SGLang**：尚不支持图像路由
- **其他模态**（音频、视频等）：尚未支持

**限制：**
- 不支持静态端点——KV router 需要通过 etcd 进行动态模型发现，以跟踪 worker 实例及其 KV 缓存状态

如果只需基础的模型注册而不需要 KV 路由，可在静态或动态端点下使用 `--router-mode round-robin`、`--router-mode random`、`--router-mode least-loaded` 或 `--router-mode device-aware-weighted`。

## 后续步骤

- **[Router 指南](router-guide.md)**：部署模式、快速开始与页面索引
- **[路由概念](router-concepts.md)**：成本模型与 worker 选择行为
- **[配置与调优](router-configuration.md)**：路由器标志、传输模式与指标
- **[分离式服务](router-disaggregated-serving.md)**：预填充与解码路由的部署
- **[Router 运维](router-operations.md)**：副本、持久化与恢复
- **[Router 示例](router-examples.md)**：Python API 用法、K8s 示例与自定义路由模式
- **[Router 测试](router-testing.md)**：从 Rust 单元测试到基于 fixture 的回放和完整进程级 E2E 的多层测试
- **[Standalone Indexer](standalone-indexer.md)**：将 KV indexer 作为独立服务运行以独立扩展
- **[Router 设计](../../design-docs/router-design.md)**：架构细节、算法与事件传输模式
