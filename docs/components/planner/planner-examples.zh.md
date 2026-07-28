---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Planner Examples
---

使用基于吞吐的伸缩部署 Planner 的实用示例。下列所有示例都使用带预部署 profiling 的 DGDR 工作流。关于部署概念，参见 [Planner Guide](planner-guide.md)。如需快速概览，参见 [Planner README](README.md)。

## 基础示例

### 使用 AIC 的最小化 DGDR（最快）

使用 Planner 进行部署的最简单方式。使用 AI Configurator 做离线 profiling（耗时 20–30 秒，而不是数小时）：

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: sla-aic
spec:
  model: Qwen/Qwen3-32B
  backend: vllm
  image: "nvcr.io/nvidia/ai-dynamo/dynamo-frontend:1.1.1"
```

部署：
```bash
export NAMESPACE=your-namespace
kubectl apply -f components/src/dynamo/profiler/deploy/profile_sla_aic_dgdr.yaml -n $NAMESPACE
```

### 在线 profiling（真实测量）

标准的在线 profiling 会在真实 GPU 上做测量，结果更准确。耗时 2–4 小时：

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: sla-online
spec:
  model: meta-llama/Llama-3.3-70B-Instruct
  backend: vllm
  image: "nvcr.io/nvidia/ai-dynamo/dynamo-frontend:1.1.1"
```

部署：
```bash
kubectl apply -f components/src/dynamo/profiler/deploy/profile_sla_dgdr.yaml -n $NAMESPACE
```

`components/src/dynamo/profiler/deploy/` 中提供的样例 DGDR：
- **`profile_sla_dgdr.yaml`**：dense 模型的标准在线 profiling
- **`profile_sla_aic_dgdr.yaml`**：使用 AI Configurator 的快速离线 profiling
- **`profile_sla_moe_dgdr.yaml`**：MoE 模型的在线 profiling（SGLang）

> **注意**：从 Dynamo 1.0.0（DGDR API 版本 v1beta1）开始，DGDR 字段使用结构化的 spec 字段（如 `spec.workload`、`spec.sla`、`spec.hardware`），而不是 v1alpha1 中嵌套的 `profilingConfig.config` blob。

## Kubernetes 示例

### MoE 模型（SGLang）

对于像 DeepSeek-R1 这类 Mixture-of-Experts 模型，使用 SGLang 后端（backend）：

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: sla-moe
spec:
  model: deepseek-ai/DeepSeek-R1
  backend: sglang
  image: "nvcr.io/nvidia/ai-dynamo/dynamo-frontend:1.1.1"
```

部署：
```bash
kubectl apply -f components/src/dynamo/profiler/deploy/profile_sla_moe_dgdr.yaml -n $NAMESPACE
```

### 使用已有 DGD 配置（自定义搭建）

通过 ConfigMap 引用一个已有的 DynamoGraphDeployment 配置：

**步骤 1：从你的 DGD 配置创建 ConfigMap：**

```bash
kubectl create configmap deepseek-r1-config \
  --from-file=disagg.yaml=/path/to/your/disagg.yaml \
  --namespace $NAMESPACE \
  --dry-run=client -o yaml | kubectl apply -f -
```

**步骤 2：在你的 DGDR 中引用它：**

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: deepseek-r1
spec:
  model: deepseek-ai/DeepSeek-R1
  backend: sglang
  image: "nvcr.io/nvidia/ai-dynamo/dynamo-frontend:1.1.1"
```

profiler 会把 ConfigMap 中的 DGD 配置作为**基础模板**，并基于你的 SLA 目标对其进行优化。控制器会自动把 `spec.model` 与 `spec.backend` 注入到最终配置中。

### 内联配置（简单场景）

对于不带自定义 DGD 配置的简单场景，可在 v1beta1 DGDR 的 spec 字段中直接给出配置。profiler 会自动生成一份基础 DGD 配置：

```yaml
spec:
  workload:
    isl: 8000
    osl: 200

  sla:
    ttft: 200.0
    itl: 10.0

  hardware:
    gpuSku: h200_sxm

  searchStrategy: rapid
```

### Mocker 部署（测试）

部署一个无需真实 GPU 也能模拟 GPU 时序行为的 mocker 后端。适用于：
- 不占用 GPU 资源做大规模实验
- 测试 planner 行为与基础设施
- 验证部署配置

