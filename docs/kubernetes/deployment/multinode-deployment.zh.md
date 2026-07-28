---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Multinode Deployments
---

本指南介绍如何跨多节点部署 Dynamo 工作负载。多节点部署可让你将计算密集型 LLM 工作负载扩展到多台物理机，最大化 GPU 利用率，并支撑更大的模型。

## 概览

Dynamo 通过资源规约中的 `multinode` 段支持多节点部署。这能让你：

- 跨多台物理节点分布工作负载
- 将 GPU 资源扩展到超出单机的规模
- 支撑需要大规模张量并行（tensor parallelism）的大模型
- 实现高可用与容错

## 基本要求

- **Kubernetes 集群**：1.24 或更高版本
- **GPU 节点**：多节点，搭载 NVIDIA GPU
- **高速网络**：InfiniBand、RoCE 或高带宽以太网（推荐，以获得最佳性能）


### 高级多节点编排

#### 使用 Grove（默认）

对于复杂的多节点部署，Dynamo 与高级 Kubernetes 编排系统集成：

- **[Grove](https://github.com/NVIDIA/grove)**：面向 AI 工作负载的、网络拓扑感知的 gang 调度（gang scheduling）与自动扩缩
- **[KAI-Scheduler](https://github.com/NVIDIA/KAI-Scheduler)**：为大规模 AI 工作负载优化的 Kubernetes 原生调度器（scheduler）

它们提供拓扑感知放置、gang 调度以及跨节点协同的自动扩缩等增强能力。

**Grove 启用的特性：**
- 声明式组合 AI 工作负载
- 多级水平自动扩缩
- 组件的自定义启动顺序
- 资源感知滚动更新
- [Topology Aware Scheduling](../topology-aware-scheduling.md) — 在机柜、block 或其他拓扑域内紧密放置 Pod 以降低延迟


[KAI-Scheduler](https://github.com/NVIDIA/KAI-Scheduler) 是为大规模 AI 工作负载优化的 Kubernetes 原生调度器。

**KAI-Scheduler 启用的特性：**
- gang 调度
- 网络拓扑感知的 Pod 放置
- 面向 AI 工作负载优化的调度算法
- GPU 资源感知与分配
- 支持复杂调度约束
- 与 Grove 集成以获得增强能力
- 大规模部署的性能优化


##### 前置条件

- 集群已安装 [Grove](https://github.com/NVIDIA/grove/blob/main/docs/installation.md)
- （可选）集群已安装 [KAI-Scheduler](https://github.com/NVIDIA/KAI-Scheduler)，并创建了名为 `dynamo` 的默认队列。如果在 DGD 资源上未指定队列注解，operator 默认使用 `dynamo` 队列。可以通过 `nvidia.com/kai-scheduler-queue` 注解指定自定义队列名，但该队列必须在部署前已存在于集群中。

KAI-Scheduler 是可选的，但推荐使用以获得高级调度能力。

#### 使用 LWS 与 Volcano

LWS 是一种简单的多节点部署机制，可让你跨多节点部署工作负载。

- **LWS**：[LWS 安装](https://github.com/kubernetes-sigs/lws#installation)
- **Volcano**：[Volcano 安装](https://volcano.sh/en/docs/installation/)

Volcano 是为大规模 AI 工作负载优化的 Kubernetes 原生调度器，与 LWS 结合使用以提供 gang 调度支持。


## 核心概念

### 编排器选择算法

Dynamo 按以下逻辑自动为多节点部署选择最合适的编排器：

#### Grove 与 LWS 都可用时：
- **默认选择 Grove**（针对高级 AI 工作负载推荐）
- 若你显式在 DGD 资源上设置 `nvidia.com/enable-grove: "false"` 注解，则**选择 LWS**

#### 仅一种编排器可用时：
- 自动选择已安装的那一种（Grove 或 LWS）

#### 调度器集成：
- **Grove**：在 [KAI-Scheduler](https://github.com/NVIDIA/KAI-Scheduler) 可用时自动集成，提供：
  - 通过 `nvidia.com/kai-scheduler-queue` 注解的高级队列管理
  - 面向 AI 优化的调度策略
  - 资源感知的工作负载放置
- **LWS**：使用 Volcano 调度器进行 gang 调度与资源协调

#### 配置示例：

**默认（Grove + KAI-Scheduler）：**
```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-multinode-deployment
  annotations:
    nvidia.com/kai-scheduler-queue: "dynamo"
spec:
  # ... your deployment spec
```

> **注意：** `nvidia.com/kai-scheduler-queue` 注解默认值为 `"dynamo"`。如指定自定义队列名，需确保该队列在集群中已存在再部署。可使用 `kubectl get queues` 查看可用队列。

**强制使用 LWS：**
```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-multinode-deployment
  annotations:
    nvidia.com/enable-grove: "false"
spec:
  # ... your deployment spec
```


### `multinode` 段

资源规约中的 `multinode` 段定义工作负载应跨多少台物理节点：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-multinode-deployment
spec:
  # ... your deployment spec
  services:
    my-service:
      ...
      multinode:
        nodeCount: 2
      resources:
        limits:
          gpu: "2"            # 2 GPUs per node
```

### GPU 分布

`multinode.nodeCount` 与 `gpu` 之间为乘法关系：

- **`multinode.nodeCount`**：物理节点数
- **`gpu`**：每个节点的 GPU 数
- **总 GPU 数**：`multinode.nodeCount × gpu`

**示例：**
- `multinode.nodeCount: "2"` + `gpu: "4"` = 共 8 个 GPU（2 节点，每节点 4 GPU）
- `multinode.nodeCount: "4"` + `gpu: "8"` = 共 32 个 GPU（4 节点，每节点 8 GPU）

### 张量并行对齐

命令/参数中的张量并行（`tp-size` 或 `--tp`）必须与总 GPU 数一致：

```yaml
# Example: 2 multinode.nodeCount × 4 GPUs = 8 total GPUs
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-multinode-deployment
spec:
  # ... your deployment spec
  services:
    my-service:
      ...
      multinode:
        nodeCount: 2
      resources:
        limits:
          gpu: "4"
      extraPodSpec:
        mainContainer:
          ...
          args:
            # Command args must use tp-size=8
            - "--tp-size"
            - "8"  # Must equal multinode.nodeCount × gpu

```


## 不同后端的 operator 行为

部署多节点工作负载时，Dynamo operator 会自动应用后端特定的配置以启用分布式执行。理解这些自动调整有助于排查问题与优化部署。

### vLLM 后端

对于 vLLM 多节点部署，operator 会根据并行设置自动选择并配置合适的分布式执行模式：

#### 部署模式

operator 基于并行配置自动判定部署模式：

**1. 张量/流水线并行模式（单模型跨节点）**
- **使用条件**：`world_size > GPUs_per_node`，其中 `world_size = tensor_parallel_size × pipeline_parallel_size`
- **使用场景**：通过张量或流水线并行将单个模型实例分布到多节点

operator 在多节点张量/流水线并行部署中使用 Ray。Ray 提供跨节点的自动 placement group 管理与 worker 拉起。

**Leader 节点：**
- **命令**：`ray start --head --port=6379 && <original-vllm-command> --distributed-executor-backend ray`
- **行为**：先启动 Ray head，再运行 vLLM；vLLM 会创建跨所有 Ray worker 的 placement group
- **探针**：所有探针保持启用（liveness、readiness、startup）

**Worker 节点：**
- **命令**：`ray start --address=<leader-hostname>:6379 --block`
- **行为**：加入 Ray 集群并阻塞；leader 上的 vLLM 会向这些 worker 拉起 Ray actor
- **探针**：所有探针（liveness、readiness、startup）会被自动移除

<Note>
vLLM 的 Ray executor 自动创建 placement group 并跨集群拉起 worker。`--nnodes` 标志不与 Ray 同用 — 它仅与 `mp` 后端兼容。
</Note>

**2. 数据并行模式（跨节点的多个模型实例）**
- **使用条件**：`world_size × data_parallel_size > GPUs_per_node`
- **使用场景**：跨节点运行多个独立模型实例，配合数据并行（如带专家并行的 MoE 模型）

**所有节点（leader 与 worker）：**
- **注入的标志**：
  - `--data-parallel-address <leader-hostname>` - 协调服务器地址
  - `--data-parallel-size-local <value>` - 每个节点的数据并行 worker 数
  - `--data-parallel-rpc-port 13445` - 数据并行协调用 RPC 端口
  - `--data-parallel-start-rank <value>` - 本节点的起始 rank（自动计算）
- **探针**：worker 探针被移除；leader 探针保持启用

**注意**：无论命令结构如何（直接 Python 命令或 shell 包装），operator 都能智能地将这些标志注入到命令中。

#### 为什么多节点 TP/PP 选 Ray？

vLLM 支持两种分布式 executor 后端：`ray` 和 `mp`。在多节点部署中：

- **Ray executor**：vLLM 创建 placement group 并跨集群拉起 Ray actor。Worker 不直接运行 vLLM — 由 leader 的 vLLM 进程统一管理。
- **mp executor**：每个节点必须自行运行带 `--nnodes`、`--node-rank`、`--master-addr`、`--master-port` 的 vLLM 进程。这种方式编排更复杂。

Dynamo operator 选择 Ray 的原因：
1. 与 vLLM 官方多节点文档一致（参见 `multi-node-serving.sh`）
2. 编排更简单 — 只有 leader 运行 vLLM，worker 仅需 Ray agent
3. vLLM 自动处理 placement group 创建与 worker 管理

#### 编译缓存（Compilation Cache）支持
当 volume mount 配置 `useAsCompilationCache: true` 时，operator 会自动设置：
- **`VLLM_CACHE_ROOT`**：指向缓存挂载点的环境变量

### SGLang 后端

对于 SGLang 多节点部署，operator 会注入分布式训练参数：

#### Leader 节点
- **分布式标志**：注入 `--dist-init-addr <leader-hostname>:29500 --nnodes <count> --node-rank 0`
- **探针**：所有探针保持启用

#### Worker 节点
- **分布式标志**：注入 `--dist-init-addr <leader-hostname>:29500 --nnodes <count> --node-rank <dynamic-rank>`
  - `node-rank` 由 Pod 的 stateful 身份自动确定
- **探针**：所有探针（liveness、readiness、startup）会被自动移除

**注意：** 无论命令结构如何（直接 Python 命令或 shell 包装），operator 都能智能注入这些标志。

### TensorRT-LLM 后端

对于 TensorRT-LLM 多节点部署，operator 会配置基于 MPI 的通信：

#### Leader 节点
- **SSH 配置**：从 Kubernetes secret 自动设置 SSH keys 与配置
- **MPI 命令**：将你的命令包装在 `mpirun` 命令中，包括：
  - 包含所有 worker 节点的 host 列表
  - 用于 2222 端口免密登录的 SSH 配置
  - 向所有节点传播环境变量
  - 激活 Dynamo 虚拟环境
- **探针**：所有探针保持启用

#### Worker 节点
- **SSH Daemon**：将你的命令替换为 SSH daemon 启动与执行
  - 在用户可写目录中生成 host keys（非特权）
  - 配置 SSH daemon 监听 2222 端口
  - 设置允许 leader 访问的 authorized keys
- **探针**：
  - **Liveness 与 Startup**：移除（worker 运行 SSH daemon，而非主应用）
  - **Readiness**：替换为对 SSH 端口 2222 的 TCP socket 检查
    - 初始延迟：20 秒
    - 周期：20 秒
    - 超时：5 秒
    - 失败阈值：10

#### 额外配置
- **环境变量**：所有节点添加 `OMPI_MCA_orte_keep_fqdn_hostnames=1`
- **SSH Volume**：自动挂载 SSH 密钥对 secret（通常命名为 `mpirun-ssh-key-<deployment-name>`）
- **自动 SSH 密钥生成**：当检测到多节点 `DynamoGraphDeployment` 时，operator 自动生成 SSH 密钥对 secret，无需手动创建。

### 编译缓存配置

operator 支持各后端用的编译缓存（compilation cache）卷：

| 后端 | 支持级别 | 环境变量 | 默认挂载点 |
|---------|--------------|----------------------|---------------------|
| vLLM | 完全支持 | `VLLM_CACHE_ROOT` | 用户指定 |
| SGLang | 部分支持 | _无（待上游支持）_ | 用户指定 |
| TensorRT-LLM | 部分支持 | _无（待上游支持）_ | 用户指定 |

要启用编译缓存，请在组件规约中添加 `useAsCompilationCache: true` 的 volume mount。对于 vLLM，operator 将自动配置必要的环境变量；对于其他后端，会创建挂载，但在上游支持就绪前可能需要额外的环境配置。

## 后续步骤

如需更多支持与示例，请参见以下可用的多节点配置：

- **SGLang**：[examples/backends/sglang/deploy/](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/sglang/deploy/README.md)
- **TensorRT-LLM**：[examples/backends/trtllm/deploy/](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/trtllm/deploy/README.md)
- **vLLM**：[examples/backends/vllm/deploy/](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/vllm/deploy/README.md)

这些示例演示了 `multinode` 段、对应 `gpu` 限制以及正确 `tp-size` 的恰当用法。
