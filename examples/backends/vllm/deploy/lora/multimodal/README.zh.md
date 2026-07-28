# 在 Kubernetes 上使用 MinIO 部署多模态 LoRA

本指南介绍如何在 Kubernetes 环境下，基于 S3 兼容存储后端部署支持多模态（视觉-语言）LoRA 的 vLLM 推理（inference）。

## 概览

该部署模式支持在 Kubernetes 中从 S3 兼容存储（MinIO）动态加载用于视觉-语言模型的 LoRA 适配器（adapter）。它采用聚合（aggregated）单 worker 架构，由 frontend 中的 Rust OpenAIPreprocessor 直接处理图像 URL。

## 前置条件

- 支持 GPU 的 Kubernetes 集群
- 已安装 Helm 3.x
- 已配置可访问集群的 `kubectl`
- 已安装 Dynamo Kubernetes Platform（[安装指南](../../../../../../docs/kubernetes/installation-guide.md)）
- 用于下载基础模型与 LoRA 适配器的 HuggingFace token

## 本目录中的文件

| 文件 | 描述 |
|------|-------------|
| `agg_qwen_lora.yaml` | 支持 LoRA 的多模态 vLLM 的 DynamoGraphDeployment |
| `minio-secret.yaml` | MinIO 凭据的 Kubernetes secret |
| `sync-lora-job.yaml` | 从 HuggingFace 下载 LoRA 并上传到 MinIO 的 Job |
| `lora-model.yaml` | 用于注册 LoRA 适配器的 DynamoModel CRD |

---

## 第 1 步：设置环境变量

```bash
export NAMESPACE=dynamo  # Your Dynamo namespace
export HF_TOKEN=your_hf_token  # Your HuggingFace token
```

---

## 第 2 步：创建 Secrets

### 创建 HuggingFace Token Secret

```bash
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=${HF_TOKEN} \
  -n ${NAMESPACE}
```

### 创建 MinIO 凭据 Secret

本示例中，我们使用 MinIO 的默认凭据。
你可以将凭据改为指向自己的 S3 兼容存储。

```bash
kubectl apply -f minio-secret.yaml -n ${NAMESPACE}
```

---

## 第 3 步：安装 MinIO

### 添加 MinIO Helm 仓库

```bash
helm repo add minio https://charts.min.io/
helm repo update
```

### 部署 MinIO

```bash
helm install minio minio/minio \
  --namespace ${NAMESPACE} \
  --set rootUser=minioadmin \
  --set rootPassword=minioadmin \
  --set mode=standalone \
  --set replicas=1 \
  --set persistence.enabled=true \
  --set persistence.size=10Gi \
  --set resources.requests.memory=512Mi \
  --set service.type=ClusterIP \
  --set consoleService.type=ClusterIP
```

### 验证 MinIO 安装

```bash
kubectl get pods -n ${NAMESPACE} | grep minio
kubectl get svc -n ${NAMESPACE} | grep minio
```

预期输出：
```text
minio-xxxx-xxxx   1/1     Running   0          1m
```

### （可选）访问 MinIO 控制台

```bash
kubectl port-forward svc/minio-console -n ${NAMESPACE} 9001:9001 9000:9000
```

在浏览器打开 http://localhost:9001：
- 用户名：`minioadmin`
- 密码：`minioadmin`

---

## 第 4 步：将 LoRA 适配器上传到 MinIO

使用提供的 Kubernetes Job 从 HuggingFace 下载视觉 LoRA 适配器并上传到 MinIO：

```bash
kubectl apply -f sync-lora-job.yaml -n ${NAMESPACE}
```

默认 job 同步的是 `Chhagan005/Chhagan-DocVL-Qwen3` —— 一个面向 Qwen3-VL-2B 的文档理解 LoRA。

### 监控 Job

```bash
# Watch job progress
kubectl get jobs -n ${NAMESPACE} -w

# Check job logs
kubectl logs job/sync-hf-lora-to-minio -n ${NAMESPACE} -f
```

等待 job 成功完成。

### 验证上传（可选）

```bash
# Port-forward MinIO API
kubectl port-forward svc/minio -n ${NAMESPACE} 9000:9000 &

# Check uploaded files
export AWS_ACCESS_KEY_ID=minioadmin
export AWS_SECRET_ACCESS_KEY=minioadmin
export AWS_ENDPOINT_URL=http://localhost:9000
aws s3 ls s3://my-loras/ --recursive
```

