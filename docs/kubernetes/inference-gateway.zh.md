---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Inference Gateway (GAIE)
---

## 与 Dynamo 配合的 Inference Gateway 搭建

# Inference Gateway（GAIE）

将 Dynamo 与 Gateway API Inference Extension 集成，在网关层实现智能 KV 感知请求路由（router）。

## 特性

- EPP 默认的 KV 路由方式不是 token 感知的，因为它不会对 prompt 做 tokenize。而 Dynamo 插件使用一种 token 感知的 KV 算法。它使用 dynamo router，通过在内联运行你的模型 tokenizer 实现 KV 路由。EPP 插件配置位于 [`helm/dynamo-gaie/epp-config-dynamo.yaml`](https://github.com/ai-dynamo/dynamo/blob/main/deploy/inference-gateway/standalone/helm/dynamo-gaie/epp-config-dynamo.yaml)，遵循本仓库内已有的 GAIE/EPP 配置布局。

- Dynamo 与 Inference Gateway 的集成同时支持聚合（Aggregated）与解耦（Disaggregated）服务。只有当 EPP 配置定义了 `prefill` profile 且存在可用的 prefill worker 时，请求才会走解耦路由。独立版的 [`epp-config-dynamo.yaml`](https://github.com/ai-dynamo/dynamo/blob/main/deploy/inference-gateway/standalone/helm/dynamo-gaie/epp-config-dynamo.yaml) 当前只定义了 `decode` profile，而 recipe 示例使用 `recipes/llama-3-70b/vllm/agg/gaie/` 与 `recipes/llama-3-70b/vllm/disagg-single-node/gaie/` 下分开的聚合与解耦配置。除非设置 `DYN_ENFORCE_DISAGG=true`，否则没有 `prefill` profile 或没有 prefill worker 的部署会回退到聚合服务。

- GAIE 集成支持 Data Parallelism。

- 如果你想使用 LoRA，请在不带 Inference Gateway 的情况下部署 Dynamo。

- 当前这些搭建仅在 kGateway 实现的 Inference Gateway 上做过测试。

## 先决条件

- Kubernetes 集群（cluster）且配置好 kubectl
- worker 节点（node）安装好 NVIDIA GPU 驱动

## 安装步骤

### 1. 安装 Dynamo 平台 ###

参见 [Quickstart Guide](./README.md) 安装 Dynamo Kubernetes Platform。
如果你是从源码树而非 release chart 安装，请按 [Path B: Custom Build from Source](./installation-guide.md#path-b-custom-build-from-source) 操作，并在 `helm install` 之前运行 `helm dep build ./platform/`，确保 vendored 子 chart 与本地 chart 内容一致。

### 2. 部署 Inference Gateway ###

首先部署一个 inference gateway 服务。本示例中我们安装基于 `kgateway` 的 gateway 实现。

```bash
cd deploy/inference-gateway
export NAMESPACE=my-model # You can put the inference gateway into another namespace and then adjust your http-route.yaml
./scripts/install_gaie_crd_kgateway.sh
```
**注意**：`config/manifests/gateway/agentgateway/gateway.yaml` 中的清单使用 `gatewayClassName: agentgateway`，但 kGateway 的 helm chart 创建的 GatewayClass 名为 `kgateway`。脚本中的 patch 命令会修复这一不一致。

#### f. 验证 Gateway 正在运行

```bash
kubectl get gateway inference-gateway

# Sample output
# NAME                CLASS      ADDRESS   PROGRAMMED   AGE
# inference-gateway   kgateway             True         1m
```


### 3. 准备 secret ###

如有需要，不要忘记 docker registry secret。

```bash
kubectl create secret docker-registry docker-imagepullsecret \
  --docker-server=$DOCKER_SERVER \
  --docker-username=$DOCKER_USERNAME \
  --docker-password=$DOCKER_PASSWORD \
  --namespace=$NAMESPACE
```

不要忘记包含 HuggingFace token。

```bash
export HF_TOKEN=your_hf_token
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=${HF_TOKEN} \
  -n ${NAMESPACE}
```

### 4. 构建 EPP 镜像（可选）

你可以使用提供的 Dynamo FrontEnd 镜像作为 EPP 镜像，或按下面步骤构建你自己的 Dynamo EPP 自定义镜像。

```bash
# export env vars
export DOCKER_SERVER=ghcr.io/nvidia/dynamo	# Container registry
export IMAGE_TAG=YOUR-TAG # Or auto from git tag
cd deploy/inference-gateway/epp
make all # Do everything in one command
# or make all-push to also push


# Or step-by-step
make dynamo-lib # Build Dynamo library and copy to project
make image-load # Build Docker image and load locally
make image-push # Build and push to registry
make info # Check image tag
```

#### 一体化目标

| Target | 说明 |
|--------|-------------|
| `make dynamo-lib` | 构建 Dynamo 静态库并拷贝到项目中 |
| `make all` | 构建 Dynamo 库 + Docker 镜像 + 本地加载 |
| `make all-push` | 构建 Dynamo 库 + Docker 镜像 + 推送到 registry |

### 5. 部署

我们推荐把 Inference Gateway 的 Endpoint Picker 作为 Dynamo operator 管理的组件部署。也可以将其作为独立 pod 部署。
注意，将 Dynamo 与 Inference Gateway Extension 一起部署时，每个 worker 都必须把 FrontEnd 作为 sidecar 运行。

#### 5.a. 作为 DGD 组件部署（推荐）

下面提供一个 Qwen vLLM 的示例。
你需要部署 Dynamo Graph 与 HttpRoute service。
对于 HttpRoute service，请确保如下指定你的 gateway（即 kGateway 部署的位置）所在的命名空间：
```bash
  parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: inference-gateway
      namespace: my-model # the namespace where your gateway is deployed.
```

```bash
cd <dynamo-source-root>
# kubectl get httproutes -n my-model # Make sure you do not have an incompatible HttpRoute running, delete if so.
# Choose disagg or agg example
kubectl apply -f examples/backends/vllm/deploy/gaie/disagg.yaml -n my-model
# or
kubectl apply -f examples/backends/vllm/deploy/gaie/agg.yaml -n my-model
# make sure to apply the route
kubectl apply -f examples/backends/vllm/deploy/gaie/http-route.yaml -n my-model
```

其他模型的示例可在 recipes 文件夹找到。

```bash
# Deploy PVC, having first Update `storageClassName` in recipes/llama-3-70b/model-cache/model-cache.yaml to match your cluster before deploying
kubectl apply -f recipes/llama-3-70b/model-cache/model-cache.yaml  -n ${NAMESPACE}
kubectl apply -f recipes/llama-3-70b/model-cache/model-download.yaml  -n ${NAMESPACE}
```
我们在 `recipes/llama-3-70b/vllm/agg/gaie/` 提供聚合的 llama-3-70b vLLM 示例，在 `recipes/llama-3-70b/vllm/disagg-single-node/gaie/` 提供解耦服务示例。
注意聚合服务时需要在 epp 配置中关闭 DYN_ENFORCE_DISAGG。
```bash
  - name: DYN_ENFORCE_DISAGG
    value: "false"
```
请在下方命令中使用合适的目录。

```bash
# Deploy your Dynamo Graph.

# agg
kubectl apply -f recipes/llama-3-70b/vllm/agg/gaie/deploy.yaml -n ${NAMESPACE}
# Deploy the GAIE http-route CR. Adjust parentRefs.namespace in this file first to point where your gateway is.
kubectl apply -f recipes/llama-3-70b/vllm/agg/gaie/http-route.yaml -n ${NAMESPACE}

# or disagg
kubectl apply -f recipes/llama-3-70b/vllm/disagg-single-node/gaie/deploy.yaml  -n ${NAMESPACE}
kubectl apply -f recipes/llama-3-70b/vllm/disagg-single-node/gaie/http-route.yaml -n ${NAMESPACE}
```

- 使用 GAIE 时，FrontEnd 不再选择 worker。路由由 EPP 决定。
- FrontEnd 必须以 `--router-mode direct` 运行，以便遵循 EPP 通过请求头传来的路由决策。
- 在 worker service 上使用 `frontendSidecar` 字段，让 operator 自动注入一个完全配置好的 frontend sidecar 容器，包含全部必需的 Dynamo 环境变量、探针与端口：

```yaml
frontendSidecar:
  image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1
  args:
    - --router-mode
    - direct
  envFromSecret: hf-token-secret
```

- 预先选择好的 worker（解耦服务时是 decode 与 prefill）通过请求头传递。
- `--router-mode direct` flag 确保路由遵循该选择。

**Startup Probe Timeout：** EPP 默认的 startup probe 超时是 30 分钟（10s × 180 次失败）。
如果你的模型加载时间更长，请增大 EPP `startupProbe` 中的 `failureThreshold`。例如允许 60 分钟启动：

```yaml
extraPodSpec:
  mainContainer:
    startupProbe:
      failureThreshold: 360  # 10s × 360 = 60 minutes
```

**Gateway 命名空间**
注意，这里假设你的 gateway 安装到了 `NAMESPACE=my-model`（示例默认值）。
如果你装到了不同命名空间，需要相应调整 `http-route.yaml` 中的 HttpRoute 项。


#### 5.b. 作为独立 pod 部署

我们不推荐这种方式，但下面提供一些提示。

##### 5.b.1 部署你的模型 ###

##### 5.b.2 安装 Dynamo GIE helm chart ###

```bash
cd deploy/inference-gateway/standalone

# Export the EPP image - use the Dynamo FrontEnd image or build your own EPP image (see section 4)
export EPP_IMAGE=<the-epp-image>
```
为你的模型创建一个类似 vllm_agg_qwen.yaml 的模型配置文件。

```bash
helm upgrade --install dynamo-gaie ./helm/dynamo-gaie -n my-model -f ./vllm_agg_qwen.yaml --set-string extension.image=$EPP_IMAGE
```

默认使用 Kubernetes 服务发现机制。如果你倾向使用 etcd，请使用下面的 `--set epp.dynamo.useEtcd=true` flag。

```bash
helm upgrade --install dynamo-gaie ./helm/dynamo-gaie -n my-model -f ./vllm_agg_qwen.yaml --set-string extension.image=$EPP_IMAGE --set epp.dynamo.useEtcd=true
```

主要配置包括：

- 用于 Qwen 模型的 InferenceModel 资源
- 用于 inference gateway 的 service
- 必需的 RBAC role 与 binding
- RBAC 权限
- dynamoGraphDeploymentName —— 你模型部署所在的 Dynamo Graph 名称


**配置**
你可以通过设置环境变量来配置插件 —— operator 管理的安装是在 DGD 的 EPP 组件中设置，独立安装是在你的 [values.yaml](https://github.com/ai-dynamo/dynamo/blob/main/deploy/inference-gateway/standalone/helm/dynamo-gaie/values.yaml) 中设置。

路由配置常用变量：

**启用 KV 感知路由（最精确）**

KV 感知路由使用来自 worker 的实时 KV 缓存（KV cache）block 事件，使 EPP 能把请求路由到前缀缓存命中最佳的 worker。要启用（默认）：

1. **Worker —— 启用前缀缓存与 KV 事件发布。** 每个 worker 必须把 KV 缓存事件发布到事件平面（NATS/ZMQ），这样 EPP 的路由就能跟踪每个 worker 的缓存状态。
   - **vLLM：** 传 `--enable-prefix-caching` 与 `--kv-events-config '{"enable_kv_cache_events":true}'`。
   - **SGLang：** 传带合适端点的 `--kv-events-config`。
   - **TRT-LLM：** 传 `--publish-events-and-metrics`。
2. **EPP —— 保持 `DYN_USE_KV_EVENTS` 为默认（`true`）。** EPP 通过事件平面（NATS/ZMQ）订阅 worker 的 KV 事件，并据此进行前缀重叠打分。
3. **block size —— 必须一致。** 所有 worker 的 `--block-size` 必须与 EPP 上的 `DYN_KV_CACHE_BLOCK_SIZE` 相同（默认：128）。block size 不一致会导致 block hash 计算错误。

**关闭 KV 感知路由**

要让 EPP 不再监听 KV 事件（例如 worker 上关闭了前缀缓存，或希望简单的负载均衡路由）：

1. **EPP：** 设置 `DYN_USE_KV_EVENTS=false`。路由会回退到近似模式（路由决策在本地按 TTL 衰减跟踪，而非来自 worker 的实时 KV 事件）。
2. **Worker：** 传 `--no-enable-prefix-caching` 完全关闭前缀缓存。无前缀缓存时无论其他 flag 如何都不会产生 KV 事件。
3. **可选**：在 EPP 上设置 `DYN_OVERLAP_SCORE_WEIGHT=0`，完全跳过前缀重叠打分，让路由仅基于负载选 worker。

- 设置 `DYN_BUSY_THRESHOLD` 配置 worker 在被路由跳过之前能“多满”的上界（通常由 kv_active_blocks 或其他负载指标推得）。如果选中的 worker 超过该值，则回退到下一个最佳候选。默认值为负数，即不启用。
- 设置 `DYN_ENFORCE_DISAGG=true`（默认 `false`）控制 prefill worker 不可用时的逐请求行为：
  - **`true`（推荐用于解耦服务）：** 没有 prefill worker 时请求直接报错。需要解耦服务且不接受聚合回退时使用。
  - **`false`（默认）：** 没有 prefill worker 时请求优雅回退到聚合模式（跳过 prefill，直接路由到 decode）。当稍后出现 prefill worker 时，后续请求会自动改为解耦路由。
- 设置 `DYN_OVERLAP_SCORE_WEIGHT` 控制评分中 token 重叠（预测的 KV 缓存命中）与其他因素（负载、历史命中率）的权重。权重越大越倾向于复用已有相似前缀的 worker。（默认 1）
- 设置 `DYN_ROUTER_TEMPERATURE` 在结合分数时软化或锐化选择曲线。低温让路由确定性地选最佳候选；高温让低分 worker 也有更多机会（探索）。
- `DYN_ROUTER_TEMPERATURE` —— 通过 softmax 进行 worker 采样的温度（默认 0.0）
- `DYN_ROUTER_REPLICA_SYNC` —— 启用副本同步（默认 false）
- `DYN_ROUTER_TRACK_ACTIVE_BLOCKS` —— 跟踪活跃 block（默认 true）
- `DYN_ROUTER_TRACK_OUTPUT_BLOCKS` —— 在生成期间跟踪输出 block（默认 false）
- 详情见 [KV cache routing design](../design-docs/router-design.md)。

仅独立安装：
- 如有需要，覆盖 `DYN_NAMESPACE` 环境变量以匹配模型的 dynamo 命名空间。

**Service Mesh 集成（Istio）**

在 Istio 等 service mesh 下运行时，mesh sidecar 代理可能与 EPP 自身的 TLS 服务冲突，导致连接失败（双层 TLS）。要避免这一点，需要通过 Istio `DestinationRule` 告诉 mesh 如何连接到 EPP service。

Dynamo operator 可以自动生成该 DestinationRule。安装或升级 Dynamo platform Helm chart 时通过设置 `dynamo.serviceMesh` 参数启用：

```bash
helm install dynamo deploy/helm/charts/platform \
  --set dynamo.serviceMesh.enabled=true
```

或在自定义 values 文件中等价地：

```yaml
dynamo:
  serviceMesh:
    enabled: true
    provider: "istio"
    istio:
      tlsMode: "SIMPLE"
      insecureSkipVerify: true
```

**Helm 参数**

| 参数 | 类型 | 默认 | 说明 |
|-----------|------|---------|-------------|
| `dynamo.serviceMesh.enabled` | bool | `false` | 为 EPP service 启用自动 DestinationRule 生成。 |
| `dynamo.serviceMesh.provider` | string | `"istio"` | service mesh provider。仅支持 `"istio"`。 |
| `dynamo.serviceMesh.istio.tlsMode` | string | `"SIMPLE"` | DestinationRule 的 TLS 模式。可选：`DISABLE`、`SIMPLE`、`MUTUAL`、`ISTIO_MUTUAL`。 |
| `dynamo.serviceMesh.istio.insecureSkipVerify` | bool | `true` | 跳过 TLS 证书验证。EPP 使用自签名证书时（默认）设为 `true`。 |

> [!NOTE]
> 启用该特性前，必须在集群上安装 Istio CRD（`networking.istio.io`）。operator 在启动时检测 Istio 可用性 —— 即便 `serviceMesh.enabled` 为 `true`，缺少 CRD 时 DestinationRule 调谐也会被跳过。

启用后，operator 会为每个 EPP service 生成等价的 `DestinationRule`：

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: <epp-service-name>
spec:
  host: <epp-service-name>.<namespace>.svc.cluster.local
  trafficPolicy:
    tls:
      mode: SIMPLE
      insecureSkipVerify: true
```

如果你**不**使用 Dynamo operator 的 Helm chart，则必须为每个 EPP service 手动创建上述 `DestinationRule`。否则 Istio 默认的 mTLS 策略会与 EPP 的 gRPC TLS 端点冲突。

### 6. 验证安装 ###

检查所有资源都已正确部署：

```bash
kubectl get inferencepool
kubectl get httproute
kubectl get service
kubectl get gateway
```

示例输出：

```bash
# kubectl get inferencepool
NAME        AGE
qwen-pool   33m

# kubectl get httproute
NAME        HOSTNAMES   AGE
qwen-route               33m
```

### 7. 使用 ###

Inference Gateway 提供用于模型推理的 HTTP 端点。

#### 1：为你的 k8s 集群获取 gateway URL ####

a. 在 minikube 中测试：
使用 minikube tunnel 把 gateway 暴露给宿主机。这需要宿主机的 `sudo` 权限。也可以像 (b) 那样用 port-forward 暴露。

```bash
# in first terminal
ps aux | grep "minikube tunnel" | grep -v grep # make sure minikube tunnel is not already running.
minikube tunnel # start the tunnel

# in second terminal where you want to send inference requests
GATEWAY_URL=$(kubectl get svc inference-gateway -n my-model -o jsonpath='{.spec.clusterIP}') && echo $GATEWAY_URL
```

b. 在集群中测试：

使用 port-forward 暴露 gateway：

```bash
# in first terminal
kubectl port-forward svc/inference-gateway 8000:80 -n ${NAMESPACE} # for NAMESPACE put wherever you installed the gateway i.e. kgateway-system or my-model

# in second terminal where you want to send inference requests
GATEWAY_URL=http://localhost:8000
```

#### 2：检查部署到 inference gateway 的模型 ####

a. 查询模型：

```bash
# in the second terminal where you GATEWAY_URL is set
curl $GATEWAY_URL/v1/models | jq .
# or if you added the host name to http route:
curl -H "Host: llama3-70b-disagg.example.com" $GATEWAY_URL/v1/models | jq .
```

示例输出：

```json
{
  "data": [
    {
      "created": 1753768323,
      "id": "Qwen/Qwen3-0.6B",
      "object": "object",
      "owned_by": "nvidia"
    }
  ],
  "object": "list"
}
```

b. 向 gateway 发送推理请求：

```bash
MODEL_NAME="Qwen/Qwen3-0.6B"
curl $GATEWAY_URL/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
      "model": "'"${MODEL_NAME}"'",
      "messages": [
      {
          "role": "user",
          "content": "In the heart of Eldoria, an ancient land of boundless magic and mysterious creatures, lies the long-forgotten city of Aeloria. Once a beacon of knowledge and power, Aeloria was buried beneath the shifting sands of time, lost to the world for centuries. You are an intrepid explorer, known for your unparalleled curiosity and courage, who has stumbled upon an ancient map hinting at ests that Aeloria holds a secret so profound that it has the potential to reshape the very fabric of reality. Your journey will take you through treacherous deserts, enchanted forests, and across perilous mountain ranges. Your Task: Character Background: Develop a detailed background for your character. Describe their motivations for seeking out Aeloria, their skills and weaknesses, and any personal connections to the ancient city or its legends. Are they driven by a quest for knowledge, a search for lost familt clue is hidden."
      }
      ],
      "stream":false,
      "max_tokens": 30,
      "temperature": 0.0
    }'
```
或

```bash
MODEL_NAME="RedHatAI/Llama-3.3-70B-Instruct-FP8-dynamic"
curl -H "Host: llama3-70b-disagg.example.com" http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
      "model": "'"${MODEL_NAME}"'",
      "messages": [
      {
          "role": "user",
          "content": "In the heart of Eldoria, an ancient land of boundless magic and mysterious creatures, lies the long-forgotten city of Aeloria. Once a beacon of knowledge and power, Aeloria was buried beneath the shifting sands of time, lost to the world for centuries. You are an intrepid explorer, known for your unparalleled curiosity and courage, who has stumbled upon an ancient map hinting at ests that Aeloria holds a secret so profound that it has the potential to reshape the very fabric of reality. Your journey will take you through treacherous deserts, enchanted forests, and across perilous mountain ranges. Your Task: Character Background: Develop a detailed background for your character. Describe their motivations for seeking out Aeloria, their skills and weaknesses, and any personal connections to the ancient city or its legends. Are they driven by a quest for knowledge, a search for lost familt clue is hidden."
      }
      ],
      "stream":false,
      "max_tokens": 30,
      "temperature": 0.0
    }'
```

推理示例输出：

```json
{
  "choices": [
    {
      "finish_reason": "stop",
      "index": 0,
      "logprobs": null,
      "message": {
        "audio": null,
        "content": "<think>\nOkay, I need to develop a character background for the user's query. Let me start by understanding the requirements. The character is an",
        "function_call": null,
        "refusal": null,
        "role": "assistant",
        "tool_calls": null
      }
    }
  ],
  "created": 1753768682,
  "id": "chatcmpl-772289b8-5998-4f6d-bd61-3659b684b347",
  "model": "Qwen/Qwen3-0.6B",
  "object": "chat.completion",
  "service_tier": null,
  "system_fingerprint": null,
  "usage": {
    "completion_tokens": 29,
    "completion_tokens_details": null,
    "prompt_tokens": 196,
    "prompt_tokens_details": null,
    "total_tokens": 225
  }
}
```

***如果你的集群中有不止一个 HttpRoute***
请在 HttpRoute.yaml 中添加 host，并加上 header
`curl -H "Host: llama3-70b-agg.example.com" ...` 或 `curl -H "Host: llama3-70b-disagg.example.com" http://localhost:8000/v1/models`

```bash
spec:
  hostnames:
    - llama3-70b-agg.example.com
```

### 8. 卸载 ###

如需卸载，请运行：

```bash
kubectl delete dynamoGraphDeployment vllm-agg
helm uninstall dynamo-gaie -n my-model

# To uninstall GAIE
# 1. Delete the inference-gateway
kubectl delete gateway inference-gateway --ignore-not-found

# 2. Uninstall kgateway helm releases
helm uninstall kgateway -n kgateway-system
helm uninstall kgateway-crds -n kgateway-system

# 3. Delete the kgateway-system namespace (optional, cleans up everything in it)
helm uninstall kgateway --namespace kgateway-system
kubectl delete namespace kgateway-system --ignore-not-found

# 4. Delete the Inference Extension CRDs
IGW_LATEST_RELEASE=v1.5.0-rc.2
kubectl delete -f https://github.com/kubernetes-sigs/gateway-api-inference-extension/releases/download/${IGW_LATEST_RELEASE}/manifests.yaml --ignore-not-found

# 5. Delete the Gateway API CRDs
GATEWAY_API_VERSION=v1.4.1
kubectl delete -f https://github.com/kubernetes-sigs/gateway-api/releases/download/$GATEWAY_API_VERSION/standard-install.yaml --ignore-not-found
```

## Gateway API Inference Extension 集成

本节文档对应 Gateway API Inference Extension **v1.5.0-rc.2** 的最新插件实现。

### 路由 bookkeeping 操作

EPP 执行 Dynamo 路由的 bookkeeping 操作，使 FrontEnd 的 Router 不必同步其状态。


### 头部路由提示

自 v1.5.0-rc.1 起，EPP 使用 **header 与 body 修改** 来传达路由决策。
插件设置 HTTP header 用于 worker 定向，并把预先计算的 token id 注入请求体（`nvext.token_data`），让 frontend sidecar 可以跳过冗余的 tokenize。

#### Dynamo 插件设置的 header

| Header | 说明 | 设置者 |
|--------|-------------|--------|
| `x-worker-instance-id` | 主 worker ID（解耦模式下为 decode worker） | kv-aware-scorer |
| `x-prefill-instance-id` | Prefill worker ID（仅解耦模式） | kv-aware-scorer |
