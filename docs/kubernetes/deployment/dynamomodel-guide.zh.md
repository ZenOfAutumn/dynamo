---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Managing Models with DynamoModel
---

## 概述

`DynamoModel` 是一个 Kubernetes 自定义资源，用于表示部署在 Dynamo 上的机器学习模型。它能让你：

- **在运行中的基础模型上部署 LoRA 适配器**
- **跨集群跟踪模型端点**及其就绪状态
- **以 Kubernetes 声明式方式管理模型生命周期**

DynamoModel 与 `DynamoGraphDeployment`（DGD）或 `DynamoComponentDeployment`（DCD）资源协同工作。DGD/DCD 部署推理（inference）基础设施（pod、service），DynamoModel 则处理模型相关操作，如加载 LoRA 适配器。

## 快速开始

### 前置条件

创建 DynamoModel 之前，你需要：

1. 一个正在运行的 `DynamoGraphDeployment` 或 `DynamoComponentDeployment`
2. 配置了指向基础模型的 `modelRef` 的组件
3. pod 已就绪并提供基础模型服务

完整的搭建（包括 DGD 配置）参见 [与 DynamoGraphDeployment 的集成](#integration-with-dynamographdeployment)。

### 部署 LoRA 适配器

**1. 创建 DynamoModel：**

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoModel
metadata:
  name: my-lora
  namespace: dynamo-system
spec:
  modelName: my-custom-lora
  baseModelName: Qwen/Qwen3-0.6B  # Must match modelRef.name in your DGD
  modelType: lora
  source:
    uri: s3://my-bucket/loras/my-lora
```

**2. 应用并验证：**

```bash
# Apply the DynamoModel
kubectl apply -f my-lora.yaml

# Check status
kubectl get dynamomodel my-lora
```

**预期输出：**
```
NAME      TOTAL   READY   AGE
my-lora   2       2       30s
```

完成！operator 会自动发现端点并加载 LoRA。

详细状态监控请参阅 [监控与运维](#monitoring--operations)。

## 理解 DynamoModel

### 模型类型

DynamoModel 支持三种模型类型：

| 类型 | 说明 | 适用场景 |
|------|-------------|----------|
| **`base`** | 引用既有基础模型 | 跟踪基础模型的端点（默认） |
| **`lora`** | 扩展基础模型的 LoRA 适配器 | 在既有模型上部署微调适配器 |
| **`adapter`** | 通用模型适配器 | 为其他适配器类型预留扩展性 |

大多数用户会使用 **`lora`** 在其基础模型部署之上部署微调模型。

### 工作原理

创建 DynamoModel 后，operator 会：

1. **发现端点**：找到所有运行 `baseModelName` 的 pod（通过匹配 DGD/DCD 中的 `modelRef.name`）
2. **创建 service**：自动创建 Kubernetes Service 跟踪这些 pod
3. **加载 LoRA**：在每个端点调用 LoRA load API（针对 `lora` 类型）
4. **更新状态**：报告哪些端点已就绪

**关键关联：**
```yaml
# DGD modelRef.name ↔ DynamoModel baseModelName must match
Worker:
  modelRef:
    name: Qwen/Qwen3-0.6B
---
spec:
  baseModelName: Qwen/Qwen3-0.6B
```

## 配置概览

DynamoModel 部署模型或适配器只需少量关键字段：

| 字段 | 必填 | 用途 | 示例 |
|-------|----------|---------|---------|
| `modelName` | 是 | 模型标识 | `my-custom-lora` |
| `baseModelName` | 是 | 关联到 DGD modelRef | `Qwen/Qwen3-0.6B` |
| `modelType` | 否 | 类型：base/lora/adapter | `lora`（默认 `base`） |
| `source.uri` | LoRA 必填 | 模型位置 | `s3://bucket/path` 或 `hf://org/model` |

**最小 LoRA 配置示例：**
```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoModel
metadata:
  name: my-lora
spec:
  modelName: my-custom-lora
  baseModelName: Qwen/Qwen3-0.6B
  modelType: lora
  source:
    uri: s3://my-bucket/my-lora
```

**完整字段规范、校验规则与所有选项，请参阅：**
📖 [DynamoModel API 参考](../api-reference.md#dynamomodel)

### 状态摘要

状态会显示已发现的端点及其就绪情况：

```bash
kubectl get dynamomodel my-lora
```

**关键状态字段：**
- `totalEndpoints` / `readyEndpoints`：发现的与就绪的端点数
- `endpoints[]`：含地址、pod 名与就绪状态的列表
- `conditions`：标准 Kubernetes 条件（EndpointsReady、ServicesFound）

详细状态使用方式请参阅下方 [监控与运维](#monitoring--operations)。

## 常见用例

### 用例 1：S3 托管的 LoRA 适配器

部署存放在 S3 桶中的 LoRA 适配器。

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoModel
metadata:
  name: customer-support-lora
  namespace: production
spec:
  modelName: customer-support-adapter-v1
  baseModelName: meta-llama/Llama-3.3-70B-Instruct
  modelType: lora
  source:
    uri: s3://my-models-bucket/loras/customer-support/v1
```

**前置条件：**
- pod 可访问的 S3 桶（IAM role 或凭据）
- 通过 DGD/DCD 已运行 `meta-llama/Llama-3.3-70B-Instruct`

**验证：**
```bash
# Check LoRA is loaded
kubectl get dynamomodel customer-support-lora -o jsonpath='{.status.readyEndpoints}'
# Should output: 2 (or your number of replicas)

# View which pods are serving
kubectl get dynamomodel customer-support-lora -o jsonpath='{.status.endpoints[*].podName}'
```

### 用例 2：HuggingFace 托管的 LoRA

从 HuggingFace Hub 部署 LoRA 适配器。

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoModel
metadata:
  name: multilingual-lora
  namespace: dynamo-system
spec:
  modelName: multilingual-adapter
  baseModelName: Qwen/Qwen3-0.6B
  modelType: lora
  source:
    uri: hf://myorg/qwen-multilingual-lora@v1.0.0  # Optional: @revision
```

**前置条件：**
- pod 可访问 HuggingFace Hub
- 私有仓库：将 HF token 配置为 secret 并挂载到 pod
- 通过 DGD/DCD 已运行基础模型 `Qwen/Qwen3-0.6B`

**附带 HuggingFace token：**
```yaml
# In your DGD/DCD
spec:
  services:
    worker:
      envFromSecret: hf-token-secret  # Provides HF_TOKEN env var
      modelRef:
        name: Qwen/Qwen3-0.6B
      # ... rest of config
```

### 用例 3：同一基础模型上的多个 LoRA

在同一基础模型部署上部署多个 LoRA 适配器。

```yaml
---
# LoRA for customer support
apiVersion: nvidia.com/v1alpha1
kind: DynamoModel
metadata:
  name: support-lora
spec:
  modelName: support-adapter
  baseModelName: Qwen/Qwen3-0.6B
  modelType: lora
  source:
    uri: s3://models/support-lora

---
# LoRA for code generation
apiVersion: nvidia.com/v1alpha1
kind: DynamoModel
metadata:
  name: code-lora
spec:
  modelName: code-adapter
  baseModelName: Qwen/Qwen3-0.6B  # Same base model
  modelType: lora
  source:
    uri: s3://models/code-lora
```

两个 LoRA 都会被加载到所有提供 `Qwen/Qwen3-0.6B` 服务的 pod 上。你的应用可据此将请求路由到对应适配器。

## 监控与运维

### 检查状态

**快速状态检查：**
```bash
kubectl get dynamomodel
```

**示例输出：**
```
NAME              TOTAL   READY   AGE
my-lora           2       2       5m
customer-lora     4       3       2h
```

**详细状态：**
```bash
kubectl describe dynamomodel my-lora
```

**示例输出：**
```
Name:         my-lora
Namespace:    dynamo-system
Spec:
  Model Name:       my-custom-lora
  Base Model Name:  Qwen/Qwen3-0.6B
  Model Type:       lora
  Source:
    Uri:  s3://my-bucket/my-lora
Status:
  Ready Endpoints:  2
  Total Endpoints:  2
  Endpoints:
    Address:   http://10.0.1.5:9090
    Pod Name:  worker-0
    Ready:     true
    Address:   http://10.0.1.6:9090
    Pod Name:  worker-1
    Ready:     true
  Conditions:
    Type:     EndpointsReady
    Status:   True
    Reason:   EndpointsDiscovered
Events:
  Type    Reason              Message
  ----    ------              -------
  Normal  EndpointsReady      Discovered 2 ready endpoints for base model Qwen/Qwen3-0.6B
```

### 理解就绪状态

某个端点处于 **ready** 当且仅当：
1. pod 在运行且健康
2. LoRA load API 调用成功

**条件状态：**
- `EndpointsReady=True`：所有端点就绪（完全可用）
- `EndpointsReady=False, Reason=NotReady`：并非所有端点就绪（详见 message 中的计数）
- `EndpointsReady=False, Reason=NoEndpoints`：未找到端点

当 `readyEndpoints < totalEndpoints` 时，operator 每 30 秒自动重试加载。

### 查看端点

**获取端点地址：**
```bash
kubectl get dynamomodel my-lora -o jsonpath='{.status.endpoints[*].address}' | tr ' ' '\n'
```

**输出：**
```
http://10.0.1.5:9090
http://10.0.1.6:9090
```

**获取端点 pod 名：**
```bash
kubectl get dynamomodel my-lora -o jsonpath='{.status.endpoints[*].podName}' | tr ' ' '\n'
```

**检查每个端点的就绪状态：**
```bash
kubectl get dynamomodel my-lora -o json | jq '.status.endpoints[] | {podName, ready}'
```

**输出：**
```json
{
  "podName": "worker-0",
  "ready": true
}
{
  "podName": "worker-1",
  "ready": true
}
```

### 更新模型

更新 LoRA（例如部署新版本）：

```bash
# Edit the source URI
kubectl edit dynamomodel my-lora

# Or apply an updated YAML
kubectl apply -f my-lora-v2.yaml
```

operator 会检测到变更并在所有端点上重新加载 LoRA。

### 删除模型

```bash
kubectl delete dynamomodel my-lora
```

对于 LoRA 模型，operator 会：
1. 从所有端点卸载 LoRA
2. 清理相关资源
3. 移除 DynamoModel CR

基础模型部署（DGD/DCD）会继续正常运行。

## 故障排查

### 未找到端点

**现象：**
```yaml
status:
  totalEndpoints: 0
  readyEndpoints: 0
  conditions:
  - type: EndpointsReady
    status: "False"
    reason: NoEndpoints
    message: "No endpoint slices found for base model Qwen/Qwen3-0.6B"
```

**常见原因：**

1. **基础模型部署未运行**
   ```bash
   # Check if pods exist
   kubectl get pods -l nvidia.com/dynamo-component-type=worker
   ```
   **解决方法：** 先部署 DGD/DCD，等待 pod 就绪。

2. **`baseModelName` 不匹配**
   ```bash
   # Check modelRef in your DGD
   kubectl get dynamographdeployment my-deployment -o yaml | grep -A2 modelRef
   ```
   **解决方法：** 确保 DynamoModel 的 `baseModelName` 与 DGD 中 `modelRef.name` 完全一致。

3. **pod 未就绪**
   ```bash
   # Check pod status
   kubectl get pods -l nvidia.com/dynamo-component-type=worker
   ```
   **解决方法：** 等待 pod 进入 `Running` 与 `Ready`。

4. **命名空间错误**
   **解决方法：** 确保 DynamoModel 与 DGD/DCD 在同一命名空间。

### LoRA 加载失败

**现象：**
```yaml
status:
  totalEndpoints: 2
  readyEndpoints: 0  # ← No endpoints ready despite pods existing
  conditions:
  - type: EndpointsReady
    status: "False"
    reason: NoReadyEndpoints
```

**常见原因：**

1. **source URI 不可访问**
   ```bash
   # Check operator logs
   kubectl logs -n dynamo-system deployment/dynamo-operator-controller-manager -f | grep "Failed to load"
   ```
   **解决方法：**
   - S3：确认桶权限、IAM role、凭据
   - HuggingFace：确认 token 有效，仓库存在且可访问

2. **LoRA 格式不合法**
   **解决方法：** 确保 LoRA 权重格式与你后端框架（SGLang、vLLM 等）期望一致。

3. **端点 API 错误**
   ```bash
   # Check operator logs for HTTP errors
   kubectl logs -n dynamo-system deployment/dynamo-operator-controller-manager | grep "error"
   ```
   **解决方法：** 检查 worker pod 中后端框架的日志：
   ```bash
   kubectl logs worker-0
   ```

4. **内存不足**
   **解决方法：** LoRA 适配器需要额外内存。增大 DGD 内存限制：
   ```yaml
   resources:
     limits:
       memory: "32Gi"  # Increase if needed
   ```

### 状态显示未就绪

**现象：**
部分端点长时间未就绪。

**诊断：**
```bash
# Check which endpoints are not ready
kubectl get dynamomodel my-lora -o json | jq '.status.endpoints[] | select(.ready == false)'

# View operator logs for that specific pod
kubectl logs -n dynamo-system deployment/dynamo-operator-controller-manager | grep "worker-0"

# Check the worker pod logs
kubectl logs worker-0 | tail -50
```

**常见原因：**

1. **网络问题**：pod 无法访问 S3/HuggingFace
2. **资源约束**：pod 发生 OOM 或被限流
3. **API 端点未响应**：后端框架未提供 LoRA API

**何时等待 vs. 调查：**
- **等待**：若 readyEndpoints 在持续上升（LoRA 正在逐步加载）
- **调查**：若 readyEndpoints 卡在同一值超过 5 分钟

### 查看 events 与日志

**查看 events：**
```bash
kubectl describe dynamomodel my-lora | tail -20
```

**查看 operator 日志：**
```bash
# Follow logs
kubectl logs -n dynamo-system deployment/dynamo-operator-controller-manager -f

# Filter for specific model
kubectl logs -n dynamo-system deployment/dynamo-operator-controller-manager | grep "my-lora"
```

**常见 event 与消息：**

| Event/Message | 含义 | 操作 |
|---------------|---------|--------|
| `EndpointsReady` | 全部端点就绪 | ✅ 良好 - 服务完全可用 |
| `NotReady` | 并非全部就绪 | ⚠️ 检查 readyEndpoints —— operator 会重试 |
| `PartialEndpointFailure` | 部分端点加载失败 | 查看错误日志 |
| `NoEndpointsFound` | 未发现 pod | 确认 DGD 已运行且 modelRef 匹配 |
| `EndpointDiscoveryFailed` | 无法查询端点 | 检查 operator 的 RBAC 权限 |
| `Successfully reconciled` | 协调完成 | ✅ 良好 |

## 与 DynamoGraphDeployment 的集成

本节展示一起部署基础模型与 LoRA 适配器的完整端到端工作流。

DynamoModel 与 DynamoGraphDeployment 协同工作以提供完整的模型部署：

- **DGD**：部署基础设施（pod、service、资源）
- **DynamoModel**：管理模型相关操作（LoRA 加载）

### 将模型与组件关联

通过 DGD 中的 `modelRef` 字段建立联接：

**完整示例：**

```yaml
---
# 1. Deploy the base model infrastructure
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: my-deployment
spec:
  backendFramework: vllm
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:latest

    Worker:
      # This modelRef creates the link to DynamoModel
      modelRef:
        name: Qwen/Qwen3-0.6B  # ← Key linking field

      componentType: worker
      replicas: 2
      resources:
        limits:
          gpu: "1"
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:latest
          args:
            - --model
            - Qwen/Qwen3-0.6B
            - --tensor-parallel-size
            - "1"

---
# 2. Deploy LoRA adapters on top
apiVersion: nvidia.com/v1alpha1
kind: DynamoModel
metadata:
  name: my-lora
spec:
  modelName: my-custom-lora
  baseModelName: Qwen/Qwen3-0.6B  # ← Must match modelRef.name above
  modelType: lora
  source:
    uri: s3://my-bucket/loras/my-lora
```

### 部署工作流

**推荐顺序：**

```bash
# 1. Deploy base model infrastructure
kubectl apply -f my-deployment.yaml

# 2. Wait for pods to be ready
kubectl wait --for=condition=ready pod -l nvidia.com/dynamo-component-type=worker --timeout=5m

# 3. Deploy LoRA adapters
kubectl apply -f my-lora.yaml

# 4. Verify LoRA is loaded
kubectl get dynamomodel my-lora
```

**幕后流程：**

| 步骤 | DGD | DynamoModel |
|------|-----|-------------|
| 1 | 创建带 modelRef 的 pod | - |
| 2 | pod 进入 running 与 ready | - |
| 3 | - | CR 创建，通过自动创建的 Service 发现端点 |
| 4 | - | 在每个端点调用 LoRA load API |
| 5 | - | 所有端点就绪 ✓ |

operator 自动处理所有服务发现——你不需要手动配置 service、label 或 selector。

## API 参考

完整字段规范、校验规则与详细类型定义请参阅：

**📖 [Dynamo CRD API 参考](../api-reference.md#dynamomodel)**

## 总结

DynamoModel 为 Dynamo 部署提供了声明式的模型管理：

✅ **简单**：两步部署 LoRA 适配器
✅ **自动化**：端点发现与加载由 operator 处理
✅ **可观察**：丰富的状态报告与 conditions
✅ **集成**：与 DynamoGraphDeployment 无缝协作

**后续步骤：**
- 试一试 [快速开始](#quick-start) 示例
- 探索 [常见用例](#common-use-cases)
- 查看 [API 参考](../api-reference.md#dynamomodel) 了解高级配置
