<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# DeepSeek-V4-Flash Recipe

Dynamo 上 **DeepSeek-V4-Flash** 的聚合服务 recipe。文档并列覆盖两种后端（**vLLM** 与 **SGLang**）和两种硬件目标（**B200** 与 **GB200**）。所有四个变体都是单副本、仅 decode 的部署，使用 4 个 GPU。

| 变体 | 后端 | 硬件 | Manifest | 拓扑 | 容器 |
|---------|---------|----------|----------|----------|-----------|
| **vllm-agg-b200**     | vLLM   | 4x B200  | [`vllm/agg_b200/deploy.yaml`](vllm/agg_b200/deploy.yaml)     | DP=4 + Expert Parallel，TP=1                   | 预构建 NGC 镜像（`...1.2.0-deepseek-v4-cuda13-dev.3`，multi-arch） |
| **vllm-agg-gb200**    | vLLM   | 4x GB200 | [`vllm/agg_gb200/deploy.yaml`](vllm/agg_gb200/deploy.yaml)   | TP=4 + Expert Parallel，`deep_gemm_mega_moe`   | 预构建 NGC 镜像（`...1.2.0-deepseek-v4-cuda13-dev.3`，multi-arch） |
| **sglang-agg**        | SGLang | 4x B200  | [`sglang/agg/deploy.yaml`](sglang/agg/deploy.yaml)           | TP=4，通过 FlashInfer 的 MXFP4 MoE，EAGLE MTP 3/4 | 预构建 NGC 镜像（`...1.2.0-deepseek-v4-cuda12-dev.3`）；可选 [自定义构建](../container/) |
| **sglang-agg-gb200**  | SGLang | 4x GB200 | [`sglang/agg-gb200/deploy.yaml`](sglang/agg-gb200/deploy.yaml) | TP=4，通过 FlashInfer 的 MXFP4 MoE，EAGLE MTP 3/4 | 预构建 NGC 镜像（`...1.2.0-deepseek-v4-cuda13-dev.3`，arm64） |

B200 变体在 B200 节点的 8 个 GPU 中占用 4 个；GB200 变体占用单个 GB200 NVL4 托盘的全部 4 个 GPU。

状态：**实验性**（Day-0）。模态：仅文本。

## 前置条件

1. **已安装 Dynamo Platform** —— 参见 [Kubernetes 部署指南](../../../docs/kubernetes/README.md)。
2. **GPU 集群。** 同一节点上至少 4 个匹配架构的 GPU：
   - **B200 变体**：4 个 B200 GPU（x86_64）。
   - **GB200 变体**：4 个 GB200 GPU（单个 NVL4 托盘，arm64）。节点必须打 `nvidia.com/gpu.product=NVIDIA-GB200` label，并打 `kubernetes.io/arch=arm64:NoSchedule` taint（manifest 中已带匹配的 `nodeSelector` + `toleration`）。
3. **HuggingFace token**，需具备访问 `deepseek-ai/DeepSeek-V4-Flash` 的权限。

## 快速开始

公共准备（执行一次——适用于所有变体）：

```bash
export NAMESPACE=dynamo-demo
kubectl create namespace ${NAMESPACE}

# HuggingFace token secret（被下载 Job 使用，方便起见 worker 也会挂载）
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="your-token-here" \
  -n ${NAMESPACE}

# 把模型下载到 model-cache PVC。
# 编辑 model-cache/model-cache.yaml，将 storageClassName 设置为集群中的 RWX 类。
# PVC 申请 400Gi；DeepSeek-V4-Flash 在磁盘上约 160GB（46 个 safetensors 分片，
# FP4+FP8 混合），首次 apply 通常需要 30-60 分钟下载。
kubectl apply -f model-cache/model-cache.yaml -n ${NAMESPACE}
kubectl apply -f model-cache/model-download.yaml -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=7200s
```

### 部署——vLLM B200（`vllm-agg-b200`）

```bash
kubectl apply -f vllm/agg_b200/deploy.yaml -n ${NAMESPACE}

# decode worker 首次启动最多约 60 分钟（权重加载 +
# FlashInfer autotune + cudagraph 预热）。startup probe 已为此预留时间。
kubectl wait --for=condition=Ready pod \
  -l nvidia.com/dynamo-graph-deployment-name=dsv4-flash-agg \
  -n ${NAMESPACE} --timeout=3600s
```

### 部署——vLLM GB200（`vllm-agg-gb200`）

