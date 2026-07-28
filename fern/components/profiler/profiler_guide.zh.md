---
# SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
---

# Profiler 指南

本指南覆盖 Dynamo Profiler 的部署、配置、集成与故障排查。

## 什么是 DynamoGraphDeploymentRequest（DGDR）？

**DynamoGraphDeploymentRequest（DGDR）** 是一种 Kubernetes 自定义资源，作为用户请求带特定性能与资源约束的模型部署的主要接口。你需要指定：

- 想部署 **什么** 模型（`model`）
- 它应该 **如何** 表现（SLA 目标：`sla.ttft`、`sla.itl`）
- 它应该 **在哪里** 运行（通过 `hardware` 给出可选的 GPU 偏好）
- 使用 **哪个** 后端（backend）（`backend`：auto、vllm、sglang 或 trtllm）
- 使用 **哪个** 镜像（`image`）

Dynamo Operator 会监听 DGDR，并自动：
1. 发现集群（cluster）中可用的 GPU 资源
2. 运行 profiling（在线或离线）找出最佳配置
3. 生成优化后的 DynamoGraphDeployment（DGD）配置
4. 把 DGD 部署到集群

**与 DGD 的关系：**
- **DGDR**：高层“意图” —— 你想部署什么
- **DGD**：底层“实现” —— 它如何被部署

## 支持矩阵

| 后端 | Dense 模型 | MoE 模型 |
|---------|-------------|------------|
| vLLM | ✅ | 🚧 |
| SGLang | ✅ | ✅ |
| TensorRT-LLM | ✅ | 🚧 |

profiler 在 prefill 与 decode 上扫描以下并行映射：

| 模型架构 | Prefill 并行映射 | Decode 并行映射 |
|---------|-------------|------------|
| MLA+MoE（DeepseekV3ForCausalLM, DeepseekV32ForCausalLM） | TEP, DEP | TEP, DEP |
| GQA+MoE（Qwen3MoeForCausalLM） | TP, TEP, DEP | TP, TEP, DEP |
| 其他模型 | TP | TP |

> [!NOTE]
> 具体“模型 × 并行映射”的支持情况依赖后端。profiler 不保证推荐的 P/D 引擎配置在后端上完全支持且无 bug。

## 部署

### Kubernetes 部署（DGDR）

推荐通过 DGDR 部署。`benchmarks/profiler/deploy/` 中提供了示例配置：

| 示例 | 说明 |
|--------|-------------|
| `profile_sla_dgdr.yaml` | 使用 AIPerf 的标准在线 profiling |
| `profile_sla_aic_dgdr.yaml` | 使用 AI Configurator 的快速离线 profiling |
| `profile_sla_moe_dgdr.yaml` | MoE 模型 profiling（SGLang） |

#### 容器镜像

每个 DGDR 都需要一个用于 profiling 与部署的容器镜像：

- **`image`**（可选）：用于 profiling 任务的容器镜像。必须包含 profiler 代码与依赖。

```yaml
spec:
  image: "nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1"
```

#### 快速开始：使用 DGDR 部署

**步骤 1：创建 DGDR**

使用样例配置或自行创建：

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: my-model-profiling
spec:
  model: "Qwen/Qwen3-0.6B"
  backend: vllm
  image: "nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1"

  workload:
    isl: 3000
    osl: 150

  sla:
    ttft: 200.0
    itl: 20.0

  autoApply: true
```

**步骤 2：应用 DGDR**

```bash
export NAMESPACE=your-namespace
kubectl apply -f my-profiling-dgdr.yaml -n $NAMESPACE
```

**步骤 3：监控进度**

```bash
# View status
kubectl get dgdr -n $NAMESPACE

# Detailed status
kubectl describe dgdr my-model-profiling -n $NAMESPACE

# Watch profiling job logs
kubectl logs -f job/profile-my-model-profiling -n $NAMESPACE
```

**DGDR 状态阶段：**
- `Pending`：初始状态，准备 profiling
- `Profiling`：正在运行 profiling 任务（AIC 约 20–30 秒，在线约 2–4 小时）
- `Ready`：profiling 完成，生成的 DGD spec 已在 status 中
- `Deploying`：正在生成并应用 DGD 配置
- `Deployed`：DGD 已成功部署并运行
- `Failed`：发生错误（详情见 events）

**步骤 4：访问你的部署**

```bash
# Find the frontend service
kubectl get svc -n $NAMESPACE | grep frontend

