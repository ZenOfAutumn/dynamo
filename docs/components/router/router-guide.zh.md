---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Router Guide
subtitle: Deployment modes, quick start, and page map for Dynamo routing docs
---

## 概述

Dynamo KV Router 通过评估请求在不同 worker 上的计算开销来智能地完成路由。它同时考虑解码（decode）开销（来自活跃 block）与预填充（prefill）开销（来自新计算的 block），并利用 KV 缓存（KV cache）的重叠以最小化重复计算。在分布式推理（inference）部署中，优化 KV Router 是实现最大吞吐量与最小延迟的关键。
本指南帮助你开始使用 Dynamo router，并指引你阅读涵盖路由概念、配置、解耦（disaggregated）服务以及运维细节的相关文档页面。

## 快速开始

Router 可以通过 [Python / CLI](#python--cli-deployment)、[Kubernetes](#kubernetes-deployment) 或作为 [独立组件](#standalone-router) 进行部署。

### Python / CLI 部署

启动启用了 KV Router 的 Dynamo 前端（frontend）：

```bash
python -m dynamo.frontend --router-mode kv --http-port 8000
```

该命令会：
- 启动启用了 KV 路由的 Dynamo 前端服务
- 在 8000 端口（可配置）暴露服务
- 自动处理所有注册到 Dynamo 端点的后端（backend）worker

后端 worker 通过 `register_model` API 进行注册，之后 KV Router 会自动追踪 worker 状态，并基于 KV 缓存的重叠做出路由决策。

#### CLI 参数

| 参数 | 默认值 | 说明 |
|----------|---------|-------------|
| `--router-mode kv` | `round-robin` | 启用基于 KV 缓存的路由 |
| `--router-temperature <float>` | `0.0` | 控制路由的随机性（0.0 = 确定，越高越随机） |
| `--kv-cache-block-size <size>` | 取决于后端 | KV 缓存 block 大小（应与后端配置一致） |
| `--router-kv-events` / `--no-router-kv-events` | `--router-kv-events` | 启用/禁用实时 KV 事件追踪 |
| `--router-kv-overlap-score-weight <float>` | `1.0` | 平衡 prefill 与 decode 优化（更高 = 更佳 TTFT） |
| `--router-track-prefill-tokens` / `--no-router-track-prefill-tokens` | `--router-track-prefill-tokens` | 在活跃 worker 负载统计中纳入 prompt 端负载 |
| `--router-prefill-load-model <none\|aic>` | `none` | Prompt 端负载模型。`aic` 仅使用 AIC 预测时长来衰减最旧的活跃 prefill |
| `--router-queue-threshold <float>` | `4.0` | 队列阈值占比；通过 `priority` 启用优先级调度 |
| `--router-queue-policy <str>` | `fcfs` | 队列调度策略：`fcfs`（尾部 TTFT）、`wspt`（平均 TTFT）或 `lcfs`（仅用于比较的逆序） |
| `--serve-indexer` | `false` | 在该前端/router 上的 worker 组件中提供 Dynamo 原生远程 indexer |
| `--use-remote-indexer` | `false` | 查询 worker 组件提供的远程 indexer，而非维护本地的 overlap indexer |

查看所有可用选项：`python -m dynamo.frontend --help`

详细配置选项与调优参数，请参阅 [配置与调优](router-configuration.md)。

#### AIC Prefill 负载模型

KV router 可使用 AIC 估计所选 worker 上 prompt 端 prefill 工作的预期时长。启用后，router 会：

- 为所选 worker 计算 `prefix = overlap_blocks * block_size`
- 计算 `effective_isl = input_tokens - prefix`
- 为已准入的请求存储一个 prompt 负载提示
- 仅对每个 worker 上 **最旧** 的活跃 prefill 请求随时间衰减

这只影响 router 端的 prompt 负载统计，不会改变后端的执行或 decode 端统计。

在前端启用方式：

```bash
python -m dynamo.frontend \
    --router-mode kv \
    --router-prefill-load-model aic \
    --aic-backend vllm \
    --aic-system h200_sxm \
    --aic-model-path nvidia/Llama-3.1-8B-Instruct-FP8
```

独立 router 使用相同的 AIC 参数：

```bash
python -m dynamo.router \
    --endpoint dynamo.prefill.generate \
    --router-prefill-load-model aic \
    --aic-backend vllm \
    --aic-system h200_sxm \
    --aic-model-path nvidia/Llama-3.1-8B-Instruct-FP8
```

启用 `--router-prefill-load-model=aic` 时所需参数：

- 前端的 `--router-mode kv`
- `--router-track-prefill-tokens`
- `--aic-backend`
- `--aic-system`
- `--aic-model-path`

可选 AIC 参数：

- `--aic-backend-version`：固定的 AIC 数据库版本；省略时 Dynamo 使用对应后端的默认值
- `--aic-tp-size`：建模后端的张量并行大小；默认 `1`

### Kubernetes 部署

要在 Kubernetes 中启用 KV Router，将 `DYN_ROUTER_MODE` 环境变量加入到前端服务：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-deployment
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      envs:
        - name: DYN_ROUTER_MODE
          value: kv  # Enable KV Smart Router
```

**关键点：**
- 仅在 **Frontend** 服务上设置 `DYN_ROUTER_MODE=kv`
- worker 自动向 router 上报 KV 缓存事件
- 不需要修改 worker 端配置

#### 环境变量

所有 CLI 参数都可通过 `DYN_` 前缀的环境变量进行配置：

| CLI 参数 | 环境变量 | 默认值 |
|--------------|---------------------|---------|
| `--router-mode kv` | `DYN_ROUTER_MODE=kv` | `round-robin` |
| `--router-temperature` | `DYN_ROUTER_TEMPERATURE` | `0.0` |
| `--kv-cache-block-size` | `DYN_KV_CACHE_BLOCK_SIZE` | 取决于后端 |
| `--no-router-kv-events` | `DYN_ROUTER_USE_KV_EVENTS=false` | `true` |
| `--router-kv-overlap-score-weight` | `DYN_ROUTER_KV_OVERLAP_SCORE_WEIGHT` | `1.0` |
| `--router-queue-policy` | `DYN_ROUTER_QUEUE_POLICY` | `fcfs` |
| `DYN_ENCODER_CUDA_TO_CPU_RATIO` | `8` | `device-aware-weighted` 路由中，单个非 CPU worker 相对单个 CPU worker 的吞吐比 |

完整 K8s 示例与高级配置，请参阅 [K8s 示例](router-examples.md#k8s-examples) 与 [配置与调优](router-configuration.md)。
关于 A/B 测试与高级 K8s 配置，请参阅 [KV Router A/B Benchmarking 指南](../../benchmarks/kv-router-ab-testing.md)。

### 独立 Router

你也可以将 KV router 作为独立服务运行（不带 Dynamo 前端），用于解耦（disaggregated）服务（例如路由到 prefill worker）、多层架构，或任何需要智能 KV 缓存感知路由决策的场景。详见 [独立 Router 组件](https://github.com/ai-dynamo/dynamo/tree/main/components/src/dynamo/router/)。

#### 嵌入前端 vs. 独立 Router

| 部署 | 进程 | 指标端口 | 适用场景 |
|------------|---------|--------------|----------|
| **嵌入前端** | `python -m dynamo.frontend --router-mode kv` | 前端 HTTP 端口（默认 8000） | 标准部署；router 在前端进程内运行 |
| **独立** | `python -m dynamo.router` | `DYN_SYSTEM_PORT`（如设置） | 多层架构、SGLang disagg prefill 路由、自定义流水线 |

独立 router 不包含 HTTP 前端（没有 `/v1/chat/completions` 端点）。它仅通过 system status 服务器暴露 `RouterRequestMetrics`。详见 [独立 Router README](https://github.com/ai-dynamo/dynamo/blob/main/components/src/dynamo/router/README.md)。

## 部署模式

Dynamo router 可以以多种配置部署。下表列出所有组合及其适用场景：

| 模式 | 命令 | 路由逻辑 | KV 事件 | 拓扑 | 适用场景 |
|------|---------|---------------|-----------|----------|----------|
| **Frontend + Round-Robin** | `python -m dynamo.frontend --router-mode round-robin` | 在 worker 间轮询 | 无 | 聚合 | 最简基线，不感知 KV |
| **Frontend + Random** | `python -m dynamo.frontend --router-mode random` | 随机选择 worker | 无 | 聚合 | 无状态负载均衡 |
| **Frontend + KV (Aggregated)** | `python -m dynamo.frontend --router-mode kv` | KV 缓存重叠 + 负载 | NATS Core / JetStream / ZMQ / Approx | 聚合 | 单池生产服务 + 缓存复用 |
| **Frontend + KV (Disaggregated)** | `python -m dynamo.frontend --router-mode kv`（含 prefill + decode worker） | KV 缓存重叠 + 负载 | NATS Core / JetStream / ZMQ / Approx | 解耦（prefill + decode 池） | 大规模服务下 prefill/decode 分离 |
| **Frontend + Least-Loaded** | `python -m dynamo.frontend --router-mode least-loaded` | 最少活跃连接 | 无 | 聚合或解耦回退 | 不感知 KV 的简单负载均衡 |
| **Frontend + Device-Aware Weighted** | `python -m dynamo.frontend --router-mode device-aware-weighted` | 设备感知预算 + 在所选设备组内 least-loaded | 无 | 聚合或解耦回退 | 异构资源池均衡（CPU/非 CPU）；只剩一种设备类型时退化为 least-loaded |
| **Frontend + Direct** | `python -m dynamo.frontend --router-mode direct` | 来自请求 hint 的 worker ID | 无 | 聚合 | 由外部编排器（如 EPP/GAIE）选择 worker |
| **Standalone Router** | `python -m dynamo.router` | KV 缓存重叠 + 负载 | NATS Core / JetStream / ZMQ | 任意 | 不带 HTTP 前端的路由（多层、自定义流水线） |

### 路由模式（`--router-mode`）

| 模式 | 取值 | worker 选择方式 |
|------|-------|-------------------------|
| **Round-Robin** | `round-robin`（默认） | 按顺序轮询可用 worker |
| **Random** | `random` | 每个请求随机选择 worker |
| **KV** | `kv` | 评估每个 worker 的 KV 缓存重叠和 decode 负载，选择开销最低者 |
| **Least-Loaded** | `least-loaded` | 路由到活跃连接最少的 worker；在解耦的 prefill 路径下跳过 bootstrap 优化，回退到同步 prefill |
| **Device-Aware Weighted** | `device-aware-weighted` | 将 worker 划分为 CPU 与非 CPU 两组，结合 `DYN_ENCODER_CUDA_TO_CPU_RATIO` 进行能力归一化的比例预算决策，选择该组中负载最低的 worker |
| **Direct** | `direct` | 从请求的路由 hint 中读取目标 `worker_id`；无选择逻辑 |

### Device-Aware Weighted 路由

`device-aware-weighted` 适用于异构资源池：例如 CPU embedding 编码器与 GPU embedding 编码器共享同一端点。

worker 被划分为 CPU 和非 CPU 两组。Router 比较两组的能力归一化负载：

```text
normalized_load = total_inflight(group) / (instance_count(group) x throughput_weight)
```

CPU worker 的 throughput weight 为 `1`，非 CPU worker 为 `DYN_ENCODER_CUDA_TO_CPU_RATIO`。下一个请求会被路由到归一化负载较低的组，再选择该组中负载最低的 worker。

通过 `DYN_ENCODER_CUDA_TO_CPU_RATIO` 来近似单个非 CPU worker 相对单个 CPU worker 的吞吐比，默认 `8`。

只剩一种设备类型时，该策略退化为标准的 least-loaded 路由。

### KV 事件传输模式（在 `--router-mode kv` 下）

使用 KV 路由时，router 需要知道每个 worker 缓存了什么。有四种方式获得这些信息：

| 事件模式 | 启用方式 | 说明 |
|------------|---------------|-------------|
| **NATS Core（本地 indexer）** | 默认（无额外参数） | worker 维护本地 indexer；router 启动时查询 worker，并通过 NATS Core 接收事件 |
| **JetStream（持久化）** | `--router-durable-kv-events` | 事件持久化到 NATS JetStream；支持快照与持久消费者。*已废弃。* |
| **ZMQ** | `--event-plane zmq` | worker 通过 ZMQ PUB socket 发布；独立的 `dynamo.indexer` 服务汇总事件 |
| **Approximate（无事件）** | `--no-router-kv-events` | 不消费事件；router 仅基于自身路由决策、并通过 TTL 过期来预测缓存状态 |

### 聚合 vs. 解耦拓扑

| 拓扑 | worker | 工作方式 |
|----------|---------|--------------|
| **聚合** | 单一池（prefill + decode 在同一进程） | 所有 worker 处理完整的请求生命周期 |
| **解耦** | 分离的 prefill 与 decode 池 | 前端先路由到 prefill worker，再路由到 decode worker；要求 worker 注册为 `ModelType.Prefill` |

当 prefill worker 与 decode worker 一同注册时，自动激活解耦模式。详见 [解耦服务](router-disaggregated-serving.md)。

## 更多 Router 文档

- **[路由概念](router-concepts.md)**：开销模型、worker 选择以及路由原语
- **[配置与调优](router-configuration.md)**：router 参数、传输模式、负载追踪与指标（metrics）
- **[解耦服务](router-disaggregated-serving.md)**：prefill 与 decode 路由配置
- **[Router 运维](router-operations.md)**：副本、远程 indexer、持久化与恢复
- **[Router 示例](router-examples.md)**：Python API 用法、K8s 示例与自定义路由模式
- **[Router 测试](router-testing.md)**：针对 router 重大变更的推荐测试层级
- **[独立 Indexer](standalone-indexer.md)**：将 KV indexer 作为独立服务运行
- **[KV 事件回放——Dynamo vs vLLM](kv-event-replay-comparison.md)**：间隙检测与回放行为
