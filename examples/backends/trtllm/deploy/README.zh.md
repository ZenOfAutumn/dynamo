# TensorRT-LLM Kubernetes 部署配置

本目录包含使用 **DynamoGraphDeployment** 资源部署 TensorRT-LLM 推理图的 Kubernetes Custom Resource Definition（CRD）模板。

## 可用的部署模式

### 1. **聚合部署**（`agg.yaml`）
基础部署模式：包含 frontend 与单个 worker。

**架构：**
- `Frontend`：兼容 OpenAI 的 API 服务器（KV router 模式禁用）
- `TRTLLMWorker`：同时处理 prefill（预填充）与 decode（解码）的单个 worker

### 2. **聚合 + Router 部署**（`agg_router.yaml`）
带 KV 缓存路由能力的增强聚合部署。

**架构：**
- `Frontend`：兼容 OpenAI 的 API 服务器（KV router 模式启用）
- `TRTLLMWorker`：同时处理 prefill 与 decode 的多个 worker（2 副本以做负载均衡）

### 3. **解耦部署**（`disagg.yaml`）
高性能部署：prefill 与 decode worker 相互分离。

**架构：**
- `Frontend`：协调各 worker 的 HTTP API 服务器
- `decode`：仅负责 decode 的专用 worker
- `prefill`：仅负责 prefill 的专用 worker

### 4. **解耦 + Router 部署**（`disagg_router.yaml`）
带 KV 缓存路由能力的高级解耦部署。

**架构：**
- `Frontend`：HTTP API 服务器（KV router 模式启用）
- `decode`：仅负责 decode 的专用 worker
- `prefill`：仅负责 prefill 的专用 worker（2 副本以做负载均衡）

### 5. **聚合 + 自定义 Config 部署**（`agg-with-config.yaml`）
带自定义配置的聚合部署。

**架构：**
- `nvidia-config`：包含自定义 trtllm 配置的 ConfigMap
- `Frontend`：兼容 OpenAI 的 API 服务器（KV router 模式禁用）
- `TRTLLMWorker`：同时处理 prefill 与 decode 的单个 worker，自定义配置从 ConfigMap 挂载

### 6. **解耦 + Planner 部署**（`disagg_planner.yaml`）
带基于 SLA 自动扩缩的高级解耦部署。

**架构：**
- `Frontend`：协调各 worker 的 HTTP API 服务器
- `Planner`：基于 SLA 的调度器（planner），监控性能并自动扩缩 worker
- `Prometheus`：指标采集与监控
- `decode`：仅负责 decode 的专用 worker
- `prefill`：仅负责 prefill 的专用 worker

> [!NOTE]
> 该部署需要先完成预部署 profiling。详见 [Pre-Deployment Profiling](../../../../docs/components/profiler/profiler-guide.md)。

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

### 关键配置选项

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
    image: my-registry/tensorrtllm-runtime:my-tag
    workingDir: /workspace/examples/backends/trtllm
    args:
      - "python3"
      - "-m"
      - "dynamo.trtllm"
      # Model-specific arguments
```

## 前置条件

在使用这些模板之前，请确保：

1. **已安装 Dynamo Kubernetes 平台** —— 参见 [Quickstart Guide](../../../../docs/kubernetes/README.md)
2. **支持 GPU 的 Kubernetes 集群**
3. **可访问的容器 Registry**，用于 TensorRT-LLM runtime 镜像
4. **HuggingFace token Secret**（被 `envFromSecret: hf-token-secret` 引用）

### 容器镜像

部署文件目前需要访问 `my-registry/tensorrtllm-runtime`。如果你没有访问权限，可以构建并推送自己的镜像：

```bash
python container/render.py --framework=trtllm --output-short-filename --cuda-version=13.1
docker build -f container/rendered.Dockerfile .
# Tag and push to your container registry
# Update the image references in the YAML files
```

**注意：** TensorRT-LLM 使用 git-lfs，需提前安装：
```bash
apt-get update && apt-get -y install git git-lfs
```

对于 ARM 机器，使用：
```bash
python container/render.py --framework=vllm --platform arm64 --output-short-filename
docker build -f container/rendered.Dockerfile .
```

## 用法

### 1. 选择模板
按需求选择部署模式：
- 简单测试用 `agg.yaml`
- 启用 KV 缓存路由与负载均衡的生产部署用 `agg_router.yaml`
- 追求最高性能、worker 分离的用 `disagg.yaml`
- 同时启用 KV 缓存路由与解耦的高性能部署用 `disagg_router.yaml`

### 2. 自定义配置
按你的环境编辑模板：

```yaml
# Update image registry and tag
image: my-registry/tensorrtllm-runtime:my-tag

# Configure your model and deployment settings
args:
  - "python3"
  - "-m"
  - "dynamo.trtllm"
  # Add your model-specific arguments
```

### 3. 部署

部署 yaml 的方式参见 [Create Deployment Guide](../../../../docs/kubernetes/deployment/create-deployment.md)。

首先，为 HuggingFace token 创建 Secret：
```bash
export HF_TOKEN=your_hf_token
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=${HF_TOKEN} \
  -n ${NAMESPACE}
