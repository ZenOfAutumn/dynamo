# SGLang Kubernetes 部署配置

本目录包含使用 **DynamoGraphDeployment** 资源部署 SGLang 推理（inference）图的 Kubernetes 自定义资源定义（CRD）模板。

## 可选部署模式

### 1. **聚合部署**（`agg.yaml`）
基础部署模式：包含一个前端（frontend）与单个解码（decode）worker。

**架构：**
- `Frontend`：兼容 OpenAI 的 API 服务器
- `SGLangDecodeWorker`：同时处理预填充（prefill）与解码（decode）的单 worker

### 2. **聚合 + 路由（router）部署**（`agg_router.yaml`）
在聚合部署的基础上引入了 KV 缓存（KV cache）路由能力。

**架构：**
- `Frontend`：启用了路由模式（`--router-mode kv`）的兼容 OpenAI API 服务器
- `SGLangDecodeWorker`：同时处理预填充与解码的单 worker

### 3. **解耦部署**（`disagg.yaml`）**
具备分离的预填充与解码 worker 的高性能部署。

**架构：**
- `Frontend`：在 worker 之间协调的 HTTP API 服务器
- `SGLangDecodeWorker`：仅解码的专用 worker（`--disaggregation-mode decode`）
- `SGLangPrefillWorker`：仅预填充的专用 worker（`--disaggregation-mode prefill`）
- 通过 NIXL 传输后端（backend）通信（`--disaggregation-transfer-backend nixl`）

## CRD 结构

所有模板均使用 **DynamoGraphDeployment** CRD：

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
    image: my-registry/sglang-runtime:my-tag
    workingDir: /workspace/examples/backends/sglang
    args:
      - "python3"
      - "-m"
      - "dynamo.sglang"
      # Model-specific arguments
```

## 前置条件

使用这些模板前，请确保：

1. **已安装 Dynamo Kubernetes Platform** - 参见 [安装 Dynamo Kubernetes Platform](../../../../docs/kubernetes/installation-guide.md)
2. **支持 GPU 的 Kubernetes 集群**
3. **可访问的容器镜像仓库**，用于获取 SGLang 运行时镜像
4. **HuggingFace token secret**（在配置中以 `envFromSecret: hf-token-secret` 引用）

## 用法

### 1. 选择模板
按需选择部署模式：
- 开发/测试使用 `agg.yaml`
- 带负载均衡的生产环境使用 `agg_router.yaml`
- 追求极致性能使用 `disagg.yaml`

### 2. 自定义配置
按你的环境编辑模板：

```yaml
# Update image registry and tag
image: my-registry/sglang-runtime:my-tag

# Configure your model
args:
  - "--model-path"
  - "your-org/your-model"
  - "--served-model-name"
  - "your-org/your-model"
```

### 3. 部署

按以下命令应用部署文件。

首先，为 HuggingFace token 创建 secret。
```bash
export HF_TOKEN=your_hf_token
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=${HF_TOKEN} \
  -n ${NAMESPACE}
```

然后使用部署文件部署模型。

```bash
export DEPLOYMENT_FILE=agg.yaml
kubectl apply -f $DEPLOYMENT_FILE -n ${NAMESPACE}
```

### 4. 为 SGLang 使用自定义 Dynamo Frameworks 镜像

要为 SGLang 使用自定义的 dynamo frameworks 镜像，可使用 yq 更新部署文件：

```bash
export DEPLOYMENT_FILE=agg.yaml
export FRAMEWORK_RUNTIME_IMAGE=<sglang-image>

yq '.spec.services.[].extraPodSpec.mainContainer.image = env(FRAMEWORK_RUNTIME_IMAGE)' $DEPLOYMENT_FILE  > $DEPLOYMENT_FILE.generated
kubectl apply -f $DEPLOYMENT_FILE.generated -n $NAMESPACE
```

## 模型配置

所有模板默认使用 **DeepSeek-R1-Distill-Llama-8B** 模型。但你可以使用任意 sglang 参数与配置。关键参数：

## 监控与健康检查

- **前端健康检查端点**：`http://<frontend-service>:8000/health`
- **存活探针（liveness probes）**：每 60 秒检查一次进程健康

## 延伸阅读

- **部署指南**：[创建 Kubernetes 部署](../../../../docs/kubernetes/deployment/create-deployment.md)
- **快速开始**：[部署快速开始](../../../../docs/kubernetes/README.md)
- **平台搭建**：[Dynamo Kubernetes Platform 安装](../../../../docs/kubernetes/installation-guide.md)
- **示例**：[部署示例](../../../../docs/getting-started/examples.md)
- **Kubernetes CRD**：[Custom Resources 文档](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

## 故障排查

常见问题与处理方式：

1. **Pod 启动失败**：检查镜像仓库访问权限与 HuggingFace token secret
2. **未分配 GPU**：确认集群已具备 GPU 节点并配置了正确的资源限制
3. **健康检查失败**：查看模型加载日志并增大 `initialDelaySeconds`
4. **内存不足（Out of memory）**：调高内存限制或减小模型 batch size

如需进一步帮助，请参考 [部署指南](../../../../docs/kubernetes/README.md)。