# Port-forward to access locally
kubectl port-forward svc/<deployment>-frontend 8000:8000 -n $NAMESPACE

# Test the endpoint
curl http://localhost:8000/v1/models
```

> [!NOTE]
> DGDR 是 **不可变的**。要更新 SLA 或配置，请删除已有 DGDR 并新建一个。

## Profiling 方法

profiler 遵循 5 步流程：

1. **硬件设置**：使用默认值或用户指定的硬件配置。集群级 operator 还可启用自动 GPU 发现，从集群节点（node）检测规格。
2. **确定扫描范围**：自动确定每个引擎的最小与最大 GPU 数。最小由模型大小与 GPU VRAM 决定。最大对 dense 模型为 1 个节点，对 MoE 模型为 4 个节点。
3. **并行映射扫描**：使用输入的 ISL 与 OSL 在不同并行映射下测试引擎性能。
   - 对 dense 模型，对 prefill 与 decode 测试不同 TP 大小。
   - 对 MoE 模型（SGLang），同时评估 TEP 与 DEP 作为 prefill 与 decode 的候选。
   - **Prefill**：
     - TP/TEP：在 batch size = 1 下（假设 ISL 足够长以打满计算）测量无 KV 复用时的 TTFT。
     - DEP：attention 使用 data parallelism。一次发出总并发为 `attention_dp_size × attn_dp_num_req_ratio`（默认 4）的突发，并把 AIPerf 总结里该突发的 TTFT 计为 `time_to_first_token.max / attn_dp_num_req_ratio`。
   ![Prefill Performance](../../images/h100_prefill_performance.png)
   - **Decode**：在不同 in-flight 请求数（从 1 到 KV 缓存（KV cache）能容纳的最大值）下测量 ITL。为避免被搭车的 prefill 请求影响，脚本启用 KV 复用，并在测量前用相同 prompt 预热引擎。
   ![Decode Performance](../../images/h100_decode_performance.png)
4. **推荐**：在满足 TTFT 与 ITL SLA 的前提下，选择能达到最高单 GPU 吞吐的 prefill 与 decode 并行映射。
5. **对推荐 P/D 引擎做深入 profiling**：用 ISL 对 TTFT、用活跃 KV 缓存与 decode 上下文长度对 ITL 做插值，以更准确地估计性能。
![ITL Interpolation](../../images/pd_interpolation.png)
   - **Prefill**：在 batch size=1 下测量不同输入长度的 TTFT 与单 GPU 吞吐。
   - **Decode**：在不同 KV 缓存负载与 decode 上下文长度下测量 ITL 与单 GPU 吞吐。

### 真实引擎上的 AIPerf

通过在 Kubernetes 中创建真实测试部署并测量其性能，对你的模型做 profiling。

- **耗时**：2–4 小时
- **准确性**：最高（真实测量）
- **GPU 要求**：完整访问以测试不同并行映射
- **后端**：SGLang、TensorRT-LLM、vLLM

基于 AIPerf 的 profiling 是默认行为。使用 `searchStrategy: thorough` 获得全面的真实引擎 profiling：

```yaml
spec:
  searchStrategy: thorough  # Deep exploration with real engine profiling
```

### AI Configurator 模拟

使用性能模拟在不实际部署的情况下，迅速估算最佳配置。

- **耗时**：20–30 秒
- **准确性**：估计值（对不寻常的配置可能有偏差）
- **GPU 要求**：无
- **后端**：所有后端（vLLM、SGLang、TensorRT-LLM）

通过 `searchStrategy: rapid` 默认启用 AI Configurator 模拟：

```yaml
spec:
  searchStrategy: rapid  # Fast profiling with AI Configurator simulation
