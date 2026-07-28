<!--
SPDX-FileCopyrightText: Copyright (c) 2024-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->

# Dynamo Python 绑定

Dynamo 运行时系统的 Python 绑定，为机器学习工作负载提供分布式计算能力。

## 🚀 快速开始

1. 安装 `uv`：https://docs.astral.sh/uv/#getting-started
```
curl -LsSf https://astral.sh/uv/install.sh | sh
```

2. 安装 `protoc` protobuf 编译器：https://grpc.io/docs/protoc-installation/。

例如，在 Ubuntu/Debian 系统上：
```
apt install protobuf-compiler
```

3. 创建 virtualenv

```
uv venv
source .venv/bin/activate
uv pip install maturin
```

4. 构建并安装 dynamo wheel
```
maturin develop --uv
```

## 运行示例

### 前置条件

参见 [README.md](../../../docs/development/runtime-guide.md#prerequisites)。

### Hello World 示例

1. 启动 3 个独立 shell，并在每个 shell 中激活虚拟环境：
```
source .venv/bin/activate
```

2. 在第一个 shell（shell 1）中，运行示例服务端 instance-1：
```
python3 ./examples/hello_world/server.py
```

3.（可选）在另一个 shell（shell 2）中，运行示例服务端 instance-2：
```
python3 ./examples/hello_world/server.py
```

4. 在最后一个 shell（shell 3）中，运行示例客户端：
```
python3 ./examples/hello_world/client.py
```

如果你快速连续运行示例客户端，并且上文启动了多个服务端实例，你应该会看到客户端发出的请求被分配到不同服务端实例上（每个服务端的输出中可见）。如果只启动了一个服务端实例，则每次请求都会发到该服务端。

## 性能

同步 Python 与 Rust 异步运行时所带来的性能影响，是优化高并发、并行分布式系统性能时的关键考量。

Python GIL 是一个全局临界区，最终是并行的杀手。雪上加霜的是，当 Rust 异步 future 就绪时，需要谨慎处理在异步事件循环中获取 GIL 的方式。在高负载下，在事件循环线程上获取 GIL 或执行 CPU 密集型任务，会导致其他异步任务无法获得 CPU 资源。然而，执行 `tokio::task::spawn_blocking` 也并非没有开销。

如果你需要在 Python 与 Rust 事件循环之间频繁来回传递大量小消息，且 Rust 端需要访问 GIL，那么把代码从 Python 迁移到 Rust 通常可以带来显著的性能收益。