```

然后用部署文件部署模型。

将你在 Dynamo Kubernetes 平台安装中使用的 NAMESPACE 导出：

```bash
cd dynamo/examples/backends/trtllm/deploy
export DEPLOYMENT_FILE=agg.yaml
kubectl apply -f $DEPLOYMENT_FILE -n $NAMESPACE
```

### 4. 为 TensorRT-LLM 使用自定义 Dynamo Frameworks 镜像

可使用 yq 更新部署文件以使用自定义的 dynamo frameworks 镜像：

```bash
export DEPLOYMENT_FILE=agg.yaml
export FRAMEWORK_RUNTIME_IMAGE=<trtllm-image>

yq '.spec.services.[].extraPodSpec.mainContainer.image = env(FRAMEWORK_RUNTIME_IMAGE)' $DEPLOYMENT_FILE  > $DEPLOYMENT_FILE.generated
kubectl apply -f $DEPLOYMENT_FILE.generated -n $NAMESPACE
```

### 5. 端口转发

部署完成后，转发 frontend 服务以访问 API：

```bash
kubectl port-forward deployment/trtllm-v1-disagg-frontend-<pod-uuid-info> 8000:8000
```

## 配置选项

### 环境变量

要修改 `DYN_LOG` 级别，编辑 yaml 文件加入：

```yaml
...
spec:
  envs:
    - name: DYN_LOG
      value: "debug" # or other log levels
  ...
```

### TensorRT-LLM Worker 配置

TensorRT-LLM worker 通过部署 YAML 中的命令行参数配置。关键配置点包括：

- **KV Cache 传输**：解耦服务下可在 UCX（默认）与 NIXL 之间选择
- **请求迁移**：通过 `--migration-limit` 启用优雅的失败处理

## 测试部署

发送一个测试请求来验证部署。详细说明见 [客户端章节](../../../../docs/backends/vllm/README.md#client)。

**注意：** 多节点部署中，请把请求发送到运行 `python3 -m dynamo.frontend <args>` 的节点。

## 模型配置

部署模板支持各种 TensorRT-LLM 模型与配置。可在 YAML 中 worker 配置部分自定义模型相关参数。

## 监控与健康检查

- **Frontend 健康检查端点**：`http://<frontend-service>:8000/health`
- **Worker 健康检查端点**：`http://<worker-service>:9090/health`
- **Liveness 探针**：每 5 秒检查进程健康
- **Readiness 探针**：以可配置的延迟检查服务就绪

## KV Cache 传输方式

TensorRT-LLM 在解耦服务下支持两种 KV cache 传输方式：

- **UCX**（默认）：标准的 KV cache 传输方式
- **NIXL**（实验）：备选传输方式

详细配置参见 [KV cache transfer guide](../../../../docs/backends/trtllm/trtllm-kv-cache-transfer.md)。

## 请求迁移

可启用[请求迁移](../../../../docs/fault-tolerance/request-migration.md)来优雅处理 worker 故障，方法是在 worker 配置中加入迁移上限参数：

```yaml
args:
  - "python3"
  - "-m"
  - "dynamo.trtllm"
  - "--migration-limit"
  - "3"
```

## 基准测试

要使用 AIPerf 对部署做基准测试，参见这个工具脚本：[perf.sh](../../../../benchmarks/llm/perf.sh)

根据你的部署配置 `model` 名称与 `host`。

## 进一步阅读

- **部署指南**：[Creating Kubernetes Deployments](../../../../docs/kubernetes/deployment/create-deployment.md)
- **快速上手**：[Deployment Quickstart](../../../../docs/kubernetes/README.md)
- **平台搭建**：[Dynamo Kubernetes Platform Installation](../../../../docs/kubernetes/installation-guide.md)
- **示例**：[Deployment Examples](../../../../docs/getting-started/examples.md)
- **架构文档**：[Disaggregated Serving](../../../../docs/design-docs/disagg-serving.md), [KV-Aware Routing](../../../../docs/components/router/README.md)
- **多节点部署**：[Multinode Examples](../../../../docs/backends/trtllm/multinode/trtllm-multinode-examples.md)
- **推测解码**：[Llama 4 + Eagle Guide](../../../../docs/backends/trtllm/trtllm-llama4-plus-eagle.md)
- **Kubernetes CRDs**：[Custom Resources Documentation](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

## 故障排查

常见问题与解决方案：

1. **Pod 启动失败**：检查镜像 registry 访问权限与 HuggingFace token Secret
2. **未分配 GPU**：确认集群存在 GPU 节点且资源限额正确
3. **健康检查失败**：检查模型加载日志并增大 `initialDelaySeconds`
4. **OOM**：增大内存限额或减小模型 batch size
5. **端口转发问题**：在 port-forward 命令中使用正确的 pod UUID
6. **Git LFS 问题**：构建容器之前确保已安装 git-lfs
7. **ARM 部署**：在 ARM 机器上构建时使用 `--platform linux/arm64`

如需更多支持，参见 [部署故障排查指南](../../../../docs/kubernetes/README.md)。
