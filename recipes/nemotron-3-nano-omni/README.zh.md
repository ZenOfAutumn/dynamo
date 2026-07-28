<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Nemotron 3 Nano Omni NVFP4

使用 vLLM 与一个聚合（aggregated）Dynamo 部署来服务 [nvidia/Nemotron-3-Nano-Omni-30B-A3B-Reasoning-NVFP4](https://huggingface.co/nvidia/Nemotron-3-Nano-Omni-30B-A3B-Reasoning-NVFP4)。

本示例构建了一个自定义容器，将 `ai-dynamo` wheel（来自 <https://pypi.nvidia.com/ai-dynamo/>）叠加到上游 vLLM 镜像之上 —— 不需要源码构建，也不需要 Rust 工具链。

## 拓扑

| 角色 | 副本数 | 每副本 GPU 数 | 备注 |
|------|----------|--------------|-------|
| Frontend | 1 | 0 | 启用 prefix-hash KV 路由的 Dynamo 前端 |
| vLLM worker | 1 | 1 | 文本、图像、视频、音频输入 |

## 前置条件

- 已安装 [Dynamo Operator](../../docs/kubernetes/README.md) 的 Kubernetes 集群
- 每个 worker 副本需要一块 NVIDIA GPU
- 用于 Hugging Face 模型缓存的共享 PVC 存储
- 对 `nvidia/Nemotron-3-Nano-Omni-30B-A3B-Reasoning-NVFP4` 的 Hugging Face 访问权限

## 步骤 1：构建容器

```bash
docker build \
  -t <your-registry>/nemotron-omni-vllm:latest \
  -f recipes/nemotron-3-nano-omni/Dockerfile \
  recipes/nemotron-3-nano-omni
docker push <your-registry>/nemotron-omni-vllm:latest
```

实用的构建参数：

- `BASE_IMAGE=<image>` —— 锁定到不同的 vLLM 基础镜像（默认 `vllm/vllm-openai:v0.20.0`）。
- `DYNAMO_VERSION=<version>` —— 锁定到来自 <https://pypi.nvidia.com/ai-dynamo/> 的特定 `ai-dynamo` 发布版或 nightly 版。默认跟踪最新已测试的 nightly。务必确保所选 wheel 的 `vllm` 依赖与 `BASE_IMAGE` 匹配。

## 步骤 2：下载模型

创建 PVC、Hugging Face token Secret，并下载模型权重：

```bash
export NAMESPACE=<your-namespace>

# Create the namespace if it does not already exist.
kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

# First edit storageClassName in model-cache.yaml for your cluster.
kubectl apply -f recipes/nemotron-3-nano-omni/model-cache/model-cache.yaml -n ${NAMESPACE}

kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=<your-hf-token> \
  -n ${NAMESPACE}

kubectl apply -f recipes/nemotron-3-nano-omni/model-cache/model-download.yaml -n ${NAMESPACE}
kubectl wait --for=condition=complete job/model-download -n ${NAMESPACE} --timeout=3600s
```

## 步骤 3：部署

编辑 `vllm/agg/deploy.yaml`，将所有 `<placeholder>` 替换为实际值：

- `<your-registry>/nemotron-omni-vllm:latest` - 你构建的容器镜像

如果你的 registry 是私有的，请在 deployment 中添加合适的 `imagePullSecrets`。

```bash
kubectl apply -f recipes/nemotron-3-nano-omni/vllm/agg/deploy.yaml -n ${NAMESPACE}
```

监控启动：

```bash
kubectl get pods -n ${NAMESPACE} -l nvidia.com/dynamo-graph-deployment-name=nemotron-omni-vllm-agg -w
```

## 步骤 4：测试

```bash
kubectl port-forward svc/nemotron-omni-vllm-agg-frontend 8000:8000 -n ${NAMESPACE}
```

在另一个终端发送一个最小的纯文本请求：

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/Nemotron-3-Nano-Omni-30B-A3B-Reasoning-NVFP4",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 128
  }'
```

要走多模态路径，可以附加一张图像：

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/Nemotron-3-Nano-Omni-30B-A3B-Reasoning-NVFP4",
    "messages": [{
      "role": "user",
      "content": [
        {"type": "image_url", "image_url": {"url": "https://huggingface.co/datasets/huggingface/documentation-images/resolve/main/diffusers/inpaint.png"}},
        {"type": "text", "text": "Describe what is in this image."}
      ]
    }],
    "max_tokens": 256
  }'
```

……或一段音频：

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nvidia/Nemotron-3-Nano-Omni-30B-A3B-Reasoning-NVFP4",
    "messages": [{
      "role": "user",
      "content": [
        {"type": "audio_url", "audio_url": {"url": "https://raw.githubusercontent.com/yuekaizhang/Triton-ASR-Client/main/datasets/mini_en/wav/1221-135766-0002.wav"}},
        {"type": "text", "text": "Transcribe this audio clip."}
      ]
    }],
    "max_tokens": 256
  }'
```

## 关键配置说明

- `--enable-multimodal` 启用图像、视频、音频输入。
- `--media-io-kwargs '{"video": {"num_frames": 512, "fps": 1}}'` 以每秒 1 帧的速率采样长视频，最多 512 帧。
- `--dyn-tool-call-parser nemotron_nano` 与 `--dyn-reasoning-parser nemotron_nano` 启用 Nemotron Nano 的工具调用与推理解析。
- 前端使用 `--router-mode kv --no-kv-events`，通过前缀哈希近似实现 KV 感知路由，无需后端发出 KV 事件。

## 可选：不使用 NATS 运行

Dynamo runtime 默认使用 NATS 作为事件面（event plane），并在环境变量中设置了 `NATS_SERVER` 时连接到 NATS 服务器（在大多数集群上 operator 会自动注入该变量）。在没有 NATS 的集群上 —— 或者你不希望引入该依赖时 —— 可以仅使用 TCP 请求面 + ZMQ 事件面运行。在 Frontend 与 VllmWorker 中均加入：

```yaml
mainContainer:
  env:
    - name: DYN_EVENT_PLANE
      value: zmq
  command: ["/bin/bash", "-lc"]
  args:
    # Operator-injected NATS_SERVER takes effect even when set to ""; we have
    # to actually unset it before the runtime reads env.
    - >-
      unset NATS_SERVER &&
      exec python3 -m dynamo.frontend ...   # or dynamo.vllm
```

请求面已默认使用 TCP，因此不需要其他额外标志。

## 文件布局

```text
recipes/nemotron-3-nano-omni/
  README.md
  Dockerfile
  model-cache/
    model-cache.yaml
    model-download.yaml
  vllm/
    agg/
      deploy.yaml
```
