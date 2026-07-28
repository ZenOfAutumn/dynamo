# 在 Kubernetes 上使用 MinIO 部署 LoRA

本指南说明如何在 Kubernetes 环境中，通过 S3 兼容存储后端部署启用 LoRA 的 vLLM 推理。

## 概述

该部署模式支持在 Kubernetes 环境中从 S3 兼容存储（MinIO）动态加载 LoRA 适配器：

## 前置条件

- 支持 GPU 的 Kubernetes 集群
- 已安装 Helm 3.x
- 已配置 `kubectl` 可访问集群
- 已安装 Dynamo Kubernetes Platform（[安装指南](../../../../../docs/kubernetes/installation-guide.md)）
- 用于下载 Base 与 LoRA 适配器的 HuggingFace token

## 本目录文件

| 文件 | 说明 |
|------|-------------|
| `agg_lora.yaml` | 启用 LoRA 的 vLLM DynamoGraphDeployment |
| `minio-secret.yaml` | MinIO 凭据的 Kubernetes secret |
| `sync-lora-job.yaml` | 从 HuggingFace 下载 LoRA 并上传到 MinIO 的 Job |
| `lora-model.yaml` | 用于注册 LoRA 适配器的 DynamoModel CRD |

---

## 步骤 1：设置环境变量

```bash
export NAMESPACE=dynamo  # Your Dynamo namespace
export HF_TOKEN=your_hf_token  # Your HuggingFace token
```

---

## 步骤 2：创建 Secret

### 创建 HuggingFace Token Secret

```bash
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=${HF_TOKEN} \
  -n ${NAMESPACE}
```

### 创建 MinIO 凭据 Secret

本示例使用 MinIO 默认凭据。
你可以将凭据修改为指向自己的 S3 兼容存储。

```bash
kubectl apply -f minio-secret.yaml -n ${NAMESPACE}
```

---

## 步骤 3：安装 MinIO

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
```
minio-xxxx-xxxx   1/1     Running   0          1m
```

### （可选）访问 MinIO 控制台

```bash
kubectl port-forward svc/minio-console -n ${NAMESPACE} 9001:9001 9000:9000
```

在浏览器中打开 http://localhost:9001：
- 用户名：`minioadmin`
- 密码：`minioadmin`

---

## 步骤 4：将 LoRA 适配器上传到 MinIO

使用提供的 Kubernetes Job 从 HuggingFace 下载 LoRA 适配器并上传到 MinIO：

```bash
kubectl apply -f sync-lora-job.yaml -n ${NAMESPACE}
```

### 监控 Job

```bash
# Watch job progress
kubectl get jobs -n ${NAMESPACE} -w

# Check job logs
kubectl logs job/sync-hf-lora-to-minio -n ${NAMESPACE} -f
```

等待 Job 成功完成。

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

要上传不同的 LoRA 适配器，编辑 `sync-lora-job.yaml` 并修改 `MODEL_NAME` 环境变量：

```yaml
env:
- name: MODEL_NAME
  value: your-org/your-lora-adapter
```

---

## 步骤 5：部署支持 LoRA 的 vLLM

### 更新镜像（如有需要）

编辑 `agg_lora.yaml` 使用你自己的容器镜像：

```bash
# Using yq to update the image
export FRAMEWORK_RUNTIME_IMAGE=your-registry/your-image:tag
yq '.spec.services.[].extraPodSpec.mainContainer.image = env(FRAMEWORK_RUNTIME_IMAGE)' agg_lora.yaml > agg_lora_updated.yaml
```

### 部署启用 LoRA 的 vLLM 图

```bash
kubectl apply -f agg_lora.yaml -n ${NAMESPACE}
```

### 验证部署

```bash
# Check pods
kubectl get pods -n ${NAMESPACE}

# Watch worker logs
kubectl logs -f deployment/vllm-agg-lora-vllmdecode-worker -n ${NAMESPACE}
```

等待 worker 输出 "Application startup complete"。


## 步骤 6：使用 DynamoModel CRD

`lora-model.yaml` 演示了如何使用 DynamoModel 自定义资源注册 LoRA 适配器：

```bash
kubectl apply -f lora-model.yaml -n ${NAMESPACE}
```

这提供了一种声明式管理集群中 LoRA 适配器的方式。

---

## 配置参考

### 环境变量

| 变量 | 说明 | 默认值 |
|----------|-------------|---------|
| `AWS_ENDPOINT` | MinIO/S3 端点 URL | `http://minio:9000` |
| `AWS_ACCESS_KEY_ID` | MinIO access key | 来自 secret |
| `AWS_SECRET_ACCESS_KEY` | MinIO secret key | 来自 secret |
| `AWS_REGION` | AWS region（S3 SDK 需要） | `us-east-1` |
| `AWS_ALLOW_HTTP` | 允许 HTTP 连接 | `true` |
| `DYN_LORA_ENABLED` | 启用 LoRA 支持 | `true` |
| `DYN_LORA_PATH` | LoRA 文件本地缓存路径 | `/tmp/dynamo_loras_minio` |
| `BUCKET_NAME` | MinIO bucket 名 | `my-loras` |

### vLLM LoRA 参数

| 参数 | 说明 |
|----------|-------------|
| `--enable-lora` | 启用 LoRA 适配器支持 |
| `--max-lora-rank` | LoRA 最大 rank（必须 >= LoRA 实际 rank） |
| `--max-loras` | 同时加载的最大 LoRA 数量 |

---

## 清理

### 移除 vLLM 部署

```bash
kubectl delete -f agg_lora.yaml -n ${NAMESPACE}
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
   kubectl exec -it deployment/vllm-agg-lora-vllmdecode-worker -n ${NAMESPACE} -- \
     curl http://minio:9000/minio/health/live
   ```

2. **确认 LoRA 在 MinIO 中存在**：
   ```bash
   kubectl port-forward svc/minio -n ${NAMESPACE} 9000:9000 &
   aws --endpoint-url=http://localhost:9000 s3 ls s3://my-loras/ --recursive
   ```

3. **查看 worker 日志**：
   ```bash
   kubectl logs deployment/vllm-agg-lora-vllmdecode-worker -n ${NAMESPACE}
   ```

### Sync Job 失败

1. **查看 job 日志**：
   ```bash
   kubectl logs job/sync-hf-lora-to-minio -n ${NAMESPACE}
   ```

2. **验证 HuggingFace token**：
   ```bash
   kubectl get secret hf-token-secret -n ${NAMESPACE} -o yaml
   ```

3. **检查 MinIO 是否可访问**：
   ```bash
   kubectl get svc minio -n ${NAMESPACE}
   ```

### MinIO 连接被拒绝

- 确认 MinIO pod 正在运行：`kubectl get pods -n ${NAMESPACE} | grep minio`
- 检查 MinIO 服务：`kubectl get svc minio -n ${NAMESPACE}`
- 验证 `AWS_ENDPOINT` URL 与服务名一致

## 延伸阅读

- [vLLM 部署指南](../README.md) - 其他部署模式
- [Dynamo Kubernetes 指南](../../../../../docs/kubernetes/README.md) - 平台搭建
- [安装指南](../../../../../docs/kubernetes/installation-guide.md) - 平台安装
