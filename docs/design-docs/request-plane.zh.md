---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 请求平面（Request Plane）
---

## 概览

Dynamo 的请求平面（request plane，即服务之间的通信层）支持多种传输机制。你可以根据部署需求从三种模式中选择：

- **TCP**（默认）：直连 TCP，性能最优
- **NATS**：基于消息代理的请求平面
- **HTTP**：基于 HTTP/2 的请求平面

本文档说明如何在 Dynamo 部署中配置和使用请求平面。

## 什么是请求平面？

请求平面是 Dynamo 各服务（如 frontend → backend、worker → worker）之间通信所走的传输层。不同的请求平面有不同的取舍：

| 请求平面 | 适用场景 | 特点 |
|--------------|----------|-----------------|
| **NATS** | 启用 KV 路由的生产部署 | 需要 NATS 基础设施，提供发布/订阅模式，灵活性最高 |
| **TCP** | 低延迟直连通信 | 点对点连接，开销最小 |
| **HTTP** | 标准化部署、调试 | HTTP/2 协议，便于用标准工具观测，兼容性广 |

## 请求平面 vs KV 事件平面

Dynamo 有 **两个相互独立的通信平面**：

- **请求平面**（**`DYN_REQUEST_PLANE`**）：组件之间（frontend → router → worker）的 **RPC 请求** 走哪条路，可选 `tcp`、`http`、`nats`。
- **KV 事件平面**（目前仅支持 **NATS**）：用于 KV-aware 路由的 **KV 缓存事件**（以及可选的 router 副本同步）的分发与持久化。

**注意：** 如果你使用 `tcp` 或 `http` 请求平面，并且 Router 端开启了 KV 事件（默认开启），NATS 会被自动初始化。SGLang 需要显式 `--kv-events-config`，TRT-LLM 需要 `--publish-events-and-metrics` 才会发布事件。vLLM 当前在 prefix caching 开启时自动配置 KV 事件（此为废弃行为——后续版本将默认关闭，请显式使用 `--kv-events-config`）。可以通过 `NATS_SERVER` 环境变量（如 `NATS_SERVER=nats://nats-hostname:port`）指定 NATS server，否则默认 `localhost:4222`。如需关闭 Router 的 KV 事件监听，在 frontend 上加 `--no-router-kv-events`。

由于两者独立，可以任意组合。

例如，TCP 请求平面可以搭配不同的 KV 事件平面：
- **JetStream KV 事件**：请求走 TCP，KV 路由仍用 NATS JetStream + 对象存储持久化。
- **NATS Core KV 事件（local indexer）**：请求走 TCP，KV 事件用 NATS Core 发布订阅，持久化落在 Worker 端。
- **不用 KV 事件**：请求走 TCP，KV 路由不依赖事件运行（无需 NATS，但也没有事件持久化）。

## 配置

### 环境变量

通过 `DYN_REQUEST_PLANE` 环境变量设置请求平面模式：

```bash
export DYN_REQUEST_PLANE=<mode>
```

`<mode>` 取值之一：
- `tcp`（默认）
- `nats`
- `http`

大小写不敏感。

### 默认行为

如果 `DYN_REQUEST_PLANE` 未设置或值非法，Dynamo 默认使用 `tcp`。

## 使用示例

### 使用 TCP（默认）

TCP 是默认请求平面，提供服务之间直连、低延迟的通信。

**配置：**

```bash
# TCP 是默认值，不显式设置也可以
# 也可以显式声明：
export DYN_REQUEST_PLANE=tcp

# 可选：配置 TCP server 的 host 和端口
export DYN_TCP_RPC_HOST=0.0.0.0  # 默认 host
# export DYN_TCP_RPC_PORT=9999   # 可选：指定固定端口

# 启动 Dynamo 服务
DYN_REQUEST_PLANE=tcp python -m dynamo.frontend --http-port=8000 &
DYN_REQUEST_PLANE=tcp python -m dynamo.vllm --model Qwen/Qwen3-0.6B
```

**注意：** 默认情况下 TCP 使用操作系统分配的空闲端口（端口 0）。这在同机多服务或需要避免端口冲突时很方便。如需固定端口（如出于防火墙规则），显式设置 `DYN_TCP_RPC_PORT`。

**什么时候用 TCP：**
- 简单的服务直连部署（如 frontend → backend）
- 基础设施需求最少（仅当 Router 监听 KV 事件时才需要 NATS；可用 `--no-router-kv-events` 关闭）
- 对延迟敏感