```

> [!NOTE]
> `aicBackendVersion` 指定 AI Configurator 模拟的 TensorRT-LLM 版本。可用版本见 [AI Configurator supported features](https://github.com/ai-dynamo/aiconfigurator#supported-features)。

**当前支持：**
- **后端**：TensorRT-LLM（版本 0.20.0、1.0.0rc3、1.0.0rc6）
- **系统**：H100 SXM、H200 SXM、B200 SXM、GB200 SXM、A100 SXM
- **模型**：覆盖广泛，包括 GPT、Llama、Mixtral、DeepSeek、Qwen 等

完整列表见 [AI Configurator 文档](https://github.com/ai-dynamo/aiconfigurator#supported-features)。

### 自动 GPU 发现

operator 在条件允许时会自动从 Kubernetes 集群节点发现 GPU 资源。GPU 发现提供：

- 硬件信息（GPU 型号、VRAM、每节点 GPU 数）
- 基于模型大小自动计算 profiling 搜索空间
- 用于 AI Configurator 集成的硬件系统标识

**权限**：GPU 发现需要集群范围的 node 读权限。集群范围 operator 自动具备此权限。命名空间受限的 operator 也可以通过 RBAC 授予 node 读权限来使用 GPU 发现。

如果 GPU 发现不可用（无权限或无 GPU 标签），profiler 将使用手动指定的硬件配置或默认值。

## 配置

### DGDR 配置结构

所有 profiler 配置都通过 v1beta1 DGDR spec 字段提供：

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeploymentRequest
metadata:
  name: my-deployment
spec:
  model: "Qwen/Qwen3-0.6B"
  backend: vllm
  image: "nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1"

  workload: { ... }
  sla: { ... }
  hardware: { ... }
  features: { ... }
  overrides: { ... }
```

### SLA 配置（可选）

```yaml
workload:
  isl: 3000      # Average input sequence length (tokens)
  osl: 150       # Average output sequence length (tokens)

sla:
  ttft: 200.0    # Target Time To First Token (milliseconds)
  itl: 20.0      # Target Inter-Token Latency (milliseconds)
```

- **ISL/OSL**：依据你预期的流量模式
- **TTFT**：首 token 延迟目标（越低需要的 GPU 越多，影响 prefill 引擎）
- **ITL**：token 生成延迟目标（越低需要的 GPU 越多，影响 decode 引擎）
- **权衡**：更严的 SLA 需要更多 GPU 资源

### 硬件配置（可选）

```yaml
hardware:
  gpuSku: h200_sxm            # GPU SKU identifier (auto-detected)
  vramMb: 81920               # VRAM per GPU in MiB
  totalGpus: 16               # Total GPUs available in the cluster
  numGpusPerNode: 8           # GPUs per node (for multi-node MoE)
```

- **numGpusPerNode**：为 dense 模型决定每节点 GPU 数上限，并为多节点 MoE 引擎配置 Grove
- **gpuSku**：GPU SKU 标识，由控制器自动检测

> [!TIP]
> 如果不指定硬件约束，控制器会基于模型大小与可用集群资源自动检测。

### Search Strategy（可选）

控制 profiling 搜索深度：

```yaml
spec:
  searchStrategy: rapid   # "rapid" (default) for fast sweep; "thorough" for deeper exploration
```

- **rapid**：在并行映射上做快速扫描（默认）
- **thorough**：探索更多配置以获得潜在更好结果

### Planner 配置（可选）

通过 features 节为 SLA planner 传参：

```yaml
features:
  planner:
    planner_min_endpoint: 2                    # Minimum endpoints to maintain
    planner_adjustment_interval: 60            # Adjustment interval (seconds)
    planner_load_predictor: linear             # Load prediction method
```

> [!NOTE]
> Planner 参数使用 `planner_` 前缀。完整列表见 AI Configurator 文档。

### 模型缓存 PVC（高级）

对于大模型，使用预先填充模型权重的 PVC，而不是从 HuggingFace 下载：

```yaml
modelCache:
  pvcName: "model-cache"
  pvcModelPath: "hub/models--deepseek-ai--DeepSeek-R1"
  pvcMountPath: "/opt/model-cache"
```

要求：
- PVC 必须存在于与 DGDR 相同的命名空间
- 模型权重必须可在 `{mountPath}/{pvcPath}` 访问

### 引擎配置（自动配置）

控制器会根据高层字段自动处理模型与后端配置：

```yaml
# You specify:
spec:
  model: "Qwen/Qwen3-0.6B"
  backend: vllm

# Controller auto-injects into the profiling job
```

**不要**在 profiling 配置 override 中手动设置 model 或 backend。

### 使用已有的 DGD 配置

通过 overrides 提供基础 DGD 配置：

```yaml
overrides:
  dgd:
    apiVersion: nvidia.com/v1alpha1
    kind: DynamoGraphDeployment
    metadata:
      name: my-dgd
    spec:
      # ... your base DGD spec
```

profiler 会把 DGD 配置作为 **基础模板**，再基于你的 SLA 目标进行优化。

## 集成

### 与 SLA Planner

Profiler 生成的插值数据被 SLA Planner 用于自动伸缩决策。