### 自定义 LoRA 适配器

要上传不同的 LoRA 适配器，编辑 `sync-lora-job.yaml`，修改 `MODEL_NAME` 环境变量：

```yaml
env:
- name: MODEL_NAME
  value: your-org/your-vision-lora-adapter
```

---

## 第 5 步：部署支持 LoRA 的多模态 vLLM

### 更新镜像（按需）

编辑 `agg_qwen_lora.yaml` 以使用你的容器镜像：

```bash
# Using yq to update the image
export FRAMEWORK_RUNTIME_IMAGE=your-registry/your-image:tag
yq '.spec.services[].extraPodSpec.mainContainer.image = env(FRAMEWORK_RUNTIME_IMAGE)' agg_qwen_lora.yaml > agg_qwen_lora_updated.yaml
```

### 部署支持 LoRA 的多模态 Graph

```bash
kubectl apply -f agg_qwen_lora.yaml -n ${NAMESPACE}
```

### 验证部署

```bash
# Check pods
kubectl get pods -n ${NAMESPACE}

# Watch worker logs
kubectl logs -f deployment/agg-qwen-multimodal-lora-vllmworker -n ${NAMESPACE}
```

等待 worker 显示 "Application startup complete"。

### 测试部署

```bash
# Port-forward the frontend
kubectl port-forward svc/agg-qwen-multimodal-lora-frontend -n ${NAMESPACE} 8000:8000 &

# List available models
curl http://localhost:8000/v1/models | jq .
```

---

## 第 6 步：使用 DynamoModel CRD

`lora-model.yaml` 演示了如何通过 DynamoModel 自定义资源注册 LoRA 适配器：

```bash
kubectl apply -f lora-model.yaml -n ${NAMESPACE}
```

这种方式提供了一种声明式的 LoRA 适配器管理方式。该模型 CRD 引用：
- **modelName**：`Chhagan005/Chhagan-DocVL-Qwen3`（适配器身份）
- **baseModelName**：`Qwen/Qwen3-VL-2B-Instruct`（基础 VLM）
- **source.uri**：`s3://my-loras/Chhagan005/Chhagan-DocVL-Qwen3`（MinIO 位置）

---

## 第 7 步：运行推理

### 使用 LoRA 适配器进行推理

```bash
curl -X POST http://localhost:8000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{
    "model": "Chhagan005/Chhagan-DocVL-Qwen3",
    "messages": [{"role": "user", "content": [
      {"type": "text", "text": "Describe this image in detail"},
      {"type": "image_url", "image_url": {"url": "https://vllm-public-assets.s3.us-west-2.amazonaws.com/vision_model_images/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg"}}
    ]}],
    "max_tokens": 300,
    "temperature": 0.0
  }' | jq .
```

### 使用基础模型进行推理

```bash
curl -X POST http://localhost:8000/v1/chat/completions -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-VL-2B-Instruct",
    "messages": [{"role": "user", "content": [
      {"type": "text", "text": "Describe this image in detail"},
      {"type": "image_url", "image_url": {"url": "https://vllm-public-assets.s3.us-west-2.amazonaws.com/vision_model_images/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg"}}
    ]}],
    "max_tokens": 300,
    "temperature": 0.0
  }' | jq .
```

---

## 配置参考

### 环境变量

| 变量 | 描述 | 默认值 |
|----------|-------------|---------|
| `DYN_REQUEST_PLANE` | 传输平面（多模态使用 TCP 以避免 NATS 1MB 限制） | `tcp` |
| `DYN_LORA_ENABLED` | 启用 LoRA 支持 | `true` |
| `DYN_LORA_PATH` | LoRA 文件本地缓存路径 | `/tmp/dynamo_loras_multimodal` |
| `DYN_SYSTEM_ENABLED` | 启用系统管理 API | `true` |
| `DYN_SYSTEM_PORT` | LoRA 管理 API 的端口 | `9090` |
| `AWS_ENDPOINT` | MinIO/S3 endpoint URL | `http://minio:9000` |
| `AWS_ACCESS_KEY_ID` | MinIO access key | 来自 secret |
| `AWS_SECRET_ACCESS_KEY` | MinIO secret key | 来自 secret |
| `AWS_REGION` | AWS region（S3 SDK 必需） | `us-east-1` |
| `AWS_ALLOW_HTTP` | 允许 HTTP 连接 | `true` |
| `BUCKET_NAME` | MinIO bucket 名称 | `my-loras` |

