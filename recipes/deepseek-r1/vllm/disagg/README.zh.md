<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

### DeepSeek-R1 + vLLM —— 32x Hopper 上的解耦（Disaggregated）部署

本 recipe 使用 vLLM 在 4 台 Hopper 节点（node）上以解耦的预填充（prefill）/解码（decode）方式部署 DeepSeek-R1，共 32 张 GPU（16 张用于 prefill，16 张用于 decode）。

- 模型缓存 PVC + 下载 Job：`recipes/deepseek-r1/model-cache/`
- 部署清单：`recipes/deepseek-r1/vllm/disagg/deploy_hopper_16gpu.yaml`

### 0）先决条件：安装平台

按照 Kubernetes 部署指南安装 Dynamo 平台与先决组件（CRD/operator 等）：
- `docs/kubernetes/README.md`

请确保有一个启用 GPU、容量充足的集群（cluster）（4 个节点共 32x H100/H200 "Hopper"），且 NVIDIA GPU Operator 处于健康状态。

### 1）设置命名空间

```bash
export NAMESPACE=dynamo-system
kubectl create namespace ${NAMESPACE} || true
```

### 2）应用 Hugging Face Secret

将你的 HF token 编辑进给定的 Secret 并应用：

```bash
# 选项 A：应用 YAML（请先编辑文件填入你的 token）
kubectl apply -f ../../hf_hub_secret/hf_hub_secret.yaml -n ${NAMESPACE}

# 选项 B：直接创建
# kubectl create secret generic hf-token-secret \
#   --from-literal=HF_TOKEN="<your-hf-token>" \
#   -n ${NAMESPACE}
```

### 3）准备模型缓存并下载模型

将 `recipes/deepseek-r1/model-cache/model-cache.yaml` 中的 `storageClassName` 修改为与你的集群匹配的值，然后应用：

```bash
# 模型缓存的 PVC
# 请确保 model-cache.yaml 中的 storageClassName 匹配集群上可用的 StorageClass
kubectl apply -f ../../../deepseek-r1/model-cache/model-cache.yaml -n ${NAMESPACE}

# 将 DeepSeek-R1 权重下载到缓存中
kubectl apply -f ../../../deepseek-r1/model-cache/model-download.yaml -n ${NAMESPACE}

# 等待下载 Job 完成
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=6000s
```

这会填充：
- `/model-cache/deepseek-r1`
- `/model-cache/deepseek-r1-fp4`

### 4）部署 vLLM（解耦，16 路 Data-Expert Parallel）

应用多节点解耦部署：

```bash
kubectl apply -f ./deploy_hopper_16gpu.yaml -n ${NAMESPACE}
```

该清单跨多节点运行独立的 prefill 与 decode worker，每个 worker 都挂载共享的模型缓存，并针对 Hopper GPU 调优了相关设置。

通过端口转发并发送请求，可在本地测试该部署：

```bash
# 将 frontend Service 端口转发到 localhost:8000（请将 <frontend-svc> 替换为实际的 Service 名）
kubectl port-forward svc/vllm-dsr1-frontend 8000:8000 -n ${NAMESPACE} &
```

```bash
curl -sS http://localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer dummy' \
  -d '{
    "model": "deepseek-ai/DeepSeek-R1",
    "messages": [{"role":"user","content":"Say hello!"}],
    "max_tokens": 64
  }'
```



### 注意事项
- 关于 expert parallel 与高级部署配置的更多细节，请参阅 [vLLM Expert Parallel 部署文档](https://docs.vllm.ai/en/latest/serving/expert_parallel_deployment/)。
- 如果你的集群/网络要求特定的网卡接口，请相应调整清单中的环境变量（如 `NCCL_SOCKET_IFNAME`）。
- 如果你的 storage class 不同，请在应用 PVC 之前更新 `storageClassName`。
- **如果你想做多节点部署，必须在节点上启用 IBGDA（InfiniBand GPU Direct Async）。** 要启用 IBGDA，可参考此配置脚本：[configure_system_drivers.sh](https://github.com/vllm-project/vllm/blob/v0.11.2/tools/ep_kernels/configure_system_drivers.sh)。该脚本会配置 NVIDIA 驱动参数，并需要重启系统使其生效。
- `VLLM_MOE_DP_CHUNK_SIZE` 可进一步调优。当前选用 384 是在 16 张 H200 上仍可部署的最大值。该值应大于每个 rank 的并发数。
