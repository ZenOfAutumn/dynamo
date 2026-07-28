---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Profiler Examples
---

使用 DGDR 进行性能剖析的完整示例。

## DGDR 示例

### 稠密模型：Rapid

快速剖析（约 30 秒）：

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: qwen-0-6b
spec:
  model: "Qwen/Qwen3-0.6B"
  image: "nvcr.io/nvidia/ai-dynamo/dynamo-frontend:1.1.1"
```

### 稠密模型：Thorough

使用真实 GPU 测量进行剖析：

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: vllm-dense-online
spec:
  model: "Qwen/Qwen3-0.6B"
  backend: vllm
  image: "nvcr.io/nvidia/ai-dynamo/dynamo-frontend:1.1.1"
  searchStrategy: thorough
```

### MoE 模型

使用 SGLang 的多节点 MoE 剖析：

> [!IMPORTANT]
> `modelCache.pvcName` 引用的 PVC 必须已经存在于同一命名空间中，并在指定的
> `pvcModelPath` 下包含模型权重。DGDR 控制器不会创建或填充该 PVC——它仅将
> 该 PVC 挂载到剖析 Job 与所部署的 worker 中。

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: sglang-moe
spec:
  model: "deepseek-ai/DeepSeek-R1"
  backend: sglang
  image: "nvcr.io/nvidia/ai-dynamo/dynamo-frontend:1.1.1"

  hardware:
    numGpusPerNode: 8

  modelCache:
    pvcName: "model-cache"
    pvcModelPath: "deepseek-r1"      # path within the PVC
```

### 私有模型

对于受门控（gated）或私有的 HuggingFace 模型，可通过环境变量将 token 注入到剖析
Job。先创建 secret：

```bash
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="${HF_TOKEN}" \
  -n ${NAMESPACE}
```

然后在 DGDR 中引用它：

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: llama-private
spec:
  model: "meta-llama/Llama-3.1-8B-Instruct"
  image: "nvcr.io/nvidia/ai-dynamo/dynamo-frontend:1.1.1"

  overrides:
    profilingJob:
      template:
        spec:
          containers: []    # required placeholder; leave empty to inherit defaults
          initContainers:
            - name: profiler
              env:
                - name: HF_TOKEN
                  valueFrom:
                    secretKeyRef:
                      name: hf-token-secret
                      key: HF_TOKEN
```

### 自定义 SLA 目标

通过指定延迟目标和工作负载特征来控制 profiler 优化部署的方式。

**显式 TTFT + ITL 目标**（默认模式）：

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: low-latency-dense
spec:
  model: "Qwen/Qwen3-0.6B"
  image: "nvcr.io/nvidia/ai-dynamo/dynamo-frontend:1.1.1"

  sla:
    ttft: 500      # Time To First Token target in milliseconds
    itl: 20        # Inter-Token Latency target in milliseconds

  workload:
    isl: 2000      # expected input sequence length (tokens)
    osl: 500       # expected output sequence length (tokens)
```

**端到端延迟目标**（替代 ttft+itl）：

```yaml
spec:
  ...
  sla:
    e2eLatency: 10000    # total request latency budget in milliseconds
```

### 覆盖（Overrides）

使用 `overrides` 来自定义剖析 Job 的 pod spec——例如为 GPU 节点 taint 添加
toleration，或注入环境变量。

**GPU 节点 toleration**（GKE 与共享集群中常见）：

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: dense-with-tolerations
spec:
  model: "Qwen/Qwen3-0.6B"
  image: "nvcr.io/nvidia/ai-dynamo/dynamo-frontend:1.1.1"

  overrides:
    profilingJob:
      template:
        spec:
          containers: []    # required placeholder; leave empty to inherit defaults
          tolerations:
            - key: nvidia.com/gpu
              operator: Exists
              effect: NoSchedule
```

**覆盖生成的 DynamoGraphDeployment**（例如使用自定义 worker 镜像）：

```yaml
spec:
  ...
  overrides:
    dgd:
      apiVersion: nvidia.com/v1alpha1
      kind: DynamoGraphDeployment
      spec:
        services:
          VllmWorker:
            extraEnvs:
              - name: CUSTOM_ENV
                value: "my-value"
```

## SGLang 运行时剖析

通过 HTTP 端点对 SGLang worker 进行运行时剖析：

```bash
# Start profiling
curl -X POST http://localhost:9090/engine/start_profile \
  -H "Content-Type: application/json" \
  -d '{"output_dir": "/tmp/profiler_output"}'

# Run inference requests to generate profiling data...

# Stop profiling
curl -X POST http://localhost:9090/engine/stop_profile
```

测试脚本位于 `examples/backends/sglang/test_sglang_profile.py`：

```bash
python examples/backends/sglang/test_sglang_profile.py
```

可使用 Chrome 的 `chrome://tracing`、[Perfetto UI](https://ui.perfetto.dev/) 或 TensorBoard 查看 trace。