### vLLM 参数

| 参数 | 描述 |
|----------|-------------|
| `--enable-multimodal` | 启用多模态（视觉）支持 |
| `--enable-lora` | 启用 LoRA 适配器支持 |
| `--max-lora-rank` | 最大 LoRA rank（必须 ≥ adapter 的 rank） |
| `--max-loras` | 同时加载的 LoRA 最大数量 |
| `--gpu-memory-utilization` | 使用的 GPU 显存比例（默认 0.85） |
| `--max-model-len` | 最大序列长度（默认 8192） |
| `--max-num-batched-tokens` | 单批最大 token 数（默认 8192） |

---

## 清理

### 移除 vLLM 部署

```bash
kubectl delete -f agg_qwen_lora.yaml -n ${NAMESPACE}
```

### 移除 DynamoModel CRD

```bash
kubectl delete -f lora-model.yaml -n ${NAMESPACE}
```

### 移除同步 Job

```bash
kubectl delete -f sync-lora-job.yaml -n ${NAMESPACE}
```

### 移除 MinIO

```bash
helm uninstall minio -n ${NAMESPACE}
```

### 移除 Secret

```bash
kubectl delete -f minio-secret.yaml -n ${NAMESPACE}
kubectl delete secret hf-token-secret -n ${NAMESPACE}
```

---

## 故障排查

### LoRA 加载失败

1. **从 worker 检查 MinIO 连通性**：
   ```bash
   kubectl exec -it deployment/agg-qwen-multimodal-lora-vllmworker -n ${NAMESPACE} -- \
     curl http://minio:9000/minio/health/live
   ```

2. **确认 MinIO 中存在该 LoRA**：
   ```bash
   kubectl port-forward svc/minio -n ${NAMESPACE} 9000:9000 &
   aws --endpoint-url=http://localhost:9000 s3 ls s3://my-loras/ --recursive
   ```

3. **查看 worker 日志**：
   ```bash
   kubectl logs deployment/agg-qwen-multimodal-lora-vllmworker -n ${NAMESPACE}
   ```

4. **确认适配器兼容性**：确保该 LoRA 适配器是基于相同的基础模型架构（Qwen3-VL-2B）训练的，并且 `max-lora-rank`（默认 64）≥ 适配器的 rank。

### 同步 Job 失败

1. **查看 job 日志**：
   ```bash
   kubectl logs job/sync-hf-lora-to-minio -n ${NAMESPACE}
   ```

2. **核对 HuggingFace token**：
   ```bash
   kubectl get secret hf-token-secret -n ${NAMESPACE} -o yaml
   ```

3. **确认 MinIO 可达**：
   ```bash
   kubectl get svc minio -n ${NAMESPACE}
   ```

### 推理时 OOM

- Qwen VL 模型采用动态分辨率：一张 2560px 的图像可能产生 5000+ token
- 调小 `agg_qwen_lora.yaml` args 中的 `--max-model-len`
- 加上 `--mm-processor-kwargs '{"max_pixels": 1003520}'` 来限制图像分辨率
- 把 `--gpu-memory-utilization` 调低到 `0.80`

### MinIO 拒绝连接

- 确认 MinIO pod 正在运行：`kubectl get pods -n ${NAMESPACE} | grep minio`
- 检查 MinIO service：`kubectl get svc minio -n ${NAMESPACE}`
- 确认 `AWS_ENDPOINT` URL 与 service 名一致

## 延伸阅读

- [多模态 LoRA 启动指南](../../../launch/lora/multimodal/README.md) - 使用 shell 脚本本地启动
- [LLM LoRA 部署](../README.md) - 纯文本 LoRA 部署模式
- [Dynamo Kubernetes 指南](../../../../../../docs/kubernetes/README.md) - 平台搭建
- [安装指南](../../../../../../docs/kubernetes/installation-guide.md) - 平台安装
