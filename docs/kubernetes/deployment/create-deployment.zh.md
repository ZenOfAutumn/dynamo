---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Creating Deployments
---

`examples/<backend>/launch` 目录下的脚本（例如 [agg.sh](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/vllm/launch/agg.sh)）演示了如何在本地为模型提供服务。
对应的 YAML 文件（例如 [agg.yaml](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/vllm/deploy/agg.yaml)）展示了如何为推理图创建 Kubernetes 部署。

本指南介绍如何创建你自己的部署文件。

## 第 1 步：选择架构模式

在选择模板之前，先了解不同的架构模式：

### Aggregated Serving（聚合服务）（agg.yaml）

**模式**：prefill 与 decode 在同一个 GPU 的同一个进程中执行。

**建议在以下场景使用**：
- 中小规模模型（小于 70B 参数）
- 开发与测试
- 低到中等流量
- 优先简洁而非极致吞吐

**取舍**：
- 部署与调试更简单
- 运维复杂度更低
- GPU 利用率可能不是最优（prefill 与 decode 互相争抢资源）
- 与解耦部署相比吞吐上限更低

**示例**：[`agg.yaml`](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/vllm/deploy/agg.yaml)

### Aggregated + Router（agg_router.yaml）

**模式**：负载均衡器在多个聚合 worker 实例之间路由。

**建议在以下场景使用**：
- 中等流量需要高可用
- 需要水平扩缩容
- 想获得部分负载均衡，但又不想引入解耦的复杂度

**取舍**：
- 比纯聚合的可扩展性更好
- 通过多副本获得高可用
- 仍存在聚合服务的 GPU 利用率不足问题
- 比纯聚合更复杂，但比解耦简单

**示例**：[`agg_router.yaml`](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/vllm/deploy/agg_router.yaml)

### Disaggregated Serving（解耦服务）（disagg_router.yaml）

**模式**：prefill 与 decode worker 分离，并各自针对性地优化。

**建议在以下场景使用**：
- 类生产部署
- 高吞吐需求
- 大模型（70B+ 参数）
- 需要最大化 GPU 利用率

**取舍**：
- 性能与吞吐最高
- GPU 利用率更好（prefill 与 decode 各自专精）
- prefill 与 decode 可独立扩缩
- 部署与调试更复杂
- 需要理解 prefill / decode 分离

**示例**：[`disagg_router.yaml`](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/vllm/deploy/disagg_router.yaml)

### 快速选择指南

请选择最贴合你使用场景的架构模式作为模板。

例如，使用 `vLLM` 后端时：

- **开发 / 测试**：使用 [`agg.yaml`](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/vllm/deploy/agg.yaml) 作为基础配置。

- **带负载均衡的生产部署**：使用 [`agg_router.yaml`](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/vllm/deploy/agg_router.yaml) 启用可扩展、可负载均衡的推理。

- **高性能 / 解耦部署**：使用 [`disagg_router.yaml`](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/vllm/deploy/disagg_router.yaml) 获得最大吞吐与模块化扩缩。


## 第 2 步：定制模板

你可以将 Frontend 部署在一台机器（例如 CPU 节点），把 worker 部署在另一台机器（GPU 节点）上。
Frontend 充当与框架无关的 HTTP 入口，通常无需做太多修改。

它承担以下角色：
1. OpenAI 兼容的 HTTP 服务器
  * 提供 `/v1/chat/completions` endpoint
  * 处理 HTTP 请求/响应格式化
  * 支持流式响应
  * 校验入站请求

2. 服务发现与路由
  * 通过 etcd 自动发现后端 worker
  * 将请求路由到合适的 Processor / Worker 组件
  * 在多个 worker 之间做负载均衡

3. 请求预处理
  * 初步请求校验
  * 模型名校验
  * 请求格式标准化

接下来选择一个 worker 并定制其配置。例如：

```yaml
VllmWorker:         # vLLM-specific config
  enforce-eager: true
  enable-prefix-caching: true

SglangWorker:       # SGLang-specific config
  router-mode: kv
  disagg-mode: true

TrtllmWorker:       # TensorRT-LLM-specific config
  engine-config: ./engine.yaml
  kv-cache-transfer: ucx
```

下面是基于示例的模板结构：

