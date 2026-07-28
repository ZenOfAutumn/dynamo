---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 容错
subtitle: 通过请求迁移、取消与优雅关闭从容应对故障
---

Dynamo 提供完整的容错机制，确保生产部署中 LLM 推理的可靠性。本节介绍 Dynamo 用以从容处理故障、维持服务可用性的各项策略与特性。

## 概述

Dynamo 中的容错在多个层级运作：

| 层级 | 机制 | 用途 |
|-------|-----------|---------|
| **请求** | 迁移、取消 | 处理 in-flight 请求失败 |
| **Worker** | 健康检查、优雅关闭 | 检测并从 worker 故障中恢复 |
| **引擎进程** | [Shadow Engine 故障转移](../kubernetes/shadow-engine-failover.md) | 在 Kubernetes 中实现同节点的 active/passive 引擎进程恢复 |
| **系统** | 减载、请求拒绝 | 防止系统过载 |
| **基础设施** | etcd HA、NATS 韧性 | 处理基础设施组件故障 |

## 关键特性

### 请求迁移

当 worker 在请求处理期间失败时，Dynamo 可将进行中的请求迁移到健康的 worker。该迁移系统：

- 保留部分生成状态（已累积的 token）
- 在新的 worker 上透明地继续生成
- 维持向客户端的无缝 token 流

详见[请求迁移](request-migration.md)。

### 请求取消

Dynamo 支持取消 in-flight 请求以释放计算资源：

- 优雅停止信号用于干净终止
- Kill 信号用于立即终止
- 通过请求链的层级化取消传播

详见[请求取消](request-cancellation.md)。

### 优雅关闭

Worker 优雅地处理关闭信号（SIGTERM/SIGINT）：

- 立即停止接收新请求
- 可选择在终止前排空 in-flight 请求
- 清理资源（引擎、连接、临时文件）

详见[优雅关闭](graceful-shutdown.md)。

### 请求拒绝（减载）

当 worker 过载时，Dynamo 拒绝新请求以防止级联故障：

- 基于 KV 缓存利用率的可配置忙阈值
- 实时 worker 负载监控
- 带重试指引的 HTTP 503 响应

详见[请求拒绝](request-rejection.md)。

### 健康检查

Dynamo 提供多种健康检查机制：

- **HTTP 端点**：用于编排的 `/health` 与 `/live` 端点
- **Canary 健康检查**：通过周期性测试请求进行主动监控
- **引擎监控**：检测到引擎故障时自动关闭

详见[健康检查](../observability/health-checks.md)。

### Shadow Engine 故障转移

对 Kubernetes 部署，[Shadow Engine 故障转移](../kubernetes/shadow-engine-failover.md)有助于从未知后端引擎或软件进程故障中实现同节点恢复。它使用 GPU Memory Service 在备用或替代引擎接入时保持模型权重常驻。它不保留 in-flight 请求或 KV 缓存状态，也不覆盖 GPU 或节点丢失情况。

## 配置速查

| 特性 | 环境变量 | 默认值 |
|---------|---------------------|---------|
| Worker 健康端口 | `DYN_SYSTEM_PORT` | `9090` |
| Canary 健康检查 | `DYN_HEALTH_CHECK_ENABLED` | `false` |
| Canary 等待时间 | `DYN_CANARY_WAIT_TIME` | `10` 秒 |
| 健康检查超时 | `DYN_HEALTH_CHECK_REQUEST_TIMEOUT` | `3` 秒 |
| Decode block 阈值 | `DYN_ACTIVE_DECODE_BLOCKS_THRESHOLD` | `1.0` |
| Prefill token 阈值 | `DYN_ACTIVE_PREFILL_TOKENS_THRESHOLD` | `10000000` |


## 故障场景与恢复

### Worker Pod 重启

1. Worker 收到来自 Kubernetes 的 SIGTERM
2. 端点立即失效（不再接收新请求）
3. 根据配置完成或迁移 in-flight 请求
4. 资源被清理
5. Pod 以全新状态重启

### Worker 崩溃（意外）

1. etcd lease 过期（基于 TTL 检测）
2. 客户端通过 etcd watch 发现端点移除
3. 新请求路由到剩余健康 worker
4. 已崩溃 worker 上的 in-flight 请求被迁移（如启用）

### 网络分区

1. Worker 失去与 etcd/NATS 的连接
2. lease keep-alive 失败，最终过期
3. Worker 从服务发现中移除
4. 流量重定向到可达的 worker

### GPU 故障

1. 引擎健康检查检测到 GPU 错误（XID、OOM 等）
2. Worker 启动优雅关闭
3. 运行时被关闭，引擎被清理
4. 进程以代码 1 退出以触发 Pod 重启

## 容错测试

Dynamo 包含完整的容错验证测试框架：

- 请求取消测试
- 带 worker 失败的迁移测试
- etcd HA 故障转移测试
- 硬件故障注入（GPU XID、网络分区）

详见[容错测试](testing.md)。

## 相关文档

- [可观测性](../observability/README.md) - 指标与监控
- [Shadow Engine 故障转移](../kubernetes/shadow-engine-failover.md) - Kubernetes 部署中的同节点 active/passive 引擎故障转移
- [分布式运行时](../design-docs/distributed-runtime.md) - 服务发现架构
- [事件平面](../design-docs/event-plane.md) - 用于 KV 缓存事件与 worker 指标的发布 / 订阅
- [发现平面](../design-docs/discovery-plane.md) - 服务发现与协调
