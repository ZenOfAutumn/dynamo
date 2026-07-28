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

# Dynamo 示例

本目录包含一系列实用示例，演示如何部署并使用 Dynamo 进行分布式 LLM 推理（inference）。每个示例都提供安装步骤、配置文件以及说明，帮助你理解不同的部署模式与使用场景。

> **想看到某个特定示例？**
> 欢迎[提交 GitHub issue](https://github.com/ai-dynamo/dynamo/issues) 提出你希望看到的示例，或者直接[发起 pull request](https://github.com/ai-dynamo/dynamo/pulls) 贡献你自己的示例！

## 基础与教程

通过下列入门示例学习 Dynamo 的核心概念：

- **[Quickstart](https://docs.nvidia.com/dynamo/getting-started/quickstart)** - 在多种受支持后端（backend）下的简单本地 Dynamo 安装
- **[Disaggregated Serving](/docs/features/disaggregated-serving/README.md)** - 通过预填充/解码（prefill/decode）解耦提升性能与可扩展性
- **[Multi-node TensorRT-LLM](/docs/backends/trtllm/multinode/trtllm-multinode-examples.md)** - 跨多节点、多 GPU 的分布式推理

## 框架支持

下列示例展示了 Dynamo 在主流推理引擎上的整体使用方式。

如果你希望了解更进阶的、特定框架的部署模式与最佳实践，请参考 [Examples Backends](/examples/backends/) 目录：
- **[vLLM](/examples/backends/vllm/)** – vLLM 专属的部署与配置
- **[SGLang](/examples/backends/sglang/)** – SGLang 集成示例与工作流
- **[TensorRT-LLM](/examples/backends/trtllm/)** – TensorRT-LLM 工作流与优化

## 部署示例

面向生产环境、平台相关的部署指南：

- **[Amazon EKS](/examples/deployments/EKS/)** - 在 Amazon Elastic Kubernetes Service 上部署 Dynamo
- **[Azure AKS](/examples/deployments/AKS/)** - 在 Azure Kubernetes Service 上部署 Dynamo
- **[Amazon ECS](/examples/deployments/ECS/)** - 在 Amazon Elastic Container Service 上部署 Dynamo
- **[Google GKE](/examples/deployments/GKE/)** - 在 Google Kubernetes Engine 上部署 Dynamo

## 运行时（Runtime）示例

面向使用 Python<>Rust 绑定的开发者的底层运行时示例：

- **[Hello World](/examples/custom_backend/hello_world/README.md)** - 演示基本概念的极简 Dynamo 运行时服务

## 上手指南

1. **选择你的部署模式**：从 [Quickstart](https://docs.nvidia.com/dynamo/getting-started/quickstart) 开始进行简单的本地部署，或者探索 [Disaggregated Serving](/docs/features/disaggregated-serving/README.md) 了解更进阶的架构。

2. **准备前置条件**：大多数示例依赖 etcd 与 NATS 服务。可使用如下命令启动：
   ```bash
   docker compose -f deploy/docker-compose.yml up -d
   ```

3. **跟随示例操作**：每个目录都包含针对该部署模式的详细安装说明与配置文件。

## 前置条件

在运行任何示例前，请确认你已具备：

- **Docker 与 Docker Compose** - 用于运行容器化服务
- **CUDA 兼容 GPU** - 用于 LLM 推理（hello_world 例外，它不依赖 GPU）
- **Python 3.9+** - 用于客户端脚本与工具

### Kubernetes 部署

如果你要运行 Kubernetes/云部署示例（EKS、AKS、GKE），还需要：

| 工具         | 最低版本     | 安装方式                                                                  |
|--------------|--------------|---------------------------------------------------------------------------|
| **kubectl**  | v1.24+       | [安装 kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)           |
| **Helm**     | v3.0+        | [安装 Helm](https://helm.sh/docs/intro/install/)                          |

详细安装说明与部署前检查项，请参阅 [Kubernetes 安装指南](/docs/kubernetes/installation-guide.md#prerequisites)。