```yaml
spec:
  model: <model-name>
  backend: trtllm  # 用于 profiling 的真实后端
  features:
    mocker:
      enabled: true  # 部署 mocker 而非真实后端

  image: "nvcr.io/nvidia/ai-dynamo/dynamo-frontend:1.1.1"
```

profiling 仍在真实后端上进行（通过 GPU 或 AIC）。然后 mocker 部署会使用 profiling 数据来模拟真实时序。

### 模型缓存 PVC（0.8.1+）

对于大模型，使用预先填充好的 PVC 而不是从 HuggingFace 下载：

配置细节参见 [SLA-Driven Profiling](../profiler/profiler-guide.md)。

## 高级示例

### 自定义负载预测器

#### 使用 trace 数据做热启动

在真实流量到来之前，用历史请求模式预先加载预测器：

```yaml
# 在 planner 参数中
args:
  - --load-predictor arima
  - --load-predictor-warmup-trace /data/trace.jsonl
  - --load-predictor-log1p
```

trace 文件应为 mooncake 风格的 JSONL 格式，包含 request-count、ISL 与 OSL 样本。

#### Kalman 滤波器调优

对于变化迅速的工作负载，调优 Kalman 滤波器：

```yaml
args:
  - --load-predictor kalman
  - --kalman-q-level 2.0      # 越大对水平变化越敏感
  - --kalman-q-trend 0.5      # 越大趋势变化越快
  - --kalman-r 5.0            # 越小越信任新观测
  - --kalman-min-points 3     # 开始预测前所需的更少点数
  - --load-predictor-log1p    # 通常对请求速率序列有帮助
```

#### Prophet 用于带季节性的工作负载

对于具有日/周模式的工作负载：

```yaml
args:
  - --load-predictor prophet
  - --prophet-window-size 100   # 更大的窗口便于识别季节性
  - --load-predictor-log1p
```

### Virtual Connector

对于非 Kubernetes 环境，使用 VirtualConnector 来传达伸缩决策：

```python
from dynamo._core import DistributedRuntime, VirtualConnectorClient

# Initialize client
client = VirtualConnectorClient(distributed_runtime, namespace)

# Main loop: watch for planner decisions and execute them
while True:
    # Block until the planner makes a new scaling decision
    await client.wait()

    # Read the decision
    decision = await client.get()
    print(f"Scale to: prefill={decision.num_prefill_workers}, "
          f"decode={decision.num_decode_workers}, "
          f"id={decision.decision_id}")

    # Execute scaling in your environment
    scale_prefill_workers(decision.num_prefill_workers)
    scale_decode_workers(decision.num_decode_workers)

    # Report completion
    await client.complete(decision)
```

完整可运行示例参见 `components/planner/test/test_virtual_connector.py`。

### Planner 配置透传

通过 DGDR 透传 planner 专用设置：

```yaml
features:
  planner:
    plannerMinEndpoint: 2
```

### 部署前审阅（autoApply: false）

关闭自动部署以便检查生成的 DGD：

```yaml
spec:
  autoApply: false
```

profiling 完成后：

```bash
# 提取并审阅生成的 DGD
kubectl get dgdr sla-aic -n $NAMESPACE \
  -o jsonpath='{.status.profilingResults.selectedConfig}' > my-dgd.yaml

# 按需审阅与修改
vi my-dgd.yaml

# 手动部署
kubectl apply -f my-dgd.yaml -n $NAMESPACE
```

### 使用 PVC 保存 profiling 产物

将详细的 profiling 产物（图表、日志、原始数据）保存到 PVC：

```yaml
spec:
  workload:
    isl: 3000
    osl: 150

  sla:
    ttft: 200
    itl: 20
```

搭建：
```bash
export NAMESPACE=your-namespace
deploy/utils/setup_benchmarking_resources.sh
```

访问结果：
```bash
kubectl apply -f deploy/utils/manifests/pvc-access-pod.yaml -n $NAMESPACE
kubectl wait --for=condition=Ready pod/pvc-access-pod -n $NAMESPACE --timeout=60s
kubectl cp $NAMESPACE/pvc-access-pod:/data ./profiling-results
kubectl delete pod pvc-access-pod -n $NAMESPACE
```

## 相关文档

- [Planner README](README.md) —— 概览与快速开始
- [Planner Guide](planner-guide.md) —— 部署、配置、集成
- [Planner Design](../../design-docs/planner-design.md) —— 架构深入
- [DGDR Configuration Reference](../profiler/profiler-guide.md#dgdr-configuration-structure)
- [SLA-Driven Profiling](../profiler/profiler-guide.md)
