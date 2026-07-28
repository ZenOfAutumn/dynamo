---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Global Planner 部署指南
---

本指南介绍如何部署 `GlobalPlanner` 以及何时使用它。`GlobalPlanner` 是集中式扩缩容执行层，适用于多个 DGD 应通过单一组件委托扩缩容的部署 —— 无论这些 DGD 各自暴露独立端点，还是位于同一共享端点之后。

> **新接触 Planner？** 我们建议先用基于吞吐量或基于负载的扩缩容做一个单 DGD 部署，再考虑使用 GlobalPlanner。参见 [Planner 概览](README.md) 与 [Planner Guide](planner-guide.md) 上手。

## 为什么需要 Global Planner？

如果没有 `GlobalPlanner`，每个 DGD 的本地 planner 仅直接对自身部署进行扩缩容。这对孤立部署而言没问题，但当你希望在同一处做以下事情时就会变得别扭：

- 跨多个 DGD 应用集中式扩缩容策略
- 强制施加共享约束，例如鉴权或总 GPU 预算
- 为单端点、多池部署协调扩缩容

`GlobalPlanner` 通过成为多个本地 planner 的公共扩缩容执行端点解决这些问题。

## 术语

- **Planner**：`dynamo.planner` 组件，用于计算维持延迟 SLA 所需的副本数。详见 [Planner 概览](README.md)。
- **Local Planner**：在单个 DGD 中运行的池本地 Planner 实例。
- **Global Planner**：集中式执行与策略层，接收来自本地 planner 的扩缩容请求。
- **单端点多池部署**：由多个 DGD 服务同一模型的单一模型端点。该模式同时使用 `GlobalRouter` 与 `GlobalPlanner`。

## 部署模式

按以下两种模式之一使用 `GlobalPlanner`：

| 模式 | 适用场景 | 是否需要 `GlobalRouter` | 公开端点形态 |
|---------|----------|----------------------|-----------------------|
| 多个模型端点或独立 DGD | 多个独立 DGD 共享集中式扩缩容策略，如鉴权或总 GPU 预算 | 否 | 每个 DGD 一个端点，或按各 DGD 自身暴露方式 |
| 一个模型端点，多个 DGD | 同一模型应通过单一公开端点访问，但不同请求类落到不同 DGD | 是 | 一个共享端点 |

## 模式 1：多个模型端点或独立 DGD

当你有多个 DGD（通常服务不同模型），并希望它们共享集中式扩缩容策略而不合并成单一端点时，使用此模式。

典型示例：

- DGD A：`qwen-0.6b` 解耦部署，带自己的本地 planner
- DGD B：`qwen-32b` 解耦部署，带自己的本地 planner
- 一个共享 `GlobalPlanner`，所有本地 planner 都委托给它

此模式中：

- 每个 DGD 保留自己的本地 planner
- 每个本地 planner 配置 `environment: "global-planner"`
- 所有这些 planner 指向同一个 `global_planner_namespace`
- 每个 DGD 按需保留各自的端点或前端
- **不**需要 `GlobalRouter`

当目标是跨多个部署或模型实现集中式扩缩容控制时使用此模式。

## 模式 2：单模型端点，多 DGD

当下列全部为真时使用此模式：

- 你希望某个模型暴露单一公开端点。
- 你希望为不同请求类（如短 ISL vs 长 ISL，或不同延迟目标）准备不同的私有池。
- 你希望每个池独立自动扩缩。
- 你希望路由与扩缩容执行集中化，而不是把多个端点暴露给客户端。

典型示例：

- 短输入请求在更小的预填充（prefill）池上更便宜
- 长输入请求需要更大的 prefill 池
- 解码（decode）容量应独立于 prefill 容量进行扩缩

如果你只需要一个池服务一个模型，使用单个 Local Planner 与 DGD/DGDR 即可。

## 你需要部署什么

在当前实现中，单端点模式由多种资源组成：

| 资源 | 用途 | 典型内容 |
|----------|---------|------------------|
| Control DGD | 公开入口与集中式控制面 | `Frontend`、`GlobalRouter`、`GlobalPlanner` |
| Prefill 池 DGD（可多个） | 私有 prefill 容量池 | `LocalRouter`、prefill worker、`Planner` |
| Decode 池 DGD（可多个） | 私有 decode 容量池 | `LocalRouter`、decode worker、`Planner` |
| 可选 DGDR | 一次生成或验证一个优化的池形态 | 模型、工作负载、SLA、硬件输入 |

