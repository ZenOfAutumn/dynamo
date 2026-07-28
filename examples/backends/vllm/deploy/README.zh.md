# vLLM Kubernetes 部署配置

本目录包含使用 **DynamoGraphDeployment** 资源部署 vLLM 推理图（inference graphs）的 Kubernetes Custom Resource Definition（CRD）模板。

## 可用部署模式

### 1. **聚合部署**（`agg.yaml`）
基础部署模式，包含前端（frontend）与单个 decode worker。

**架构：**
- `Frontend`：兼容 OpenAI 的 API 服务（KV router 模式禁用）
- `VLLMDecodeWorker`：单个同时处理 prefill 与 decode 的 worker

### 2. **聚合 + 路由部署**（`agg_router.yaml`）
增强型聚合部署，启用 KV 缓存（KV cache）路由能力。

**架构：**
- `Frontend`：兼容 OpenAI 的 API 服务（KV router 模式启用）
- `VLLMDecodeWorker`：单个同时处理 prefill 与 decode 的 worker

### 3. **解耦部署**（`disagg.yaml`）
高性能部署，prefill 与 decode worker 分离。

**架构：**
- `Frontend`：在 worker 之间协调的 HTTP API 服务
- `VLLMDecodeWorker`：仅处理 decode 的专用 worker
- `VLLMPrefillWorker`：仅处理 prefill 的专用 worker（`--disaggregation-mode prefill`）
- 通过 NIXL 传输后端通信

### 4. **解耦 + 路由部署**（`disagg_router.yaml`）
带 KV 缓存路由能力的高级解耦部署。

**架构：**
- `Frontend`：带 KV 感知路由的 HTTP API 服务
- `VLLMDecodeWorker`：仅处理 decode 的专用 worker
- `VLLMPrefillWorker`：仅处理 prefill 的专用 worker（`--disaggregation-mode prefill`）

### 5. **Global Planner 部署**（参见 [`examples/global_planner/`](../../../global_planner/)）
通过 GlobalPlanner 在多个 DGD 之间集中扩缩容。包含单端点多池与多模型 GPU 预算等示例。详见 [global planner 示例](../../../global_planner/)。

### 6. **基于 Intel XPU 的部署（可选）**（`agg_xpu_dra.yaml` 或 `disagg_xpu_dra.yaml`）
使用 Kubernetes Dynamic Resource Allocation（DRA）的硬件特定聚合/解耦部署。

**聚合架构：**
- `Frontend`：兼容 OpenAI 的 API 服务
- `VllmDecodeWorker`：以 XPU 为目标设备的单个 worker（`VLLM_TARGET_DEVICE=xpu`）
- 通过 `ResourceClaimTemplate` 与 pod 级 `resourceClaims` 分配 GPU

**解耦架构：**
- `Frontend`：在 worker 之间协调的 HTTP API 服务
- `VllmDecodeWorker`：以 XPU 为目标的仅 decode worker
- `VllmPrefillWorker`：以 XPU 为目标的仅 prefill worker
- 通过 `ResourceClaimTemplate` 与 pod 级 `resourceClaims` 分配 GPU
- 通过 NIXL 传输后端 + XPU buffer 通信

## CRD 结构

所有模板都使用 **DynamoGraphDeployment** CRD：

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: <deployment-name>
spec:
  services:
    <ServiceName>:
      # Service configuration
```

### 关键配置项

**资源管理：**
```yaml
resources:
  requests:
    cpu: "10"
    memory: "20Gi"
    gpu: "1"
  limits:
    cpu: "10"
    memory: "20Gi"
    gpu: "1"
```

**容器配置：**
```yaml
extraPodSpec:
  mainContainer:
    image: my-registry/vllm-runtime:my-tag
    workingDir: /workspace/examples/backends/vllm
    args:
      - "python3"
      - "-m"
      - "dynamo.vllm"
      - "--model"
      - "Qwen/Qwen3-0.6B"
      # Optional: Enable prompt embeddings feature
      # - "--enable-prompt-embeds"
      # Other model-specific arguments