**Prefill 插值**（`selected_prefill_interpolation/raw_data.npz`）：
- `prefill_isl`：所测试的输入序列长度的一维数组
- `prefill_ttft`：每个 ISL 上 TTFT（ms）的一维数组
- `prefill_thpt_per_gpu`：每个 ISL 上吞吐（tokens/s/GPU）的一维数组

**Decode 插值**（`selected_decode_interpolation/raw_data.npz`）：
- `max_kv_tokens`：decode 引擎中 KV token 总容量
- `x_kv_usage`：活跃 KV 使用率 [0, 1] 的一维数组
- `y_context_length`：所测试的平均上下文长度的一维数组
- `z_itl`：每个 (KV usage, context length) 点上 ITL（ms）的一维数组
- `z_thpt_per_gpu`：每个点上吞吐（tokens/s/GPU）的一维数组

### 与 Dynamo Operator

使用 DGDR 时，Dynamo Operator 会：

1. 自动创建 profiling 任务
2. 把 profiling 数据存入 ConfigMap（`planner-profile-data`）
3. 生成优化后的 DGD 配置
4. 部署带 SLA Planner 集成的 DGD

#### 失败处理

profiling 失败不会在 Kubernetes Job 层重试（`backoffLimit: 0`）。
profiler 的多数错误 —— 校验失败、不支持的“模型/硬件”组合、缺失配置 —— 都是确定性的，重试也不会成功，所以重新跑完整 profiling 周期只会浪费 GPU 时间。

当 profiler 报告失败时，output-copier sidecar 会把错误详情（阶段、错误信息、profiler 状态）写入输出 ConfigMap 并以成功状态退出。DGDR 控制器从 ConfigMap 读取失败信息，把 DGDR 直接转入 `Failed` 阶段，并附带具体的子阶段失败原因（如 `SweepingDecodeFailed`、`GeneratingDGDFailed`）。使用 `kubectl describe dgdr <name>` 在 conditions 中查看失败详情。

生成的 DGD 通过 label 跟踪：
```yaml
metadata:
  labels:
    dgdr.nvidia.com/name: my-deployment
    dgdr.nvidia.com/namespace: your-namespace
```

### 与可观测性

监控 profiling 任务：

```bash
kubectl logs -f job/profile-<dgdr-name> -n $NAMESPACE
kubectl describe dgdr <name> -n $NAMESPACE
```

## 高级话题

### 手动控制部署

关闭自动部署，先审阅生成的 DGD 再应用：

```yaml
spec:
  autoApply: false
```

然后手动提取并应用：

```bash
# Extract generated DGD from DGDR status
kubectl get dgdr my-deployment -n $NAMESPACE -o jsonpath='{.status.profilingResults.selectedConfig}' | kubectl apply -f -

# Or save to file for review
kubectl get dgdr my-deployment -n $NAMESPACE -o jsonpath='{.status.profilingResults.selectedConfig}' > my-dgd.yaml
```

### Mocker 部署

部署一个无需 GPU 即可模拟引擎的 mocker 部署：

```yaml
spec:
  model: <model-name>
  backend: trtllm
  features:
    mocker:
      enabled: true    # Deploy mocker instead of real backend
  autoApply: true
```

profiling 仍在真实后端上运行以采集性能数据。mocker 使用这些数据模拟真实时序行为。适用于大规模实验、测试 Planner 行为以及验证配置。

### 访问 profiling 产物

默认 profiling 数据保存在 ConfigMap。要保存详细产物（图、日志、原始数据），可通过 overrides 挂载 PVC：

```yaml
overrides:
  profilingJob:
    template:
      spec:
        volumes:
        - name: profiling-output
          persistentVolumeClaim:
            claimName: "dynamo-pvc"
```

**ConfigMap（始终创建）：**
- `dgdr-output-<name>`：生成的 DGD 配置
- `planner-profile-data`：用于 Planner 的 profiling 数据（JSON）

**PVC 产物（可选）：**
- 性能图表（PNG）
- 每次 profiled 部署的 DGD 配置
- AIPerf profiling 产物
- 原始 profiling 数据（`.npz` 文件）
- profiler 日志

访问 PVC 中的结果：
```bash
kubectl apply -f deploy/utils/manifests/pvc-access-pod.yaml -n $NAMESPACE
kubectl wait --for=condition=Ready pod/pvc-access-pod -n $NAMESPACE --timeout=60s
kubectl cp $NAMESPACE/pvc-access-pod:/data ./profiling-results
kubectl delete pod pvc-access-pod -n $NAMESPACE
```