> **当前工作流**
>
> 当前**单个 DGDR 不会**生成完整的单端点多池拓扑。改为对每个目标池运行一个 DGDR 或 profiling 作业，然后手动组合最终的 control DGD 与池 DGD。

## 架构

```text
Client
  |
  v
Frontend (single public endpoint)
  |
  v
GlobalRouter
  |
  +--> Prefill pool 0 Dynamo namespace --> LocalRouter --> Prefill workers --> Pool Planner
  +--> Prefill pool 1 Dynamo namespace --> LocalRouter --> Prefill workers --> Pool Planner
  |
  +--> Decode pool 0 Dynamo namespace  --> LocalRouter --> Decode workers  --> Pool Planner
  +--> Decode pool 1 Dynamo namespace  --> LocalRouter --> Decode workers  --> Pool Planner

Pool Planners
  |
  v
GlobalPlanner
  |
  v
Kubernetes scaling updates on the target DGDs
```

`Frontend` 暴露一个模型端点。`GlobalRouter` 为每个请求选择最佳池。每个池本地的 `Planner` 决定其池所需容量。`GlobalPlanner` 接收这些扩缩容请求，并集中地应用 Kubernetes 副本变更。

## 前置条件

- 已安装 Dynamo Kubernetes Platform。详见 [Kubernetes Quickstart](../../kubernetes/README.md)。
- 已部署 Prometheus 并采集 router 指标。global planner 示例假定集群 Prometheus 可用。
- 已为所选框架（`vllm`、`sglang` 或 `trtllm`）准备后端镜像。
- 用于模型访问的 secret，例如 Hugging Face token secret。
- 若 worker 应共享模型缓存 PVC，需要为模型权重制定存储策略。

对于基于吞吐量的扩缩容，还需要每个池的 profiling 数据。详见 [Profiler Guide](../profiler/profiler-guide.md)。

## 你需要预先确定的输入

在编写清单（manifest）之前，确定以下内容：

| 输入 | 重要性 | 示例 |
|-------|----------------|---------|
| 模型名 | 同一层级中的所有池服务相同模型 | `meta-llama/Llama-3.3-70B-Instruct` |
| 后端 | worker 参数与 profiling 流程依赖它 | `vllm` |
| 池清单 | 专门化的 prefill 和 decode 池数量 | 2 个 prefill 池，1 个 decode 池 |
| 工作负载类 | 决定生成多少个池 profile | 短 ISL、长 ISL、长上下文 decode |
| SLA 目标 | 指导 profiling 与路由决策 | `ttft: 200 ms`、`itl: 20 ms` |
| Worker 形态 | tensor 并行、每 worker GPU 数与显存占用 | TP1 prefill vs. TP2 prefill |
| 路由策略 | 在运行时把请求映射到池 | 低 ISL 请求 -> pool 0 |
| 可选全局预算 | 受管池总 GPU 上限 | `--max-total-gpus 16` |

## 步骤 1：独立对每个目标池进行 Profiling

先决定每个池的专门化方向。常见示例：

- Prefill 池 0：成本更低，专攻短 prompt。
- Prefill 池 1：更大的池，处理长 prompt。
- Decode 池 0：标准 decode 池，承担多数请求。

对每个目标池，运行单独的 DGDR 或 profiling 作业，使用代表该池请求类的工作负载与 SLA。

DGDR 骨架示例：

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: llama-prefill-short
spec:
  model: meta-llama/Llama-3.3-70B-Instruct
  backend: vllm
  image: nvcr.io/nvidia/ai-dynamo/dynamo-frontend:<tag>
  workload:
    isl: 2048
    osl: 256
  sla:
    ttft: 200.0
    itl: 20.0
  searchStrategy: rapid
  autoApply: false