```

**常用 vLLM 参数：**
- `--enable-prompt-embeds`：启用 prompt embeddings 功能
- `--enable-multimodal`：启用多模态（视觉）支持
- `--disaggregation-mode prefill`：解耦服务下的仅 prefill 模式
- `--kv-transfer-config '<json>'`：KV 传输后端配置（如 `'{"kv_connector":"NixlConnector","kv_role":"kv_both"}'`）

## 前置条件

使用这些模板前，请确保你具备：

1. **已安装 Dynamo Kubernetes Platform** —— 参见 [快速入门指南](../../../../docs/kubernetes/README.md)
2. **支持 GPU 的 Kubernetes 集群**
3. **可访问的镜像仓库**（用于 vLLM 运行时镜像；默认 NGC CUDA 镜像可选——`nvcr.io/nvidia/ai-dynamo/*` 公共可访问；Intel XPU 用户需以 `--device xpu` 构建自定义镜像）
4. **HuggingFace token secret**（被引用为 `envFromSecret: hf-token-secret`）

### 容器镜像

[NGC Catalog](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/collections/ai-dynamo/artifacts) 上提供了公共镜像。如果你想使用自己的镜像仓库，可以自行构建并推送：

```bash
python container/render.py --framework=vllm --output-short-filename
docker build -f container/rendered.Dockerfile .
# Tag and push to your container registry
# Update the image references in the YAML files
```

### 部署前剖析（仅 SLA Planner）

如使用 SLA Planner 部署（`disagg_planner.yaml`），请按照 [部署前剖析指南](../../../../docs/components/profiler/profiler-guide.md) 运行部署前剖析。

## 用法

### 1. 选择模板
按需求选择对应部署模式：
- `agg.yaml`：简单测试
- `agg_router.yaml`：带负载均衡的生产部署
- `disagg.yaml`：极致性能
- `disagg_router.yaml`：带 KV 缓存路由的高性能
- `disagg_planner.yaml`：SLA 优化性能
- `agg_xpu_dra.yaml`：Intel XPU 集群上使用 K8s DRA 的聚合部署
- `disagg_xpu_dra.yaml`：Intel XPU 集群上使用 K8s DRA 的解耦部署
- 多 DGD 集中扩缩容：参见 [global planner 示例](../../../global_planner/)

### 2. 自定义配置
按你的环境编辑模板：

```yaml
# Update image registry and tag
image: my-registry/vllm-runtime:my-tag

# Configure your model
args:
  - "--model"
  - "your-org/your-model"
```

### 3. 部署

使用以下命令部署。

首先创建 HuggingFace token 的 secret：
```bash
export HF_TOKEN=your_hf_token
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=${HF_TOKEN} \
  -n ${NAMESPACE}
```

然后用部署文件部署模型。

导出 Dynamo Kubernetes Platform 安装时使用的 NAMESPACE。

```bash
cd <dynamo-source-root>/examples/backends/vllm/deploy
export DEPLOYMENT_FILE=agg.yaml

kubectl apply -f $DEPLOYMENT_FILE -n $NAMESPACE
```

#### 使用 Intel XPU 部署（可选）
如果你的集群通过 Kubernetes Dynamic Resource Allocation（DRA）使用 Intel GPU 设备，请确保：
- Kubernetes 集群版本为 **v1.34+**（DRA API v1 所必需），且
- 已安装 [Intel XPU Resource Driver](https://github.com/intel/intel-resource-drivers-for-kubernetes)。

部署 XPU 模板（包含 ResourceClaimTemplate）：
```bash
cd <dynamo-source-root>/examples/backends/vllm/deploy

# For aggregated deployment
kubectl apply -f agg_xpu_dra.yaml -n $NAMESPACE

# OR for disaggregated deployment
kubectl apply -f disagg_xpu_dra.yaml -n $NAMESPACE
```

验证 claim 分配：

```bash
kubectl get resourceclaim -n $NAMESPACE
kubectl get dynamographdeployment -n $NAMESPACE
```

`agg_xpu_dra.yaml` 与 `disagg_xpu_dra.yaml` 是可选的硬件特定模板，不会改变 `agg.yaml` 与 `disagg.yaml` 定义的默认部署路径。

### 4. 为 vLLM 使用自定义 Dynamo Frameworks 镜像

要为 vLLM 使用自定义的 dynamo frameworks 镜像，可使用 yq 更新部署文件：

```bash
export DEPLOYMENT_FILE=agg.yaml
export FRAMEWORK_RUNTIME_IMAGE=<vllm-image>

yq '.spec.services.[].extraPodSpec.mainContainer.image = env(FRAMEWORK_RUNTIME_IMAGE)' $DEPLOYMENT_FILE  > $DEPLOYMENT_FILE.generated
kubectl apply -f $DEPLOYMENT_FILE.generated -n $NAMESPACE
```

### 5. 端口转发

部署后转发 frontend 服务以访问 API：

```bash
kubectl port-forward deployment/vllm-v1-disagg-frontend-<pod-uuid-info> 8000:8000
```

## 配置选项

### 环境变量

修改 `DYN_LOG` 级别，编辑 yaml 文件添加：

```yaml
...
spec:
  envs:
    - name: DYN_LOG
      value: "debug" # or other log levels
  ...
```

### vLLM Worker 配置

vLLM worker 通过命令行参数配置。关键参数包括：

- `--model`：要服务的模型（如 `Qwen/Qwen3-0.6B`）
- `--disaggregation-mode prefill`：启用解耦服务下的仅 prefill 模式
- `--metrics-endpoint-port`：向 Dynamo 发布 KV 指标（metrics）的端口

完整配置选项参见 [vLLM CLI 文档](https://docs.vllm.ai/en/v0.9.2/configuration/serve_args.html?h=serve+arg)。

## 测试部署

发送测试请求验证部署：

```bash
curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [
    {
        "role": "user",
        "content": "In the heart of Eldoria, an ancient land of boundless magic and mysterious creatures, lies the long-forgotten city of Aeloria. Once a beacon of knowledge and power, Aeloria was buried beneath the shifting sands of time, lost to the world for centuries. You are an intrepid explorer, known for your unparalleled curiosity and courage, who has stumbled upon an ancient map hinting at ests that Aeloria holds a secret so profound that it has the potential to reshape the very fabric of reality. Your journey will take you through treacherous deserts, enchanted forests, and across perilous mountain ranges. Your Task: Character Background: Develop a detailed background for your character. Describe their motivations for seeking out Aeloria, their skills and weaknesses, and any personal connections to the ancient city or its legends. Are they driven by a quest for knowledge, a search for lost familt clue is hidden."
    }
    ],
    "stream": false,
    "max_tokens": 30
  }'
```

## 模型配置

所有模板默认使用 **Qwen/Qwen3-0.6B**，但你可以使用任何 vLLM 支持的 LLM 模型与配置参数。

## 监控与健康检查

- **前端健康端点**：`http://<frontend-service>:8000/health`
- **存活探针**：定期检查进程健康
- **KV 指标**：通过 metrics 端口发布

## 请求迁移

可在 worker 配置中加入迁移上限参数，启用 [请求迁移](../../../../docs/fault-tolerance/request-migration.md) 以优雅处理 worker 故障：

```yaml
args:
  - "--migration-limit"
  - "3"
```

## 延伸阅读

- **部署指南**：[创建 Kubernetes 部署](../../../../docs/kubernetes/deployment/create-deployment.md)
- **快速入门**：[部署快速入门](../../../../docs/kubernetes/README.md)
- **平台搭建**：[Dynamo Kubernetes Platform 安装](../../../../docs/kubernetes/installation-guide.md)
- **SLA Planner**：[SLA Planner 快速入门指南](../../../../docs/components/planner/planner-guide.md)
- **Global Planner**：[Global Planner 部署指南](../../../../docs/components/planner/global-planner.md)
- **示例**：[部署示例](../../../../docs/getting-started/examples.md)
- **架构文档**：[解耦服务](../../../../docs/design-docs/disagg-serving.md)、[KV 感知路由](../../../../docs/components/router/README.md)

## 故障排查

常见问题与解决方法：

1. **Pod 无法启动**：检查镜像仓库访问与 HuggingFace token secret
2. **未分配 GPU**：确认集群有 GPU 节点且资源 limits 正确
3. **健康检查失败**：查看模型加载日志并适当增大 `initialDelaySeconds`
4. **内存不足（OOM）**：增加内存 limits 或减小 batch size
5. **端口转发问题**：确保 port-forward 命令中使用正确的 pod UUID

如需更多支持，请参阅 [部署故障排查指南](../../../../docs/kubernetes/README.md)。
