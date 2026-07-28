<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# GLM-5 NVFP4 — 在 GB200 上进行解耦 Prefill/Decode

通过 Dynamo，在 GB200 节点上使用 SGLang，以解耦的预填充（prefill）/解码（decode）以及 EAGLE 推测式解码（speculative decoding）方式提供 [nvidia/GLM-5-NVFP4](https://huggingface.co/nvidia/GLM-5-NVFP4) 服务。

## 拓扑

| 角色      | 节点数 | 每节点 GPU | GPU 总数 | 并行方式               |
|-----------|--------|------------|----------|------------------------|
| Decode    | 4      | 4          | 16       | TP16 / DP16 / EP16     |
| Prefill   | 1      | 4          | 4        | TP4                    |

## 前置条件

- 5 个 4xGB200 节点，位于 NVL36 或 NVL72 域内
- 一个已安装 [Dynamo Operator](../../docs/kubernetes/README.md) 的 Kubernetes 集群（cluster）
- 一个用于存放模型权重的共享 NFS PVC

## 步骤 1：构建容器

本配方目前需要使用尚未发布的 SGLang 与 Dynamo 版本。
使用如下命令构建并推送一个包含所需依赖的容器：

```bash
docker buildx build \
  --platform linux/arm64 \
  --build-arg ARCH=arm64 \
  -t <your-registry>/sglang-dynamo-glm5:latest \
  -f recipes/glm-5-nvfp4/Dockerfile \
  --push .
```

## 步骤 2：下载模型

创建 PVC、HuggingFace token Secret，并下载模型权重：

```bash
kubectl apply -f recipes/glm-5-nvfp4/model-cache/model-cache.yaml

kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=<your-hf-token>

kubectl apply -f recipes/glm-5-nvfp4/model-cache/model-download.yaml
kubectl wait --for=condition=complete job/model-download --timeout=3600s
```

## 步骤 3：部署

编辑 `sglang/disagg/deploy.yaml` 并替换所有 `<placeholder>` 占位符：

- `<your-namespace>` —— 你的 Kubernetes 命名空间
- `<your-registry>/sglang-dynamo-glm5:latest` —— 你构建好的容器镜像

```bash
kubectl apply -f recipes/glm-5-nvfp4/sglang/disagg/deploy.yaml
```

监控启动过程（decode 加载权重并捕获 CUDA graphs 大约需要 15 分钟）：

```bash
kubectl get pods -n <your-namespace> -l app.kubernetes.io/part-of=glm5-sglang -w
```

## 步骤 4：测试

```bash
kubectl port-forward svc/glm5-sglang-frontend 8000:8000 -n <your-namespace> &
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"nvidia/GLM-5-NVFP4","messages":[{"role":"user","content":"Hello!"}],"max_tokens":128}'
```

## 步骤 5：性能基准（可选）

编辑 `sglang/disagg/perf.yaml` 设置好命名空间与 PVC，然后运行：

```bash
kubectl apply -f recipes/glm-5-nvfp4/sglang/disagg/perf.yaml
kubectl logs -f -l job-name=glm5-disagg-bench -n <your-namespace>
```

默认基准：ISL=1000，OSL=8192，并发=512（每 GPU 32）。

## 关键配置说明

### 推测式解码（EAGLE MTP）
两个环境变量启用可正常工作的推测式解码（接受率约 85-95%）：

- `SGLANG_ENABLE_SPEC_V2=1` —— 使用 EAGLEWorkerV2 与 overlap scheduler（重叠调度器）
- `SGLANG_NVFP4_CKPT_FP8_NEXTN_MOE=1` —— 在加载时将 BF16 的 MTP 层量化为 FP8，从而与基础模型的计算路径保持一致

`nvidia/GLM-5-NVFP4` 中的 MTP 层权重为 BF16（分布在分片 271-274 中），并已完整索引在 checkpoint 的 `model.safetensors.index.json` 内。

### KV 缓存（KV cache）
使用 `--kv-cache-dtype fp8_e4m3`（NSA 后端在 SM100/GB200 上会自动选择该值）。
相比 BF16 可节省约 50% 的 KV 内存。

### 服务发现
使用 Kubernetes 服务发现。Worker 注册通过 Kubernetes EndpointSlices 与 Pod 生命周期绑定，从而在高负载下避免 TTL 过期问题。

## 性能（ISL=1k，OSL=8k，并发=512）

| 指标               | 数值                  |
|--------------------|-----------------------|
| 输出吞吐           | 约 19,000 tokens/sec  |
| TTFT p50           | 约 850ms              |
| ITL 平均           | 约 24ms/token         |
| Tokens/user/sec    | 约 41                 |