```

每个计划的池重复一次，按请求类调整工作负载与 SLA 输入。

每次 profiling 结果中要保留：

- worker 形态（`tensor-parallel-size`、每 worker GPU 数、显存/缓存设置）。
- planner 的 profile 数据目录或生成的 ConfigMap。
- planner 设置，如 `prefill_engine_num_gpu` 或 `decode_engine_num_gpu`。
- 各池间不同的后端特定 flag。

DGDR 详情参见 [Planner Examples](planner-examples.md) 与 [Profiler Guide](../profiler/profiler-guide.md)。

## 步骤 2：创建 Control DGD

部署一个 control DGD，包含：

- `Frontend`：单一公开模型端点。
- `GlobalRouter`：决定每个请求由哪个池接收。
- `GlobalPlanner`：从池 planner 接收扩缩容请求并应用副本变更。

vLLM 示例拓扑见 [examples/global_planner/global-planner-vllm-test.yaml](https://github.com/ai-dynamo/dynamo/blob/main/examples/global_planner/global-planner-vllm-test.yaml)。

`GlobalPlanner` 段落很简洁：

```yaml
GlobalPlanner:
  componentType: default
  replicas: 1
  extraPodSpec:
    mainContainer:
      image: ${DYNAMO_IMAGE}
      command:
        - python3
        - -m
        - dynamo.global_planner
      args:
        - --managed-namespaces
        - ${K8S_NAMESPACE}-gp-prefill-0
        - ${K8S_NAMESPACE}-gp-prefill-1
        - ${K8S_NAMESPACE}-gp-decode-0
```

传给 `--managed-namespaces` 的是池 planner 的 **Dynamo 命名空间**（`caller_namespace`），而非原始 Kubernetes 命名空间。在很多示例中两者使用相同字符串前缀，但它们在逻辑上是不同的标识。

**管理模式**：当设置 `--managed-namespaces` 时（显式模式），仅列出的 Dynamo 命名空间被授权发送扩缩容请求，且仅对应 DGD 计入 GPU 预算。DGD 名按 operator 约定 `DYN_NAMESPACE = {k8s_namespace}-{dgd_name}` 由 Dynamo 命名空间派生。当省略时（隐式模式），任何调用方都被接受，且 Kubernetes 命名空间内所有 DGD 都计入 GPU 预算。

如果希望中央执行器拒绝超出总 GPU 预算的扩缩容请求，加上 `--max-total-gpus`。详见 [examples/global_planner/global-planner-gpu-budget.yaml](https://github.com/ai-dynamo/dynamo/blob/main/examples/global_planner/global-planner-gpu-budget.yaml)。

## 步骤 3：每个池一个 DGD

每个私有池得到自己的 DGD。池 DGD 通常包含：

- `LocalRouter`
- 一种 worker 类型（`prefill` 或 `decode`）
- 一个 `Planner`

每个池内的 planner 必须配置为 `global-planner` 模式以将扩缩容委托给控制栈：

```json
{
  "environment": "global-planner",
  "global_planner_namespace": "${K8S_NAMESPACE}-gp-ctrl",
  "backend": "vllm",
  "mode": "prefill",
  "enable_load_scaling": false,
  "enable_throughput_scaling": true,
  "throughput_metrics_source": "router",
  "ttft": 2000,
  "prefill_engine_num_gpu": 2,
  "model_name": "${MODEL_NAME}",
  "profile_results_dir": "/workspace/components/src/dynamo/planner/tests/data/profiling_results/H200_TP1P_TP1D"
}
```

`global_planner_namespace` 必须指向控制栈的 **Dynamo 命名空间**。在参考清单中即传给控制 `Frontend` 与 `GlobalRouter` 的命名空间字符串。

使用：

- `mode: "prefill"` 用于 prefill-only 池
- `mode: "decode"` 用于 decode-only 池

每个池的 worker 与 planner 设置来自步骤 1 中你为该池所做的 profiling 结果。

在参考 vLLM 示例中：

- `gp-prefill-0` 使用 1 GPU 的 TP1 prefill worker
- `gp-prefill-1` 使用 2 GPU 的 TP2 prefill worker
- `gp-decode-0` 使用 1 GPU 的 TP1 decode worker

详见 [global-planner-vllm-test.yaml](https://github.com/ai-dynamo/dynamo/blob/main/examples/global_planner/global-planner-vllm-test.yaml)。

## 步骤 4：配置 GlobalRouter 选择池

`GlobalRouter` 读取一份 JSON 配置，列出池命名空间以及每种请求类型的路由网格。

示例：

```json
{
  "num_prefill_pools": 2,
  "num_decode_pools": 1,
  "prefill_pool_dynamo_namespaces": [
    "${K8S_NAMESPACE}-gp-prefill-0",
    "${K8S_NAMESPACE}-gp-prefill-1"
  ],
  "decode_pool_dynamo_namespaces": [
    "${K8S_NAMESPACE}-gp-decode-0"
  ],
  "prefill_pool_selection_strategy": {
    "ttft_min": 10,
    "ttft_max": 3000,
    "ttft_resolution": 2,
    "isl_min": 0,
    "isl_max": 32000,
    "isl_resolution": 2,
    "prefill_pool_mapping": [[0, 1], [0, 1]]
  },
  "decode_pool_selection_strategy": {
    "itl_min": 10,
    "itl_max": 500,
    "itl_resolution": 2,
    "context_length_min": 0,
    "context_length_max": 32000,
    "context_length_resolution": 2,
    "decode_pool_mapping": [[0, 0], [0, 0]]
  }
}
```

`prefill_pool_dynamo_namespaces` 与 `decode_pool_dynamo_namespaces` 中的条目是池本地路由所注册的 **Dynamo 命名空间**。

重要的运行时行为：

- Prefill 池选择使用 **ISL + TTFT 目标**
- Decode 池选择使用 **上下文长度 + ITL 目标**
- OSL 在**设计与剖析池**时有用，但当前 `GlobalRouter` **不直接将其作为路由 key**

客户端可通过 `extra_args` 传递请求目标：

```json
{
  "extra_args": {
    "ttft_target": 200,
    "itl_target": 20
  }
}
```

详情见 [Global Router README](https://github.com/ai-dynamo/dynamo/blob/main/components/src/dynamo/global_router/README.md)。

## 步骤 5：按顺序部署

对于全新集群，常见顺序为：

1. 安装 Dynamo 平台与 Prometheus。
2. 创建 worker 所需的 secret 与 PVC。
3. 创建 `GlobalRouter` ConfigMap。
4. 应用 control DGD。
5. 应用各池 DGD。
6. 等待所有 DGD 进入 ready 状态。
7. 暴露或端口转发控制 `Frontend`。

示例：

```bash
export K8S_NAMESPACE=my-llama
export MODEL_NAME=meta-llama/Llama-3.3-70B-Instruct
export DYNAMO_IMAGE=<dynamo-image>
export DYNAMO_VLLM_IMAGE=<vllm-image>
export STORAGE_CLASS_NAME=<rwx-storage-class>

kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=${HF_TOKEN} \
  -n ${K8S_NAMESPACE}

envsubst < examples/global_planner/global-planner-vllm-test.yaml | \
  kubectl apply -n ${K8S_NAMESPACE} -f -
```

唯一面向用户的端点是控制 DGD 中的 `Frontend`，而不是池 DGD。

## 步骤 6：验证全栈

由外到内验证部署：

- 确认控制 `Frontend` 健康并提供模型端点。
- 确认 `GlobalRouter` 日志显示请求被分配到预期的池命名空间。
- 确认池本地 planner 在产生扩缩容请求。
- 确认 `GlobalPlanner` 日志显示扩缩容操作被接受。
- 确认目标 DGD 的副本数按预期变化。

如使用 Prometheus 与 Grafana，还可观察：

- TTFT 与 ITL 随时间的变化
- 每个池的 worker 数
- 每个池的请求构成
- 总 GPU 使用量

## 新部署的推荐工作流

对多数团队而言，构建此部署最简单的方式是：

1. 根据预期流量模式设计你的池类。
2. 每个池类运行一个 DGDR 以生成或验证池配置。
3. 把选定的 worker 形态与 planner 设置复制到最终的池 DGD。
4. 构建一个包含 `Frontend`、`GlobalRouter` 与 `GlobalPlanner` 的 control DGD。
5. 让所有客户端流量通过控制 `Frontend`。

这能在保持 profiling 与池选择简单的同时，仍为模型提供一个公开端点。

## 当前限制

- 单端点 `GlobalPlanner` 部署目前由人工组装。单个 DGDR 不会输出完整的 control DGD + 池 DGD 拓扑。
- `GlobalRouter` 按 ISL/TTFT 与 上下文长度/ITL 网格路由，不直接按 OSL 路由。
- 在单端点模式中，所有池预期服务同一模型。

## 参见

- [Planner README](README.md) —— Planner 概览与快速上手
- [Planner Guide](planner-guide.md) —— Planner 配置参考
- [Planner Examples](planner-examples.md) —— 用于生成各池配置的 DGDR 示例
- [Profiler Guide](../profiler/profiler-guide.md) —— 部署前 profiling 工作流
- [Global Planner README](https://github.com/ai-dynamo/dynamo/blob/main/components/src/dynamo/global_planner/README.md) —— 集中式扩缩容执行
- [Global Router README](https://github.com/ai-dynamo/dynamo/blob/main/components/src/dynamo/global_router/README.md) —— 跨池请求路由
- [vLLM global planner 示例](https://github.com/ai-dynamo/dynamo/blob/main/examples/global_planner/global-planner-vllm-test.yaml) —— 端到端参考清单
