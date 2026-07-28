---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Runtime 指南
---

<h4>面向数据中心规模的分布式推理服务框架</h4>

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

Dynamo runtime 系统的 Rust 实现，为机器学习负载提供分布式计算能力。

## 前置条件

### 通过 [rustup](https://rustup.rs/) 安装 Rust 与 Cargo：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### 构建

```
cargo build
cargo test
```

### 启动依赖服务

#### Docker Compose

部署所需依赖服务的最简单方式是使用 [docker-compose](https://docs.docker.com/compose/install/linux/)，其配置定义在 [deploy/docker-compose.yml](https://github.com/ai-dynamo/dynamo/tree/main/deploy/docker-compose.yml)。

```
# 在仓库根目录执行：
docker compose -f deploy/docker-compose.yml up -d
```

这会部署一个 [NATS.io](https://nats.io/) server 和一个 [etcd](https://etcd.io/) server，用于运行时各组件之间的通信与服务发现。


#### 本地启动（替代方案）

如果不想使用上面的 `docker-compose`，你也可以手动启动这些前置服务：

- 启用了 [Jetstream](https://docs.nats.io/nats-concepts/jetstream) 的 [NATS.io](https://docs.nats.io/running-a-nats-service/introduction/installation) server
    - 示例：`nats-server -js --trace`
- [etcd](https://etcd.io) server
    - 按照 [etcd installation](https://etcd.io/docs/v3.5/install/) 的说明在本地启动 `etcd-server`


### 运行示例

在开发或运行示例时，任何与你共享核心服务（`etcd` 与 `nats.io`）的进程或用户，都将处于你的分布式 runtime 中。

当前示例使用的是硬编码的 `namespace`。`namespace` 冲突问题我们后续会处理。

大多数示例都需要 `etcd` 用于服务发现。`nats.io` 在带事件追踪的 KV 感知路由中是必需的；对于近似模式（`--no-router-kv-events`），NATS 是可选的。

#### Rust `hello_world`

打开两个终端，在第一个终端中：

```
cd examples/hello_world
cargo run --bin server
```

在第二个终端中执行：

```
cd examples/hello_world
cargo run --bin client
```

应该会得到类似下面的输出：
```
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 6.25s
     Running `target/debug/client`
Annotated { data: Some("h"), id: None, event: None, comment: None }
Annotated { data: Some("e"), id: None, event: None, comment: None }
Annotated { data: Some("l"), id: None, event: None, comment: None }
Annotated { data: Some("l"), id: None, event: None, comment: None }
Annotated { data: Some("o"), id: None, event: None, comment: None }
Annotated { data: Some(" "), id: None, event: None, comment: None }
Annotated { data: Some("w"), id: None, event: None, comment: None }
Annotated { data: Some("o"), id: None, event: None, comment: None }
Annotated { data: Some("r"), id: None, event: None, comment: None }
Annotated { data: Some("l"), id: None, event: None, comment: None }
Annotated { data: Some("d"), id: None, event: None, comment: None }
```

#### Python

详情参见 [README.md](https://github.com/ai-dynamo/dynamo/tree/main/lib/runtime/lib/bindings/python/README.md)。

Python 与 Rust 的 `hello_world` 客户端和服务端是可互换的——你可以启动 Python 的 `server.py`，再用 Rust 的 `client` 与之通信。

