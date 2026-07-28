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

# Dynamo Components

本目录包含构成 Dynamo 推理框架的核心组件。每个组件在分布式 LLM 服务架构中各司其职，共同实现跨多节点、多 GPU 的高吞吐、低延迟推理。

## 核心组件

### Backends

Dynamo 支持多种推理引擎，每种引擎都有各自的部署配置与能力：

- **[vLLM](/docs/backends/vllm/README.md)** —— 完整集成的 vLLM，支持解耦（disaggregated）服务、KV 感知路由、基于 SLA 的 planning、原生 KV cache 事件，以及基于 NIXL 的传输机制
- **[SGLang](/docs/backends/sglang/README.md)** —— SGLang 引擎集成，使用基于 ZMQ 的通信，支持解耦服务与 KV 感知路由
- **[TensorRT-LLM](/docs/backends/trtllm/README.md)** —— TensorRT-LLM 集成，具备解耦服务能力以及 TensorRT 加速

每个引擎在 [examples](../examples/backends/) 目录中都提供了对应不同部署模式的启动脚本与部署脚本。


### [Frontend](src/dynamo/frontend/)

Frontend 组件提供 HTTP API 层与请求处理：

- **OpenAI 兼容的 HTTP server** —— 用于 LLM 推理请求的 RESTful API endpoint
- **Pre-processor** —— 负责请求的预处理与校验
- **Router** —— 根据负载与 KV cache 状态把请求路由到合适的 worker
- **Auto-discovery** —— 自动发现并注册可用 worker

### [Planner](src/dynamo/planner/)

Planner 组件监控系统状态并动态调整 worker 分配：

- **Dynamic scaling** —— 基于 metrics 对 prefill/decode worker 进行扩缩容
- **SLA-based planning** —— 确保达成推理性能目标
- **Load-based planning** —— 根据需求优化资源利用率

## 快速开始

上手 Dynamo 组件的步骤：

1. **选择推理引擎** —— 从受支持的 backend 中挑一个
2. **启动所需服务** —— 使用 Docker Compose 启动 etcd 与 NATS
3. **配置所选引擎** —— 使用 Python wheel 安装或构建镜像
4. **运行部署脚本** —— 从该引擎的 launch 目录中执行
5. **监控性能** —— 使用 metrics 组件

详细说明请参见各组件目录下的 README 以及主 [Dynamo 文档](../docs/)。