```bash
kubectl apply -f vllm/agg_gb200/deploy.yaml -n ${NAMESPACE}

# 首次启动约 60 分钟；manifest 的 startup probe 已预留时间。
kubectl wait --for=condition=Ready pod \
  -l nvidia.com/dynamo-graph-deployment-name=dsv4-flash-agg \
  -n ${NAMESPACE} --timeout=3600s
```

### 部署——SGLang B200（`sglang-agg`）

```bash
kubectl apply -f sglang/agg/deploy.yaml -n ${NAMESPACE}

# decode worker 首次启动最多约 60 分钟（权重加载 +
# DeepGEMM 预热 + cudagraph 预热）。startup probe 已为此预留时间。
kubectl wait --for=condition=Ready pod \
  -l nvidia.com/dynamo-graph-deployment-name=sglang-dsv4-flash \
  -n ${NAMESPACE} --timeout=3600s
```

### 部署——SGLang GB200（`sglang-agg-gb200`）

```bash
kubectl apply -f sglang/agg-gb200/deploy.yaml -n ${NAMESPACE}

kubectl wait --for=condition=Ready pod \
  -l nvidia.com/dynamo-graph-deployment-name=sglang-dsv4-flash \
  -n ${NAMESPACE} --timeout=3600s
```

## 测试部署

对所部署的变体进行端口转发：

```bash
# vLLM
kubectl port-forward svc/dsv4-flash-agg-frontend 8000:8000 -n ${NAMESPACE}

# SGLang
kubectl port-forward svc/sglang-dsv4-flash-frontend 8000:8000 -n ${NAMESPACE}
```

请求形式相同——同样的模型名、同样的 OpenAI 兼容端点：

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-ai/DeepSeek-V4-Flash",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 100
  }'