### 输出性能图

profiler 会生成可视化图：

**并行映射扫描图：**
- `prefill_performance.png`：TTFT vs 并行映射大小
- `decode_performance.png`：ITL vs 并行映射大小与 in-flight 请求数

**深入 profiling 图：**
- `selected_prefill_interpolation/prefill_ttft_interpolation.png`：TTFT vs ISL
- `selected_prefill_interpolation/prefill_throughput_interpolation.png`：吞吐 vs ISL
- `selected_decode_interpolation/decode_itl_interplation.png`：ITL vs KV 使用率与上下文长度
- `selected_decode_interpolation/decode_throughput_interpolation.png`：吞吐 vs KV 使用率与上下文长度

## 运行时 Profiling（SGLang）

SGLang worker 暴露了用于运行时性能分析的 profiling 端点：

```bash
# Start profiling
curl -X POST http://localhost:9090/engine/start_profile \
  -H "Content-Type: application/json" \
  -d '{"output_dir": "/tmp/profiler_output"}'

# Run inference requests...

# Stop profiling
curl -X POST http://localhost:9090/engine/stop_profile
```

可使用 Chrome 的 `chrome://tracing`、[Perfetto UI](https://ui.perfetto.dev/) 或 TensorBoard 查看 trace。

## 故障排查

### profiling 太慢

**方案 1**：使用 `searchStrategy: rapid` 进行快速 AI Configurator profiling（仅 TensorRT-LLM）：
```yaml
spec:
  searchStrategy: rapid
```

**方案 2**：在 DGDR 中通过指定硬件约束缩小搜索空间：
```yaml
spec:
  hardware:
    numGpusPerNode: 4
    totalGpus: 8
```

### 无法满足 SLA

**症状**：profiler 报告没有配置能满足目标

**方案：**
1. 放宽 SLA 目标（增大 TTFT/ITL）
2. 增加 GPU 资源
3. 尝试不同的后端
4. 使用更小的模型

### AI Configurator：attention head 约束错误

**症状**：profiling 失败，错误：
```text
AssertionError: num_heads <N> should be divisible by tp_size <M> and the division result should be >= 4
```

**原因**：AI Configurator 要求 **每 GPU ≥4 个 attention head**。head 较少的小模型不能使用大 TP。

**受影响模型：**
- **Qwen3-0.6B**（16 头）：最大 TP = 4
- **GPT-2**（12 头）：最大 TP = 3
- 大多数 **<1B 参数** 的模型：可能触发该约束

**方案**：限制 `maxNumGpusPerEngine`：
```yaml
hardware:
  maxNumGpusPerEngine: 4  # For Qwen3-0.6B (16 heads / 4 = max TP of 4)
```

**计算最大 TP**：`max_tp = num_attention_heads / 4`

> [!NOTE]
> 这是 AI Configurator 的限制。在线 profiling 没有此约束。

### 镜像拉取错误

**症状**：`ErrImagePull` 或 `ImagePullBackOff`

**方案**：确保已配置 image pull secret：
```bash
kubectl create secret docker-registry nvcr-imagepullsecret \
  --docker-server=nvcr.io \
  --docker-username='$oauthtoken' \
  --docker-password=<NGC_API_KEY> \
  --namespace <your-namespace>
```

### profiling 期间 OOM

**症状**：profiling 任务中出现 OOM 错误

**方案：**
1. 在引擎配置中减小 `gpu_memory_utilization`
2. 减小 `--max-context-length`
3. 跳过更大的 TP 配置
4. 每次测试使用更少 GPU

### 后端不支持的并行映射

**症状**：后端的启动/运行时错误（如质数个 attention head 限制 TP=1，或后端不支持 prefill 与 decode 使用不同 TP 大小）。

**方案：**
1. 联系后端添加支持，并提升 Dynamo 中的后端版本
2. 把每个引擎的最大与最小 GPU 数限制在受支持范围内

## 另请参阅

- [DGDR Examples](../../../docs/components/profiler/profiler-examples.md) - 完整 DGDR YAML 示例
- [DGDR API Reference](/docs/kubernetes/api-reference.md) - DGDR 规格
- [Profiler Arguments Reference](https://github.com/ai-dynamo/dynamo/blob/main/components/src/dynamo/profiler/utils/dgdr_v1beta1_types.py) - 完整配置参考
