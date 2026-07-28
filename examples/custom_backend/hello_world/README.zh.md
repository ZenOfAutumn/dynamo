<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Hello World 示例

这是最简单的 Dynamo 示例，演示了一个使用 Dynamo 分布式 runtime 的基础服务。它展示了在 Dynamo runtime 系统中创建 endpoint 与 worker 的核心概念。

## 架构

```text
Client (dynamo_worker)
      │
      ▼
┌─────────────┐
│   Backend   │  Dynamo endpoint (/generate)
└─────────────┘
```

## 组件

- **Backend（后端）**：一个 Dynamo 服务，其 endpoint 接收文本输入，并对输入中以逗号分隔的每个单词流式返回问候语
- **Client（客户端）**：一个 Dynamo worker，连接到后端服务并发送请求，然后打印响应

## 实现细节

该示例演示了：

- **Endpoint 定义**：使用 `@dynamo_endpoint` 装饰器创建流式 endpoint
- **Worker 配置**：使用 `@dynamo_worker()` 装饰器创建分布式 runtime worker
- **服务创建**：通过分布式 runtime API 创建服务与 endpoint
- **流式响应**：通过 yield 输出实现实时流
- **客户端集成**：连接到服务并处理流
- **日志**：使用 `configure_dynamo_logging` 进行基础日志配置

## 快速上手

### 前置条件

需要安装 Dynamo。本地开发不需要任何外部服务 —— 该示例默认使用基于文件的 KV 存储。

### 运行示例

首先，启动后端服务：
```bash
cd examples/custom_backend/hello_world
DYN_DISCOVERY_BACKEND=file DYN_EVENT_PLANE=zmq python hello_world.py
```

然后，在另一个终端运行客户端：
```bash
cd examples/custom_backend/hello_world
DYN_DISCOVERY_BACKEND=file DYN_EVENT_PLANE=zmq python client.py
```

> **说明**：设置 `DYN_DISCOVERY_BACKEND=file` 表示使用基于文件的服务发现，而不使用 etcd；
> 设置 `DYN_EVENT_PLANE=zmq` 表示事件面（event plane）使用 ZMQ 而不是 NATS。
> 后端与客户端必须使用相同的设置，才能彼此发现。

客户端将连接到后端服务并打印流式结果。

### 预期输出

运行客户端时，你应当看到类似如下的流式输出：
```text
Hello world!
Hello sun!
Hello moon!
Hello star!
```

## 代码结构

### 后端服务（`hello_world.py`）

- **`content_generator`**：一个 dynamo endpoint，处理文本输入并 yield 问候语
- **`worker`**：一个 dynamo worker，搭建服务、创建 endpoint 并提供服务

### 客户端（`client.py`）

- **`worker`**：一个 dynamo worker，连接到后端服务并处理流式响应

## 部署到 Kubernetes

请注意，这是一个非常简化的退化示例，并未演示标准的 Dynamo Frontend-Backend 部署。hello-world 客户端不是一个 Web 服务，而是一个一次性函数，它将预定义文本 "world,sun,moon,star" 发送给后端。该示例的目的在于展示 HelloWorldWorker。因此你只会在部署中看到 HelloWorldWorker pod。客户端会运行后退出，pod 不会持续运行。

请参考 [Quickstart Guide](/docs/kubernetes/README.md) 安装 Dynamo Kubernetes 平台。
然后部署到 Kubernetes：

```bash
export NAMESPACE=<your-namespace>
cd dynamo
kubectl apply -f examples/custom_backend/hello_world/deploy/hello_world.yaml -n ${NAMESPACE}
```

删除部署：

```bash
kubectl delete dynamographdeployment hello-world -n ${NAMESPACE}
```