**TCP 配置参数：**

额外的 TCP 专属环境变量：
- `DYN_TCP_RPC_HOST`：server host 地址（默认：自动检测）
- `DYN_TCP_RPC_PORT`：server 端口。未设置则由 OS 自动分配空闲端口（推荐）。仅在需要特定端口（如防火墙规则）时显式设置。
- `DYN_TCP_MAX_MESSAGE_SIZE`：TCP 客户端最大消息大小（默认 32MB）
- `DYN_TCP_SHRINK_MESSAGE_SIZE`：处理大消息后，将 zero-copy 解码缓冲区缩回初始大小的阈值（默认 8MB，上限：DYN_TCP_MAX_MESSAGE_SIZE）
- `DYN_TCP_REQUEST_TIMEOUT`：TCP 客户端请求超时（默认 10 秒）
- `DYN_TCP_POOL_SIZE`：TCP 客户端连接池大小（默认 50）
- `DYN_TCP_CONNECT_TIMEOUT`：TCP 客户端连接超时（默认 3 秒）
- `DYN_TCP_CHANNEL_BUFFER`：TCP 客户端请求通道缓冲大小（默认 100）

### 使用 HTTP

HTTP/2 提供基于标准协议的请求平面，易调试、兼容性广。

**配置：**

```bash
# 可选：配置 HTTP server 的 host 和端口
export DYN_HTTP_RPC_HOST=0.0.0.0      # 默认 host
export DYN_HTTP_RPC_PORT=8888         # 默认端口
export DYN_HTTP_RPC_ROOT_PATH=/v1/rpc # 默认路径

# 启动 Dynamo 服务
DYN_REQUEST_PLANE=http python -m dynamo.frontend --http-port=8000 &
DYN_REQUEST_PLANE=http python -m dynamo.vllm --model Qwen/Qwen3-0.6B
```

**什么时候用 HTTP：**
- 需要 HTTP 兼容性的标准部署
- 调试场景（curl、浏览器工具等）
- 与 HTTP 基础设施集成
- 与基于 HTTP 的负载均衡和代理协作

**HTTP 配置参数：**

额外的 HTTP 专属环境变量：
- `DYN_HTTP_RPC_HOST`：server host 地址（默认：自动检测）
- `DYN_HTTP_RPC_PORT`：server 端口（默认 8888）
- `DYN_HTTP_RPC_ROOT_PATH`：RPC 端点的根路径（默认 /v1/rpc）

`DYN_HTTP2_*`：HTTP/2 客户端的各类配置：
- `DYN_HTTP2_MAX_FRAME_SIZE`：HTTP 客户端最大帧大小（默认 1MB）
- `DYN_HTTP2_MAX_CONCURRENT_STREAMS`：HTTP 客户端最大并发流（默认 1000）
- `DYN_HTTP2_POOL_MAX_IDLE_PER_HOST`：HTTP 客户端每个 host 最大空闲连接（默认 100）
- `DYN_HTTP2_POOL_IDLE_TIMEOUT_SECS`：HTTP 客户端空闲超时（默认 90 秒）
- `DYN_HTTP2_KEEP_ALIVE_INTERVAL_SECS`：HTTP 客户端 keep-alive 间隔（默认 30 秒）
- `DYN_HTTP2_KEEP_ALIVE_TIMEOUT_SECS`：HTTP 客户端 keep-alive 超时（默认 10 秒）
- `DYN_HTTP2_ADAPTIVE_WINDOW`：启用自适应流控（默认 true）

### 使用 NATS

NATS 为请求平面提供持久化的 JetStream 消息，也可用于 KV 事件（及 Router 副本同步）。

**前置条件：**
- NATS server 已启动且可访问
- 通过标准 Dynamo NATS 环境变量配置连接

```bash
# 显式设置为 NATS
export DYN_REQUEST_PLANE=nats

# 启动 Dynamo 服务
DYN_REQUEST_PLANE=nats python -m dynamo.frontend --http-port=8000 &
DYN_REQUEST_PLANE=nats python -m dynamo.vllm --model Qwen/Qwen3-0.6B
```

**什么时候用 NATS：**
- 带服务发现的生产部署
- 需要精确缓存状态追踪的 KV-aware 路由（事件传输依赖 NATS）。注意：近似模式（`--no-router-kv-events`）可在不依赖 NATS 的情况下做 KV 路由，但精度下降。
- 需要消息重放与持久化能力