```yaml
    YourWorker:
      componentType: worker
      replicas: N
      envFromSecret: your-secrets  # e.g., hf-token-secret
      # Health checks for worker initialization
      readinessProbe:
        exec:
          command: ["/bin/sh", "-c", 'grep "Worker.*initialized" /tmp/worker.log']
      resources:
        requests:
          gpu: "1"  # GPU allocation
      extraPodSpec:
        mainContainer:
          image: your-image
          command:
            - /bin/sh
            - -c
          args:
            - python -m dynamo.YOUR_INFERENCE_ENGINE --model YOUR_MODEL --your-flags
```

请参考对应的 sh 文件。每条用于启动组件的 Python 命令都会被放进你的 yaml spec 的 `extraPodSpec: -> mainContainer: -> args:`。

Frontend 通过 `python3 -m dynamo.frontend [--http-port 8000] [--router-mode kv]` 启动。
每个 worker 通过 `python -m dynamo.YOUR_INFERENCE_BACKEND --model YOUR_MODEL --your-flags` 命令启动。


## 第 3 步：关键定制点

### 模型配置

```yaml
   args:
     - "python -m dynamo.YOUR_INFERENCE_BACKEND --model YOUR_MODEL --your-flag"
```

### 资源分配

```yaml
   resources:
     requests:
       cpu: "N"
       memory: "NGi"
       gpu: "N"
```

### 扩缩

```yaml
   replicas: N  # Number of worker instances
```

### 路由模式
```yaml
   args:
     - --router-mode
     - kv  # Enable KV-cache routing
```

### Worker 专精

```yaml
   args:
     - --disaggregation-mode
     - prefill  # For disaggregated prefill workers
```

### 拓扑感知调度

你可以选择性地将相关 pod 打包在同一个拓扑域（例如 rack 或 block）内以降低节点间延迟，对解耦服务工作负载尤其有益。在 deployment 级别、service 级别（或两者）添加 `topologyConstraint`：

```yaml
spec:
  topologyConstraint:
    packDomain: rack
  services:
    VllmWorker:
      # ...
```

这需要 Grove，以及由集群管理员配置的 `ClusterTopology` CR。完整说明、可用的拓扑域、层级规则与示例请参考 **[Topology Aware Scheduling](../topology-aware-scheduling.md)**。

### 镜像拉取密钥配置

#### 自动发现与注入

默认情况下，Dynamo operator 会基于容器镜像 registry 主机名匹配自动发现并注入 image pull secrets。Operator 会扫描同 namespace 下的 Docker config secret，将其 registry 主机名与容器镜像 URL 进行匹配，并自动把合适的 secret 注入 pod 的 `imagePullSecrets`。

**关闭自动发现：**
若想关闭某个组件的自动行为并手动控制 image pull secret：

```yaml
    YourWorker:
      componentType: worker
      annotations:
        nvidia.com/disable-image-pull-secret-discovery: "true"
```

关闭后，你可以像普通 pod spec 那样手动指定 secret：
```yaml
    YourWorker:
      componentType: worker
      annotations:
        nvidia.com/disable-image-pull-secret-discovery: "true"
      extraPodSpec:
        imagePullSecrets:
          - name: my-registry-secret
          - name: another-secret
        mainContainer:
          image: your-image
```

这种自动发现机制让你无需为每个部署手动配置 image pull secret。

## 第 6 步：部署 LoRA 适配器（可选）

基础模型部署运行起来后，你可以使用 `DynamoModel` 自定义资源部署 LoRA 适配器。这让你可以微调与扩展模型，而无需修改基础部署。

要把 LoRA 适配器加入部署，请在 worker 配置中通过 `modelRef` 关联：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-deployment
spec:
  services:
    Worker:
      modelRef:
        name: Qwen/Qwen3-0.6B  # Base model identifier
      componentType: worker
      # ... rest of worker config
```

然后为 LoRA 创建一个 `DynamoModel` 资源：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoModel
metadata:
  name: my-lora
spec:
  modelName: my-custom-lora
  baseModelName: Qwen/Qwen3-0.6B  # Must match modelRef.name above
  modelType: lora
  source:
    uri: s3://my-bucket/loras/my-lora
```

**关于模型与 LoRA 适配器管理的完整细节，请参考：**
📖 **[Managing Models with DynamoModel Guide](./dynamomodel-guide.md)**
