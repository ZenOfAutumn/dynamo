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

![Dynamo banner](./docs/assets/img/dynamo-frontpage-banner.png)

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![GitHub Release](https://img.shields.io/github/v/release/ai-dynamo/dynamo)](https://github.com/ai-dynamo/dynamo/releases/latest)
[![PyPI](https://img.shields.io/pypi/v/ai-dynamo)](https://pypi.org/project/ai-dynamo/)
[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/ai-dynamo/dynamo)
[![Discord](https://dcbadge.limes.pink/api/server/D92uqZRjCZ?style=flat)](https://discord.gg/D92uqZRjCZ)
![Community Contributors](https://img.shields.io/badge/community_contributors-70%2B-brightgreen)

| **[文档](https://docs.nvidia.com/dynamo/)** | **[路线图](https://github.com/ai-dynamo/dynamo/issues/5506)** | **[Recipes](https://github.com/ai-dynamo/dynamo/tree/main/recipes)** | **[示例](https://github.com/ai-dynamo/dynamo/tree/main/examples)** | **[预构建容器](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/collections/ai-dynamo)** | **[Digest](docs/digest/index.mdx)** | **[设计提案](https://github.com/ai-dynamo/enhancements)** | **[如何贡献](#社区与贡献)** |

> 📘 English version: [README.md](./README.md)

# Dynamo

<!-- 临时横幅：V4 recipes 成熟后可移除。 -->
> [!NOTE]
> **Day-0 DeepSeek-V4 recipes 已就绪。** 已通过测试的 Kubernetes 部署方案：[DeepSeek-V4-Pro](recipes/deepseek-v4/deepseek-v4-pro/) 与 [DeepSeek-V4-Flash](recipes/deepseek-v4/deepseek-v4-flash/)，已合入 main 分支，同时支持 **vLLM** 与 **SGLang**，并在 NGC 上发布了预构建的 SGLang 容器镜像。

**开源、面向数据中心规模的推理栈。** Dynamo 是位于推理引擎之上的编排层 —— 它不替代 SGLang、TensorRT-LLM 或 vLLM，而是把它们组合成一个协同工作的多节点推理系统。Dynamo 通过分离式服务（disaggregated serving）、智能路由、多层级 KV 缓存以及自动伸缩协同配合，为 LLM、推理（reasoning）、多模态和视频生成工作负载最大化吞吐、最小化延迟。

底层使用 Rust 构建以获得高性能，使用 Python 提供良好的可扩展性。

## 何时使用 Dynamo

- 你需要在**多 GPU 或多节点**上服务 LLM，并对它们进行协调
- 你希望使用 **KV 感知路由**避免重复的 prefill 计算
- 你需要**独立伸缩 prefill 与 decode**（分离式服务）
- 你希望在最低 TCO（总拥有成本）下通过**自动伸缩**满足延迟 SLA
- 你需要在扩容新副本时获得**快速冷启动**

如果只是在单 GPU 上跑单个模型，单纯使用推理引擎本身通常就够了。

**功能支持一览：**

| | [SGLang](https://docs.nvidia.com/dynamo/backends/sg-lang) | [TensorRT-LLM](https://docs.nvidia.com/dynamo/backends/tensor-rt-llm) | [vLLM](https://docs.nvidia.com/dynamo/backends/v-llm) |
|---|:----:|:----------:|:--:|
| [**分离式服务**](https://docs.nvidia.com/dynamo/design-docs/disaggregated-serving) | ✅ | ✅ | ✅ |
| [**KV 感知路由**](https://docs.nvidia.com/dynamo/components/router) | ✅ | ✅ | ✅ |
| [**SLA 驱动的 Planner**](https://docs.nvidia.com/dynamo/components/planner/planner-guide) | ✅ | ✅ | ✅ |
| [**KVBM**](https://docs.nvidia.com/dynamo/components/kvbm) | 🚧 | ✅ | ✅ |
| [**多模态**](https://docs.nvidia.com/dynamo/user-guides/multimodal) | ✅ | ✅ | ✅ |
| [**Tool Calling**](docs/tool-calling/README.md) | ✅ | ✅ | ✅ |

> **[完整功能矩阵 →](https://docs.nvidia.com/dynamo/resources/feature-matrix)** —— 涵盖 LoRA、请求迁移、投机解码及功能间交互。

## 关键成果

| 成果 | 上下文 |
|--------|---------|
| 单卡吞吐 **7 倍** 提升 | DeepSeek R1 在 GB200 NVL72 + Dynamo 上对比未启用 Dynamo 的 B200（[InferenceX](https://inferencex.semianalysis.com/)） |
| 模型启动 **7 倍** 加速 | ModelExpress 权重流式加载（DeepSeek-V3 在 H200 上） |
| 首 token 时延（TTFT）**2 倍** 提升 | KV 感知路由，Qwen3-Coder 480B（[Baseten benchmark](https://www.baseten.co/blog/how-baseten-achieved-2x-faster-inference-with-nvidia-dynamo/)） |
| SLA 违例减少 **80%** | Planner 自动伸缩，TCO 同时降低 5%（[阿里云栖大会 2025 @ 2:50:00](https://yunqi.aliyun.com/2025/session?agendaId=6062)） |
| 吞吐 **750 倍** 提升 | DeepSeek-R1 在 GB300 NVL72 上（[InferenceXv2](https://inferencex.semianalysis.com/)） |


## Dynamo 做了什么

大多数推理引擎只优化单 GPU 或单节点。Dynamo 是**它们之上的编排层** —— 它把一组 GPU 变成一个协同工作的推理系统。

<p align="center">
  <img src="./docs/assets/img/dynamo-readme-overview.svg" alt="Dynamo architecture overview" width="600" />
</p>

**[架构深入解读 →](https://docs.nvidia.com/dynamo/design-docs/overall-architecture)**

### 核心能力

| 能力 | 它做什么 | 为何重要 |
|------------|-------------|----------------|
| [**Prefill / Decode 分离**](https://docs.nvidia.com/dynamo/design-docs/disaggregated-serving) | 将 prefill 与 decode 拆分到可独立伸缩的 GPU 池 | 最大化 GPU 利用率；每个阶段都跑在最适合自身工作负载的硬件上 |
| [**KV 感知路由**](https://docs.nvidia.com/dynamo/components/router) | 基于 worker 负载与 KV 缓存重叠度路由请求 | 消除重复 prefill 计算 —— TTFT 提升 2 倍 |
| [**KV Block Manager (KVBM)**](https://docs.nvidia.com/dynamo/components/kvbm) | 将 KV 缓存逐级卸载到 GPU → CPU → SSD → 远端存储 | 突破 GPU 显存限制，扩展有效上下文长度 |
| [**ModelExpress**](https://github.com/ai-dynamo/modelexpress) | 通过 NIXL/NVLink 在 GPU 间流式传输模型权重 | 新副本冷启动加速 7 倍 |
| [**Planner**](https://docs.nvidia.com/dynamo/components/planner/planner-guide) | SLA 驱动的自动伸缩器，会对工作负载做 profile 并合理调整资源池 | 在最低 TCO 下满足延迟目标 |
| [**Grove**](https://github.com/ai-dynamo/grove) | 用于拓扑感知 gang scheduling 的 K8s Operator（NVL72） | 在机柜、主机和 NUMA 节点之间最优地放置工作负载 |
| [**AIConfigurator**](https://github.com/ai-dynamo/aiconfigurator) | 数秒内仿真 10K+ 部署配置 | 不烧 GPU 时长就能找到最优的服务化配置 |
| [**容错**](https://docs.nvidia.com/dynamo/user-guides/fault-tolerance/request-migration) | 金丝雀健康检查 + 在途请求迁移 | worker 可以失败，但用户请求不会失败 |

### 1.0 新特性

- **零配置部署 ([DGDR](https://docs.nvidia.com/dynamo/kubernetes-deployment/deployment-guide/dgdr-reference))** *(beta)*：在一份 YAML 中指定模型、硬件和 SLA —— AIConfigurator 自动 profile 工作负载，Planner 优化拓扑，Dynamo 完成部署
- **Agentic 推理：** 单请求级提示，可设置延迟优先级、预期输出长度、缓存 pin 的 TTL。已集成 [LangChain](https://docs.langchain.com/oss/python/integrations/chat/nvidia_ai_endpoints#use-with-nvidia-dynamo) 与 [NeMo Agent Toolkit](https://github.com/NVIDIA/NeMo-Agent-Toolkit)
- **多模态 E/P/D：** Encode / Prefill / Decode 三段分离 + embedding 缓存 —— 图像类工作负载 TTFT 提升 30%
- **视频生成：** 原生支持 [FastVideo](https://github.com/hao-ai-lab/FastVideo) 与 [SGLang Diffusion](https://lmsys.org/blog/2026-02-16-sglang-diffusion-advanced-optimizations/) —— 单张 B200 上实时 1080p
- **K8s Inference Gateway 插件：** 在标准 Kubernetes Gateway 内置 KV 感知路由
- **存储层 KV 卸载：** 支持 S3/Azure Blob，并通过全局 KV 事件实现集群范围的缓存可见性

## 快速上手

### 方式 A：容器（最快）

```bash
# 拉取预构建容器（以 SGLang 为例）
docker run --gpus all --network host --rm -it nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.1.1

# 在容器内 —— 启动 frontend 与 worker
python3 -m dynamo.frontend --http-port 8000 --discovery-backend file > /dev/null 2>&1 &
python3 -m dynamo.sglang --model-path Qwen/Qwen3-0.6B --discovery-backend file &

# 发送一次请求
curl -s localhost:8000/v1/chat/completions -H "Content-Type: application/json" -d '{
  "model": "Qwen/Qwen3-0.6B",
  "messages": [{"role": "user", "content": "Hello!"}],
  "max_tokens": 100
}' | jq
```

同样可用：[`tensorrtllm-runtime:1.1.1`](https://docs.nvidia.com/dynamo/resources/release-artifacts) 与 [`vllm-runtime:1.1.1`](https://docs.nvidia.com/dynamo/resources/release-artifacts)。

### 方式 B：从 PyPI 安装

先安装 [uv](https://github.com/astral-sh/uv)（`curl -LsSf https://astral.sh/uv/install.sh | sh`），然后：

```bash
uv pip install --prerelease=allow "ai-dynamo[sglang]"   # 或 [vllm]
```

> **注意：** TensorRT-LLM 需要使用 `pip` 并附加 `--extra-index-url https://pypi.nvidia.com`。详见 [安装指南](docs/getting-started/local-installation.md) 中关于 TRT-LLM 的说明。

之后按上面的方式启动 frontend 与 worker。系统依赖与各后端的具体说明详见 [完整安装指南](docs/getting-started/local-installation.md)。

### 方式 C：Kubernetes（推荐）

对于生产环境的多节点集群，请安装 [Dynamo Platform](https://docs.nvidia.com/dynamo/kubernetes-deployment/deployment-guide)，并通过单一 manifest 部署：

```yaml
# 零配置部署：指定模型 + SLA，其余交给 Dynamo
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: my-model
spec:
  model: Qwen/Qwen3-0.6B
  backend: vllm
  sla:
    ttft: 200.0   # ms
    itl: 20.0     # ms
  autoApply: true
```

常见模型的预置 recipes：

| 模型 | 框架 | 模式 | Recipe |
|-------|-----------|------|--------|
| Llama-3-70B | vLLM | 聚合（Aggregated） | [查看](recipes/llama-3-70b/vllm/) |
| DeepSeek-R1 | SGLang | 分离式（Disaggregated） | [查看](recipes/deepseek-r1/sglang/) |
| Qwen3-32B-FP8 | TensorRT-LLM | 聚合（Aggregated） | [查看](recipes/qwen3-32b-fp8/trtllm/) |

完整列表见 [recipes/](recipes/README.md)。云厂商专属指南：[AWS EKS](examples/deployments/EKS/) · [Google GKE](examples/deployments/GKE/)

## 从源码构建

适用于希望本地构建与开发的贡献者。详见 [完整构建指南](docs/getting-started/building-from-source.md)。

```bash
# 安装系统依赖（Ubuntu 24.04）
sudo apt install -y build-essential libhwloc-dev libudev-dev pkg-config libclang-dev protobuf-compiler python3-dev cmake

# 安装 Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh && source $HOME/.cargo/env

# 创建 venv 并构建
uv venv dynamo && source dynamo/bin/activate
uv pip install pip maturin
cd lib/bindings/python && maturin develop --uv && cd $PROJECT_ROOT
uv pip install -e lib/gpu_memory_service
uv pip install -e .
```

> VSCode/Cursor 用户：参见 [`.devcontainer`](.devcontainer/README.md) 获取一份预配置的开发环境。

## 社区与贡献

Dynamo 采用 OSS 优先的开放开发模式，欢迎各种形式的贡献。

- **[贡献指南](https://docs.nvidia.com/dynamo/getting-started/contribution-guide)** —— 如何贡献代码、文档和 recipes
- **[设计提案](https://github.com/ai-dynamo/enhancements)** —— 重大特性的 RFC
- **[Office Hours](https://www.youtube.com/playlist?list=PL5B692fm6--tgryKu94h2Zb7jTFM3Go4X)** —— 双周直播
- **[社区会议](https://docs.google.com/document/d/1uR8xD_hlYGwV6QspvSc36k1H-wo1BUcVmFbHH9xlXd8/view)** —— 每周（PT 周三 10:30 AM）开发者社区会议
- **[Discord](https://discord.gg/D92uqZRjCZ)** —— 与团队和社区交流
- **[Dynamo Day 录播](https://nvevents.nvidia.com/dynamoday)** —— 来自生产用户的深度分享

## 最新动态

- [03/15] [Dynamo 1.0 发布 —— 已可投入生产，社区采用度强劲](https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models/)
- [03/15] [NVIDIA Blackwell Ultra 在 MLPerf 推理基准测试中创造新纪录](https://developer.nvidia.com/blog/nvidia-blackwell-ultra-sets-new-inference-records-in-mlperf-debut/)
- [03/15] [NVIDIA Blackwell 在 SemiAnalysis InferenceMax 基准测试中领先](https://developer.nvidia.com/blog/nvidia-blackwell-leads-on-new-semianalysis-inferencemax-benchmarks/)
- [12/05] [月之暗面 Kimi K2 借助 Dynamo 在 GB200 上获得 10 倍推理加速](https://quantumzeitgeist.com/kimi-k2-nvidia-ai-ai-breakthrough/)
- [12/02] [Mistral AI 借助 Dynamo 在 Mistral Large 3 上实现 10 倍推理加速](https://www.marktechpost.com/2025/12/02/nvidia-and-mistral-ai-bring-10x-faster-inference-for-the-mistral-3-family-on-gb200-nvl72-gpu-systems/)
- [11/20] [Dell 通过 NIXL 集成 PowerScale，TTFT 提升 19 倍](https://www.dell.com/en-us/dt/corporate/newsroom/announcements/detailpage.press-releases~usa~2025~11~dell-technologies-and-nvidia-advance-enterprise-ai-innovation.htm)

<details>
<summary>更早动态</summary>

Dynamo 提供完善的基准测试工具：

- **[基准测试指南](docs/benchmarks/benchmarking.md)** – 使用 AIPerf 比较不同部署拓扑
- **[SLA 驱动部署](docs/components/planner/planner-guide.md)** – 优化部署以满足 SLA 要求

## Frontend OpenAPI 规范

OpenAI 兼容 frontend 在 `/openapi.json` 上暴露了 OpenAPI 3 规范。无需启动服务即可生成：

```bash
cargo run -p dynamo-llm --bin generate-frontend-openapi
```

文件会写入 `docs/reference/api/openapi.json`。

## 服务发现与消息

Dynamo 在组件间通信使用 TCP。在 Kubernetes 上，使用原生资源（[CRDs + EndpointSlices](docs/kubernetes/service-discovery.md)）做服务发现。对大多数部署而言，外部组件是可选的：

| 部署形态 | etcd | NATS | 说明 |
|------------|------|------|-------|
| **本地开发** | ❌ 不需要 | ❌ 不需要 | 传入 `--discovery-backend file`；vLLM 还需 `--kv-events-config '{"enable_kv_cache_events": false}'` |
| **Kubernetes** | ❌ 不需要 | ❌ 不需要 | K8s 原生服务发现；TCP 请求面 |

> **注意：** KV 感知路由需要 NATS 来协调前缀缓存。

对于 Slurm 或其他分布式部署（以及 KV 感知路由）：

- [etcd](https://etcd.io/) 可直接以 `./etcd` 形式运行。
- [nats](https://nats.io/) 需启用 JetStream：`nats-server -js`。

一键起两者：`docker compose -f dev/docker-compose.yml up -d`

## 更多动态

- [11/20] [Dell 通过 Dynamo 的 NIXL 集成 PowerScale，TTFT 提升 19 倍](https://www.dell.com/en-us/dt/corporate/newsroom/announcements/detailpage.press-releases~usa~2025~11~dell-technologies-and-nvidia-advance-enterprise-ai-innovation.htm)
- [11/20] [WEKA 与 NVIDIA 合作为 Dynamo 提供 KV 缓存存储](https://siliconangle.com/2025/11/20/nvidia-weka-kv-cache-solution-ai-inferencing-sc25/)
- [11/13] [Dynamo Office Hours 播放列表](https://www.youtube.com/playlist?list=PL5B692fm6--tgryKu94h2Zb7jTFM3Go4X)
- [10/16] [Baseten 借助 NVIDIA Dynamo 实现 2 倍推理加速](https://www.baseten.co/blog/how-baseten-achieved-2x-faster-inference-with-nvidia-dynamo/)
- [12/01] [InfoQ：NVIDIA Dynamo 简化 LLM 推理的 Kubernetes 部署](https://www.infoq.com/news/2025/12/nvidia-dynamo-kubernetes/)

</details>

## 参考资料

- **[支持矩阵](https://docs.nvidia.com/dynamo/resources/support-matrix)** —— 硬件、操作系统、CUDA 与各后端版本
- **[功能矩阵](https://docs.nvidia.com/dynamo/resources/feature-matrix)** —— 各后端兼容性细节
- **[发布产物](https://docs.nvidia.com/dynamo/resources/release-artifacts)** —— 容器、wheel、Helm chart
- **[服务发现](https://docs.nvidia.com/dynamo/kubernetes-deployment/deployment-guide/service-discovery)** —— K8s 原生 vs etcd vs 文件式发现
- **[基准测试指南](https://docs.nvidia.com/dynamo/user-guides/benchmarking)** —— 使用 AIPerf 对比不同部署拓扑

<!-- 功能兼容矩阵的引用链接 -->
[disagg]: docs/design-docs/disagg-serving.md
[kv-routing]: docs/components/router/README.md
[planner]: docs/components/planner/planner-guide.md
[kvbm]: docs/components/kvbm/README.md
[migration]: docs/fault-tolerance/request-migration.md
[lora]: examples/backends/vllm/deploy/lora/README.md
[tools]: docs/tool-calling/README.md