限制：
- NATS 不支持大于 16MB 的消息体（大消息请用 TCP）

## 完整示例

下面是用不同请求平面启动 Dynamo 的完整例子：

详见 [`examples/backends/vllm/launch/agg_request_planes.sh`](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/vllm/launch/agg_request_planes.sh)，演示了如何用 TCP、HTTP、NATS 三种请求平面启动 Dynamo。

## 真实场景示例

Dynamo 仓库内置了一个三种请求平面的完整示例：

**位置：** `examples/backends/vllm/launch/agg_request_planes.sh`

```bash
cd examples/backends/vllm/launch

# 用 TCP
./agg_request_planes.sh --tcp

# 用 HTTP
./agg_request_planes.sh --http

# 用 NATS
./agg_request_planes.sh --nats
```

## 架构细节

### Network Manager

请求平面的实现集中在 Network Manager（`lib/runtime/src/pipeline/network/manager.rs`），它负责：

1. 启动时读取 `DYN_REQUEST_PLANE` 环境变量
2. 创建对应的 server 和 client 实现
3. 对上层代码暴露与传输方式无关的统一接口
4. 管理所有网络配置和生命周期

### 传输抽象

所有请求平面实现都遵循统一的 trait 接口：
- `RequestPlaneServer`：服务端接收请求的接口
- `RequestPlaneClient`：客户端发送请求的接口

这层抽象意味着切换请求平面时业务代码无需改动。

### 配置加载

请求平面配置在启动时从环境变量加载并全局缓存。配置层级如下：

1. **模式选择**：`DYN_REQUEST_PLANE`（默认 `tcp`）
2. **传输专属配置**：模式相关的环境变量（如 `DYN_TCP_*`、`DYN_HTTP2_*`）

## 迁移指南

### 从 NATS 迁到 TCP

1. 停掉 Dynamo 服务
2. 设置环境变量 `DYN_REQUEST_PLANE=tcp`
3. 按需配置 TCP 专属设置（如 `DYN_TCP_RPC_HOST`）。注意 `DYN_TCP_RPC_PORT` 是可选的，不设置则自动分配。
4. 重启服务

### 从 NATS 迁到 HTTP

1. 停掉 Dynamo 服务
2. 设置环境变量 `DYN_REQUEST_PLANE=http`
3. 按需配置 HTTP 专属设置（`DYN_HTTP_RPC_PORT` 等）
4. 重启服务

### 验证迁移

切换请求平面后，验证部署：

```bash
# 发个简单请求测试
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

## 故障排查

### 问题：服务之间无法通信

**症状：** 请求超时或无法到达 backend

**解决：**
- 确认所有服务使用同一个 `DYN_REQUEST_PLANE` 设置
- 检查 server 端口未被 K8s 网络策略或防火墙阻挡
- TCP/HTTP：确认 host/port 配置正确且可访问
- NATS：确认 NATS server 在跑且可达

### 问题："Invalid request plane mode" 报错

**症状：** 服务启动失败，提示配置错误

**解决：**
- 检查 `DYN_REQUEST_PLANE` 拼写（合法值：`nats`、`tcp`、`http`）
- 取值大小写不敏感，但必须是上述三个之一
- 不设置则默认 `tcp`

### 问题：端口冲突

**症状：** server 启动报 "address already in use"

**解决：**
- TCP：默认 OS 自动分配端口，几乎不会冲突。如果显式设置 `DYN_TCP_RPC_PORT` 并冲突，要么换端口，要么去掉这个设置回退到自动分配。
- HTTP 默认端口：8888（通过 `DYN_HTTP_RPC_PORT` 调整）

## 性能考量

### 延迟

- **TCP**：直连 + 二进制序列化，延迟最低
- **HTTP**：HTTP/2 协议开销，延迟中等
- **NATS**：JetStream 持久化带来一定延迟

### 资源占用

- **TCP**：基础设施需求极少（启用 KV 事件时才需 NATS，可在 Router 端用 `--no-router-kv-events` 关掉）
- **HTTP**：基础设施需求极少（启用 KV 事件时才需 NATS，可在 Router 端用 `--no-router-kv-events` 关掉）
- **NATS**：需要运行 NATS server（额外的内存/CPU 开销）

