---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Profiler
---

Dynamo Profiler 是一个自动化的性能分析工具，用于测量模型的推理特征以优化部署配置。它确定 prefill 和 decode 阶段的最优张量并行（TP）设置，生成性能插值数据，并通过 Planner 实现 SLA 驱动的自动扩缩容。

## 功能矩阵

| 特性 | SGLang | TensorRT-LLM | vLLM |
|---------|--------|--------------|------|
| Dense 模型剖析 | ✅ | ✅ | ✅ |
| MoE 模型剖析 | ✅ | 🚧 | 🚧 |
| AI Configurator（离线） | ✅ | ✅ | ✅ |
| 在线剖析（AIPerf） | ✅ | ✅ | ✅ |
| 交互式 WebUI | ✅ | ✅ | ✅ |
| 运行时剖析端点 | ✅ | ❌ | ❌ |

## 快速开始

### 前置条件

- 已安装 Dynamo 平台（参见[安装指南](../../kubernetes/installation-guide.md)）
- 带 GPU 节点的 Kubernetes 集群（用于基于 DGDR 的剖析）
- 已安装 kube-prometheus-stack（SLA planner 必需）

### 使用 DynamoGraphDeploymentRequest（推荐）

剖析模型的推荐方式是通过 DGDR，它会自动化整个剖析与部署流程。

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: my-model-profiling
spec:
  model: "Qwen/Qwen3-0.6B"
  backend: vllm
  image: "nvcr.io/nvidia/ai-dynamo/dynamo-frontend:1.1.1"

  workload:
    isl: 3000      # Average input sequence length
    osl: 150       # Average output sequence length

  sla:
    ttft: 200.0    # Target Time To First Token (ms)
    itl: 20.0      # Target Inter-Token Latency (ms)

  autoApply: true
```

```bash
kubectl apply -f my-profiling-dgdr.yaml -n $NAMESPACE
```

### 使用 AI Configurator（快速离线剖析）

AI Configurator 支持快速离线剖析（约 30 秒），并支持所有后端（vLLM、SGLang、TensorRT-LLM）。由于 `searchStrategy: rapid` 是默认值，除非你显式设置 `searchStrategy: thorough`，否则会自动使用 AIC。

## 配置

| 参数 | 默认值 | 描述 |
|-----------|---------|-------------|
| `workload.isl` | 4000 | 平均输入序列长度（token） |
| `workload.osl` | 1000 | 平均输出序列长度（token） |
| `sla.ttft` | 2000 | 目标首 token 时间（毫秒） |
| `sla.itl` | 30 | 目标 token 间延迟（毫秒） |
| `hardware.numGpusPerNode` | 自动 | 每节点 GPU 数 |
| `hardware.gpuSku` | 自动 | GPU SKU 标识 |

## 剖析方法

| 方法 | 时长 | 准确性 | 是否需要 GPU | 后端 |
|--------|----------|----------|--------------|----------|
| 在线（AIPerf） | 2-4 小时 | 最高 | 是 | 全部 |
| 离线（AI Configurator） | 20-30 秒 | 估算 | 否 | 全部 |

## 输出

Profiler 生成：

1. **最优配置**：prefill 与 decode 引擎的推荐 TP 大小
2. **性能数据**：用于 SLA Planner 的插值模型
3. **生成的 DGD**：带优化设置的完整部署 manifest

示例建议：
```text
Suggested prefill TP:4 (TTFT 48.37 ms, throughput 15505.23 tokens/s/GPU)
Suggested decode TP:4 (ITL 4.83 ms, throughput 51.22 tokens/s/GPU)
```

## 后续步骤

| 文档 | 描述 |
|----------|-------------|
| [Profiler 指南](profiler-guide.md) | 配置、方法与故障排查 |
| [Profiler 示例](profiler-examples.md) | 完整的 DGDR YAML、WebUI、脚本示例 |
| [SLA Planner 指南](../planner/planner-guide.md) | 端到端部署工作流程 |
| [SLA Planner 架构](../planner/planner-guide.md) | Planner 如何使用剖析数据 |
