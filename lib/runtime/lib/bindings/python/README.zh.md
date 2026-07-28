<!--
SPDX-FileCopyrightText: Copyright (c) 2024-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# Dynamo Runtime

<h4>面向数据中心规模的分布式推理服务框架</h4>

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

Dynamo 运行时系统的 Rust 实现，为机器学习负载提供分布式计算能力。

## 🛠️ 前置条件

### 通过 [rustup](https://rustup.rs/) 安装 Rust 与 Cargo：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### 构建

```
cargo build
cargo test
```

### 启动依赖项

#### Docker Compose

部署前置依赖服务最简单的方式是使用
[docker-compose](https://docs.docker.com/compose/install/linux/)，相关定义见项目
根目录下的 [docker-compose.yml](../../../../../deploy/docker-compose.yml)。

```
docker-compose up -d
```

这会启动一个 [NATS.io](https://nats.io/) 服务器和一个 [etcd](https://etcd.io/)
服务器，用于在运行时进行组件间通信和发现。

#### 本地（备选）

如果你不希望使用上面的 `docker-compose`，也可以在本地手动启动每一项前置服务：

- [NATS.io](https://docs.nats.io/running-a-nats-service/introduction/installation)
  服务器，并启用 [Jetstream](https://docs.nats.io/nats-concepts/jetstream)
    - 示例：`nats-server -js --trace`
- [etcd](https://etcd.io) 服务器
    - 按照 [etcd installation](https://etcd.io/docs/v3.5/install/) 的指南在本地
      启动一个 `etcd-server`

### 运行示例

在开发或运行示例时，任何与你共享核心服务（`etcd` 与 `nats.io`）的进程或用户都将
处于同一个分布式运行时中。

当前的示例使用了硬编码的 `namespace`。后续我们会处理 `namespace` 冲突问题。

所有示例都需要 `etcd` 和 `nats.io` 这两项前置服务处于运行状态。

#### Rust `hello_world`

打开两个终端，在第一个窗口中：

```
cd examples/hello_world
cargo run --bin server
```

在第二个终端中执行：

```
cd examples/hello_world
cargo run --bin client
```

输出大致如下：
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

详情请参见 [README.md](/lib/bindings/python/README.md)

Python 与 Rust 的 `hello_world` 客户端 / 服务端示例是可以互通的，因此你可以启动
Python 的 `server.py`，然后用 Rust 的 `client` 与之通信。