```

## Recipe 详情

### vLLM B200（`vllm/agg_b200/deploy.yaml`）

| Flag | 用途 |
|------|---------|
| `--tokenizer-mode deepseek_v4` | 选择 DeepSeek-V4 tokenizer |
| `--dyn-reasoning-parser deepseek_v4` | 将 chain-of-thought 提取为 `message.reasoning_content` |
| `--dyn-tool-call-parser deepseek_v4` | 输出兼容 OpenAI 的结构化 `tool_calls` |
| `--attention-config '{"use_fp4_indexer_cache":true}'` | 用于 CSA+HCA attention 的 Blackwell FP4 indexer 缓存 |
| `--kv-cache-dtype fp8` + `--block-size 256` | FP8 KV 缓存；block size 与上游 recipe 对齐 |
| `--tensor-parallel-size 1 --data-parallel-size 4 --enable-expert-parallel` | 4 GPU 上的 DP=4 + EP（TP=1） |
| `--compilation-config '{"cudagraph_mode":"FULL_AND_PIECEWISE","custom_ops":["all"]}'` | 来自上游 recipe 的单节点 DEP 编译配置 |
| `--no-enable-flashinfer-autotune` | 启动时跳过 per-shape FlashInfer autotune；dsv4 上需启用以确保正确的精度 |
| `--max-num-seqs 256` | 并发上限 |

### vLLM GB200（`vllm/agg_gb200/deploy.yaml`）

OpenAI renderer 接线与 B200 变体相同；下方差异来自 V4-Flash 的 [上游 vLLM GB200 recipe](https://recipes.vllm.ai/deepseek-ai/DeepSeek-V4-Flash?features=tool_calling,reasoning&hardware=gb200)。

| Flag / 环境变量 | 用途 |
|---|---|
| `--tensor-parallel-size 4 --enable-expert-parallel` | NVL4 托盘 4 GPU 上的 **TP=4 + EP**（去除 DP——GB200 托盘内的 NVLink 让该规模下 TP 更具吸引力） |
| `--moe-backend deep_gemm_mega_moe` | DeepGEMM "mega MoE" kernel——在 Blackwell 上为 V4 expert 路由优化的 FP8 MoE 路径 |
| `--no-enable-flashinfer-autotune` | 启动时跳过 per-shape FlashInfer autotune；dsv4 上需启用以确保正确的精度 |
| `NCCL_NVLS_ENABLE=1`、`NCCL_P2P_LEVEL=NVL`、`VLLM_USE_NCCL_SYMM_MEM=1` | 启用 NVLink Sharp（NVLS）多播以实现托盘内的 one-shot all-reduce |

### SGLang B200（`sglang/agg/deploy.yaml`）

| Flag | 用途 |
|------|---------|
| `--dyn-reasoning-parser deepseek_v4` | 将 chain-of-thought 提取为 `message.reasoning_content` |
| `--dyn-tool-call-parser deepseek_v4` | 输出兼容 OpenAI 的结构化 `tool_calls` |
| `--trust-remote-code` | V4 架构的自定义建模代码所必需 |
| `--tp 4` | 一个节点上 4 GPU 的 tensor-parallel |
| `--moe-runner-backend flashinfer_mxfp4` | 通过 FlashInfer 为 V4 expert 权重提供的 MXFP4 MoE kernel |
| `--speculative-algo EAGLE` + `--speculative-num-steps 3` + `--speculative-eagle-topk 1` + `--speculative-num-draft-tokens 4` | EAGLE MTP 投机解码（3 个 draft step、EAGLE head 上 top-1、每步 4 个 draft token） |
| `--chunked-prefill-size 4096` | 以 4k tokens 为粒度对长 prompt 分块以保证稳态 decode 交错 |
| `--disable-flashinfer-autotune` | 启动时跳过 per-shape autotune；dsv4 base 已带预调优默认值 |

## 模型详情

| | |
|---|---|
| **模型** | `deepseek-ai/DeepSeek-V4-Flash`（MoE，284B 总 / 13B 激活） |
| **Checkpoint** | 混合 FP4（专家权重）+ FP8（attention、norm、router） |
| **Attention** | 混合 CSA + HCA，Blackwell FP4 indexer 缓存 |

Recipe 级（每变体）设置：

| | vLLM B200（`vllm-agg-b200`） | vLLM GB200（`vllm-agg-gb200`） | SGLang B200（`sglang-agg`） | SGLang GB200（`sglang-agg-gb200`） |
|---|---|---|---|---|
| **后端镜像** | 预构建 `nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.2.0-deepseek-v4-cuda13-dev.3`（multi-arch） | 预构建 `nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.2.0-deepseek-v4-cuda13-dev.3`（multi-arch） | 预构建 `nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.2.0-deepseek-v4-cuda12-dev.3` | 预构建 `nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.2.0-deepseek-v4-cuda13-dev.3` |
| **并行度** | DP=4 + Expert Parallel，TP=1 | TP=4 + Expert Parallel | TP=4 | TP=4 |
| **MoE 后端** | vLLM 的 V4 expert kernel（FP4） | DeepGEMM mega MoE | FlashInfer MXFP4 | FlashInfer MXFP4 |
| **KV 缓存** | FP8，block size 256 | FP8，block size 256 | engine 默认 | engine 默认 |
| **投机解码** | — | — | EAGLE MTP（3 步 / 4 个 draft token） | EAGLE MTP（3 步 / 4 个 draft token） |

## 验证 Reasoning

两个变体上流程一致——同一模型，同一 `--dyn-reasoning-parser deepseek_v4`：

```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-ai/DeepSeek-V4-Flash",
    "messages": [{"role": "user", "content": "What is 2+2? Answer briefly."}],
    "max_tokens": 200
  }' | python3 -m json.tool
```

期望：

- `choices[0].message.reasoning_content` 包含模型的 chain-of-thought。
- `choices[0].message.content` 仅包含最终答案。
- 两个字段中均无原始 `</think>` 标签。

如果 `reasoning_content` 为 `null` 且 `</think>` 出现在 `content` 中，则 reasoning parser 未接通——确认 worker 命令上有 `--dyn-reasoning-parser deepseek_v4`。

## 验证工具调用

两个变体上流程一致：

```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-ai/DeepSeek-V4-Flash",
    "messages": [{"role": "user", "content": "What is the weather in San Francisco?"}],
    "tools": [{
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get the current weather for a location",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {"type": "string", "description": "City name"}
          },
          "required": ["location"]
        }
      }
    }],
    "max_tokens": 300
  }' | python3 -m json.tool
