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

| **[文档](https://docs.nvidia.com/dynamo/)** | **[路线图](https://github.com/ai-dynamo/dynamo/issues/5506)** | **[Recipes](https://github.com/ai-dynamo/dynamo/tree/main/recipes)** | **[示例](https://github.com/ai-dynamo/dynamo/tree/main/examples)** | **[预构建容器](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/collections/ai-dynamo)** | **[摘要文集](docs/digest/index.mdx)** | **[设计提案](https://github.com/ai-dynamo/enhancements)** | **[如何贡献](#社区与贡献)** |

# Dynamo

<!-- TEMPORARY BANNER: remove once V4 recipes mature. -->
> [!NOTE]
> **Day-0 DeepSeek-V4 recipes 已可用。** 已验证的 [DeepSeek-V4-Pro](recipes/deepseek-v4/deepseek-v4-pro/) 与 [DeepSeek-V4-Flash](recipes/deepseek-v4/deepseek-v4-flash/) Kubernetes 部署路径已合入 main，覆盖 **vLLM** 与 **SGLang**，并在 NGC 上发布了 SGLang 的预构建容器镜像。

**面向数据中心规模的开源推理栈。** Dynamo 是位于推理引擎之上的编排层——它并不替代 SGLang、TensorRT-LLM 或 vLLM，而是把它们组织成一套协同工作的多节点推理系统。分离式服务（Disaggregated Serving）、智能路由、多层 KV 缓存与自动扩缩容协同发力，为 LLM、推理（reasoning）、多模态以及视频生成等工作负载最大化吞吐、最小化延迟。

底层使用 Rust 构建以获得高性能，上层使用 Python 以保持可扩展性。

## 何时使用 Dynamo

- 你正在跨**多块 GPU 或多台节点**提供 LLM 服务，需要协调它们
- 你希望使用 **KV 感知路由（KV-aware routing）** 来避免重复的 prefill 计算
- 你需要 **独立扩缩容 prefill 与 decode**（分离式服务）
- 你希望以 **最低 TCO（总拥有成本）** 满足延迟 SLA 的 **自动扩缩容**
- 你需要在拉起新副本时实现 **快速冷启动**

如果你只在单卡 GPU 上运行单个模型，单独使用一个推理引擎通常就够了。

**功能特性一览：**

| | [SGLang](https://docs.nvidia.com/dynamo/backends/sg-lang) | [TensorRT-LLM](https://docs.nvidia.com/dynamo/backends/tensor-rt-llm) | [vLLM](https://docs.nvidia.com/dynamo/backends/v-llm) |
|---|:----:|:----------:|:--:|
| [**分离式服务**](https://docs.nvidia.com/dynamo/design-docs/disaggregated-serving) | ✅ | ✅ | ✅ |
| [**KV 感知路由**](https://docs.nvidia.com/dynamo/components/router) | ✅ | ✅ | ✅ |
| [**基于 SLA 的 Planner**](https://docs.nvidia.com/dynamo/components/planner/planner-guide) | ✅ | ✅ | ✅ |
| [**KVBM**](https://docs.nvidia.com/dynamo/components/kvbm) | 🚧 | ✅ | ✅ |
| [**多模态**](https://docs.nvidia.com/dynamo/user-guides/multimodal) | ✅ | ✅ | ✅ |
| [**工具调用**](https://docs.nvidia.com/dynamo/user-guides/agents/tool-calling) | ✅ | ✅ | ✅ |

> **[完整功能矩阵 →](https://docs.nvidia.com/dynamo/resources/feature-matrix)** —— 包含 LoRA、请求迁移、投机解码以及各项功能之间的相互兼容性。

## 关键成果

| 成果 | 背景 |
|--------|---------|
| 单 GPU 吞吐提升 **7 倍** | DeepSeek R1，GB200 NVL72 + Dynamo vs B200 不使用 Dynamo（[InferenceX](https://inferencex.semianalysis.com/)） |
| 模型启动加速 **7 倍** | ModelExpress 权重流式加载（H200 上的 DeepSeek-V3） |
| 首 token 时间（TTFT）加速 **2 倍** | KV 感知路由，Qwen3-Coder 480B（[Baseten 基准](https://www.baseten.co/blog/how-baseten-achieved-2x-faster-inference-with-nvidia-dynamo/)） |
| SLA 违规减少 **80%** | Planner 自动扩缩容，TCO 降低 5%（[阿里云 APSARA 2025 @ 2:50:00](https://yunqi.aliyun.com/2025/session?agendaId=6062)） |
| 吞吐提升 **750 倍** | DeepSeek-R1，GB300 NVL72（[InferenceXv2](https://inferencex.semianalysis.com/)） |


## Dynamo 做什么

大多数推理引擎专注于优化单卡或单节点。Dynamo 是位于其**之上的编排层**——把一整片 GPU 集群整合成一套协同的推理系统。

<p align="center">
  <img src="./docs/assets/img/dynamo-readme-overview.svg" alt="Dynamo architecture overview" width="600" />
</p>

**[架构深度解读 →](https://docs.nvidia.com/dynamo/design-docs/overall-architecture)**

### 核心能力

| 能力 | 作用 | 价值所在 |
|------------|-------------|----------------|
| [**Prefill/Decode 分离**](https://docs.nvidia.com/dynamo/design-docs/disaggregated-serving) | 把 prefill 与 decode 拆分到可独立扩缩容的 GPU 池 | 最大化 GPU 利用率；每个阶段都跑在适配其负载的硬件上 |
| [**KV 感知路由**](https://docs.nvidia.com/dynamo/components/router) | 基于 worker 负载与 KV 缓存重叠度路由请求 | 消除重复的 prefill 计算 —— TTFT 提升 2 倍 |
| [**KV Block Manager（KVBM）**](https://docs.nvidia.com/dynamo/components/kvbm) | 把 KV 缓存逐层下放：GPU → CPU → SSD → 远端存储 | 把有效上下文长度扩展到 GPU 显存之外 |
| [**ModelExpress**](https://github.com/ai-dynamo/modelexpress) | 通过 NIXL/NVLink 在 GPU 之间流式传输模型权重 | 新副本冷启动加速 7 倍 |
| [**Planner**](https://docs.nvidia.com/dynamo/components/planner/planner-guide) | 由 SLA 驱动的自动扩缩容器，对工作负载画像并按需调整池规模 | 在最低 TCO 下满足延迟目标 |
| [**Grove**](https://github.com/ai-dynamo/grove) | 面向 K8s 的拓扑感知 gang 调度（NVL72） | 在机柜、主机和 NUMA 节点间最优放置工作负载 |
| [**AIConfigurator**](https://github.com/ai-dynamo/aiconfigurator) | 数秒内模拟超过 10K 种部署配置 | 找到最优服务配置，无需烧 GPU 时长 |
| [**容错**](https://docs.nvidia.com/dynamo/user-guides/fault-tolerance/request-migration) | 金丝雀健康检查 + 在飞请求迁移 | Worker 可能挂，但用户请求不会挂 |

### 1.0 新特性

- **零配置部署（[DGDR](https://docs.nvidia.com/dynamo/kubernetes-deployment/deployment-guide/dgdr-reference)）**（beta）：在一份 YAML 中指定模型、硬件与 SLA —— AIConfigurator 自动画像负载，Planner 优化拓扑，Dynamo 完成部署
- **Agentic 推理**：按请求传入延迟优先级、预期输出长度、缓存固定 TTL 等提示。已集成 [LangChain](https://docs.langchain.com/oss/python/integrations/chat/nvidia_ai_endpoints#use-with-nvidia-dynamo) 与 [NeMo Agent Toolkit](https://github.com/NVIDIA/NeMo-Agent-Toolkit)
- **多模态 E/P/D**：编码（encode）/prefill/decode 分离 + embedding 缓存 —— 图像类负载 TTFT 提升 30%
- **视频生成**：原生支持 [FastVideo](https://github.com/hao-ai-lab/FastVideo) 与 [SGLang Diffusion](https://lmsys.org/blog/2026-02-16-sglang-diffusion-advanced-optimizations/) —— 单卡 B200 实时生成 1080p
- **K8s Inference Gateway 插件**：在标准 Kubernetes 网关内提供 KV 感知路由
- **存储级 KV 卸载**：支持 S3/Azure blob，并提供集群范围的全局 KV 事件以观测整集群缓存状态

## 快速开始

### 方式 A：容器（最快）

```bash
# 拉取预构建容器（以 SGLang 为例）
docker run --gpus all --network host --rm -it nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.1.1

# 在容器内 —— 启动前端与 worker
python3 -m dynamo.frontend --http-port 8000 --discovery-backend file > /dev/null 2>&1 &
python3 -m dynamo.sglang --model-path Qwen/Qwen3-0.6B --discovery-backend file &

# 发送一次请求
curl -s localhost:8000/v1/chat/completions -H "Content-Type: application/json" -d '{
  "model": "Qwen/Qwen3-0.6B",
  "messages": [{"role": "user", "content": "Hello!"}],
  "max_tokens": 100
}' | jq
```

也提供：[`tensorrtllm-runtime:1.1.1`](https://docs.nvidia.com/dynamo/resources/release-artifacts) 与 [`vllm-runtime:1.1.1`](https://docs.nvidia.com/dynamo/resources/release-artifacts)。

### 方式 B：从 PyPI 安装

先安装 [uv](https://github.com/astral-sh/uv)（`curl -LsSf https://astral.sh/uv/install.sh | sh`），然后：

```bash
uv pip install --prerelease=allow "ai-dynamo[sglang]"   # 或 [vllm]
```

> **注意：** TensorRT-LLM 需要 `pip` 并附带 `--extra-index-url https://pypi.nvidia.com`。TRT-LLM 专属说明请参见 [安装指南](docs/getting-started/local-installation.md)。

随后按上文示例启动前端与 worker。系统依赖与各后端的具体说明请参见 [完整安装指南](docs/getting-started/local-installation.md)。

### 方式 C：Kubernetes（推荐）

面向生产多节点集群，请安装 [Dynamo Platform](https://docs.nvidia.com/dynamo/kubernetes-deployment/deployment-guide)，然后用一份 manifest 即可部署：

```yaml
# 零配置部署：指定模型 + SLA，剩下的交给 Dynamo
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
| DeepSeek-R1 | SGLang | 分离（Disaggregated） | [查看](recipes/deepseek-r1/sglang/) |
| Qwen3-32B-FP8 | TensorRT-LLM | 聚合（Aggregated） | [查看](recipes/qwen3-32b-fp8/trtllm/) |

完整列表见 [recipes/](recipes/README.md)。云上部署指南：[AWS EKS](examples/deployments/EKS/) · [Google GKE](examples/deployments/GKE/)

## 从源码构建

面向希望本地构建与开发的贡献者。详见 [完整构建指南](docs/getting-started/building-from-source.md)。

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

> VSCode/Cursor 用户：可参见 [`.devcontainer`](.devcontainer/README.md) 获取预配置的开发环境。

## 社区与贡献

Dynamo 以开源优先（OSS-first）的开发模式公开构建。欢迎各种形式的贡献。

- **[贡献指南](https://docs.nvidia.com/dynamo/getting-started/contribution-guide)** —— 如何贡献代码、文档与 recipes
- **[设计提案](https://github.com/ai-dynamo/enhancements)** —— 重大特性的 RFC
- **[Office Hours](https://www.youtube.com/playlist?list=PL5B692fm6--tgryKu94h2Zb7jTFM3Go4X)** —— 双周一次的线上交流
- **[社区例会](https://docs.google.com/document/d/1uR8xD_hlYGwV6QspvSc36k1H-wo1BUcVmFbHH9xlXd8/view)** – 每周（周五太平洋时间下午 2:30）研发社区会议
- **[Discord](https://discord.gg/D92uqZRjCZ)** —— 与团队和社区聊天
- **[Dynamo Day 回放](https://nvevents.nvidia.com/dynamoday)** —— 来自生产用户的深度分享

## 最新动态

- [03/15] [Dynamo 1.0 发布 —— 已具备生产可用度，社区采用度强劲](https://developer.nvidia.com/blog/introducing-nvidia-dynamo-a-low-latency-distributed-inference-framework-for-scaling-reasoning-ai-models/)
- [03/15] [NVIDIA Blackwell Ultra 在 MLPerf 上刷新推理纪录](https://developer.nvidia.com/blog/nvidia-blackwell-ultra-sets-new-inference-records-in-mlperf-debut/)
- [03/15] [NVIDIA Blackwell 在 SemiAnalysis InferenceMax 基准中领先](https://developer.nvidia.com/blog/nvidia-blackwell-leads-on-new-semianalysis-inferencemax-benchmarks/)
- [12/05] [Moonshot AI 的 Kimi K2 在 GB200 上借助 Dynamo 实现 10 倍推理加速](https://quantumzeitgeist.com/kimi-k2-nvidia-ai-ai-breakthrough/)
- [12/02] [Mistral AI 使用 Dynamo 让 Mistral Large 3 推理速度提升 10 倍](https://www.marktechpost.com/2025/12/02/nvidia-and-mistral-ai-bring-10x-faster-inference-for-the-mistral-3-family-on-gb200-nvl72-gpu-systems/)
- [11/20] [Dell 将 PowerScale 与 NIXL 集成，TTFT 提升 19 倍](https://www.dell.com/en-us/dt/corporate/newsroom/announcements/detailpage.press-releases~usa~2025~11~dell-technologies-and-nvidia-advance-enterprise-ai-innovation.htm)

<details>
<summary>更早的动态</summary>

Dynamo 提供完善的基准测试工具：

- **[基准测试指南](docs/benchmarks/benchmarking.md)** – 使用 AIPerf 对比不同部署拓扑
- **[SLA 驱动部署](docs/components/planner/planner-guide.md)** – 优化部署以满足 SLA 要求

## 前端 OpenAPI 规范

兼容 OpenAI 的前端在 `/openapi.json` 暴露 OpenAPI 3 规范。如果不想启动服务也可以生成它：

```bash
cargo run -p dynamo-llm --bin generate-frontend-openapi
```

输出会写入 `docs/reference/api/openapi.json`。

## 服务发现与消息

Dynamo 组件之间使用 TCP 通信。在 Kubernetes 上由原生资源（[CRDs + EndpointSlices](docs/kubernetes/service-discovery.md)）来做服务发现。绝大多数部署不需要外部依赖：

| 部署形态 | etcd | NATS | 备注 |
|------------|------|------|-------|
| **本地开发** | ❌ 不需要 | ❌ 不需要 | 传入 `--discovery-backend file`；vLLM 还需 `--kv-events-config '{"enable_kv_cache_events": false}'` |
| **Kubernetes** | ❌ 不需要 | ❌ 不需要 | K8s 原生发现；TCP 请求通道 |

> **注意：** KV 感知路由需要 NATS 来协调前缀缓存。

对于 Slurm 或其他分布式部署（以及 KV 感知路由）：

- [etcd](https://etcd.io/) 可以直接作为 `./etcd` 运行。
- [nats](https://nats.io/) 需要启用 JetStream：`nats-server -js`。

一键拉起两者：`docker compose -f deploy/docker-compose.yml up -d`

## 更多动态

- [11/20] [Dell 将 PowerScale 与 Dynamo 的 NIXL 集成，TTFT 提升 19 倍](https://www.dell.com/en-us/dt/corporate/newsroom/announcements/detailpage.press-releases~usa~2025~11~dell-technologies-and-nvidia-advance-enterprise-ai-innovation.htm)
- [11/20] [WEKA 与 NVIDIA 在面向 Dynamo 的 KV 缓存存储上展开合作](https://siliconangle.com/2025/11/20/nvidia-weka-kv-cache-solution-ai-inferencing-sc25/)
- [11/13] [Dynamo Office Hours 视频合集](https://www.youtube.com/playlist?list=PL5B692fm6--tgryKu94h2Zb7jTFM3Go4X)
- [10/16] [Baseten 如何借助 NVIDIA Dynamo 实现 2 倍推理加速](https://www.baseten.co/blog/how-baseten-achieved-2x-faster-inference-with-nvidia-dynamo/)
- [12/01] [InfoQ：NVIDIA Dynamo 简化 LLM 推理的 Kubernetes 部署](https://www.infoq.com/news/2025/12/nvidia-dynamo-kubernetes/)

</details>

## 参考资料

- **[支持矩阵](https://docs.nvidia.com/dynamo/resources/support-matrix)** —— 硬件、操作系统、CUDA 与各后端版本
- **[功能矩阵](https://docs.nvidia.com/dynamo/resources/feature-matrix)** —— 各后端兼容性详情
- **[发布产物](https://docs.nvidia.com/dynamo/resources/release-artifacts)** —— 容器、wheels、Helm chart
- **[服务发现](https://docs.nvidia.com/dynamo/kubernetes-deployment/deployment-guide/service-discovery)** —— K8s 原生 vs etcd vs 文件方式
- **[基准测试指南](https://docs.nvidia.com/dynamo/user-guides/benchmarking)** —— 使用 AIPerf 对比部署拓扑

<!-- Reference links for Feature Compatibility Matrix -->
[disagg]: docs/design-docs/disagg-serving.md
[kv-routing]: docs/components/router/README.md
[planner]: docs/components/planner/planner-guide.md
[kvbm]: docs/components/kvbm/README.md
[migration]: docs/fault-tolerance/request-migration.md
[lora]: examples/backends/vllm/deploy/lora/README.md
[tools]: docs/agents/tool-calling.md