```

期望：

- `choices[0].message.tool_calls` 是带有 `function.name`、`function.arguments` 与 `id` 的结构化数组。
- `choices[0].finish_reason` 为 `"tool_calls"`。
- `choices[0].message.reasoning_content` 可能包含模型对工具选择的推理。

如果 `tool_calls` 缺失而原始的 tool-call 标记出现在 `content` 中，确认 worker 命令上有 `--dyn-tool-call-parser deepseek_v4`。

## 备注

### 通用

- **Storage class。** 将 `model-cache/model-cache.yaml` 中的 `storageClassName` 更新为可向 Frontend 与 worker pod 提供 PVC 的 RWX 类。
- **模型大小。** `deepseek-ai/DeepSeek-V4-Flash` 在磁盘上约 160 GB（46 个 FP4+FP8 混合 safetensors 分片）。400Gi PVC 为 HF 缓存元数据与一个备用 revision 留出余量。
- **Parser 标志。** 在 worker 上使用 Dynamo 变体（`--dyn-reasoning-parser`、`--dyn-tool-call-parser`）。各引擎自带的 `--reasoning-parser` / `--tool-call-parser` 是引擎侧的，不会接入 Dynamo 的 OpenAI renderer。
- **离线模型缓存。** 两种 worker 都以 `HF_HUB_OFFLINE=1` 运行，因此引擎从 PVC 读取已缓存权重，启动时不会联系 HF Hub。HF token secret 出于防御性目的挂载；下载 Job 完成后运行时不再需要它。
- **首次启动较慢。** decode worker 在首次启动时加载权重并预热 CUDA graph / DeepGEMM kernel；manifest 的 startup probe 允许最多约 60 分钟（`failureThreshold: 360`，`periodSeconds: 10`）。

### vLLM 专属

- **预构建镜像。** `vllm/agg_b200/deploy.yaml` 与 `vllm/agg_gb200/deploy.yaml` 都引用 `nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.2.0-deepseek-v4-cuda13-dev.3`（multi-arch）。如需从源码重新构建（自定义 Dynamo 分支、不同的 vLLM base 等），参见 [`<repo_root>/container/README.md`](../../../container/README.md)。
- **Engine-ready 超时。** `VLLM_ENGINE_READY_TIMEOUT_S=3600` 与两个变体的 startup probe 预算一致。
- **DP 稳定性（仅 B200）。** `VLLM_RANDOMIZE_DP_DUMMY_INPUTS=1` 与 `VLLM_SKIP_P2P_CHECK=1` 沿用 DeepSeek-R1 vLLM recipe，并稳定 DP 假输入。GB200 变体使用 TP（无 DP），所以未设置 `VLLM_RANDOMIZE_DP_DUMMY_INPUTS`。
- **FlashInfer autotune。** `--no-enable-flashinfer-autotune` 在启动时跳过 per-shape FlashInfer autotune，两个 vLLM 变体上均设置。dsv4 上必需：当前 autotuner 产生的调优会让 GSM8k 精度回退。跳过它也缩短首次启动预热时间。
- **GB200 上的 FlashInfer TRT-LLM allreduce。** 你可能会看到非致命的启动警告 `Failed to initialize FlashInfer Allreduce norm fusion workspace ... Flashinfer allreduce-norm fusion will be disabled`。vLLM 会回退到非融合的 allreduce + RMSNorm；正确性不受影响。要启用融合 kernel，设置编译 pass：`--compilation-config '{"mode":3,"cudagraph_mode":"FULL_AND_PIECEWISE","custom_ops":["all"],"pass_config":{"fuse_allreduce_rms":true}}'`。

### SGLang 专属

- **预构建镜像。** `sglang/agg/deploy.yaml` 引用 `nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.2.0-deepseek-v4-cuda12-dev.3`，`sglang/agg-gb200/deploy.yaml` 引用 arm64 兄弟版本 `nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.2.0-deepseek-v4-cuda13-dev.3`。如需重新构建（自定义 Dynamo 分支、不同的 SGLang base 等），参见 [`recipes/deepseek-v4/container/README.md`](../container/README.md)。
- **DeepGEMM / FlashInfer 预热。** `SGLANG_JIT_DEEPGEMM_PRECOMPILE=0` + `SGLANG_JIT_DEEPGEMM_FAST_WARMUP=1` 跳过缓慢的预编译并使用快速预热路径。`--disable-flashinfer-autotune` 在启动时跳过 per-shape FlashInfer autotune；dsv4 base 自带预调优默认值。
- **NCCL / Gloo。** `NCCL_CUMEM_ENABLE=1` 为 Blackwell 上的 V4 NCCL 集合通信设置。`GLOO_SOCKET_IFNAME=eth0` 把 Gloo 绑定到标准 pod 网卡。

## 同系 Recipe

[DeepSeek-V4-Pro](../deepseek-v4-pro/) 是更大的同系 recipe（1.6T / 49B 激活、1M 上下文、8x B200），共享相同的 dsv4 vLLM 与 SGLang 容器镜像。
