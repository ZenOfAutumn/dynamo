<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# DeepSeek-V4-Pro Recipe

Dynamo 上 **DeepSeek-V4-Pro** 跨两种后端（**vLLM**、**SGLang**）和两种硬件目标（**B200**、**GB200**）的 recipe。单节点聚合服务占满 B200 节点的 8 个 GPU；在 GB200 上，模型超出了单个 4 GPU NVL4 托盘，因此 V4-Pro 通过 NVLink72（MNNVL）跨两个 GB200 托盘——可以是 **跨节点 TP=8 的聚合** 或 **解耦 prefill/decode**。

| 变体 | 后端 | 硬件 | Manifest | 拓扑 | 容器 |
|---------|---------|----------|----------|----------|-----------|
| **vllm-agg-b200**       | vLLM   | 1 节点，8x B200 | [`vllm/agg/b200/deploy.yaml`](vllm/agg/b200/deploy.yaml)         | TP=8 + Expert Parallel（单节点）                                            | 预构建 NGC 镜像（`...1.2.0-deepseek-v4-cuda13-dev.3`，multi-arch） |
| **vllm-agg-gb200**      | vLLM   | 2 节点，每节点 4x GB200（共 8 个） | [`vllm/agg/gb200/deploy.yaml`](vllm/agg/gb200/deploy.yaml)       | TP=8 + Expert Parallel 跨节点，通过 ComputeDomain 的 MNNVL（NVLink72）           | 预构建 NGC 镜像（`...1.2.0-deepseek-v4-cuda13-dev.3`，multi-arch） |
| **vllm-disagg-gb200**   | vLLM   | 2 节点，每节点 4x GB200（8 prefill + 8 decode = 共 16 个） | [`vllm/disagg/gb200/deploy.yaml`](vllm/disagg/gb200/deploy.yaml) | 1P + 1D，每个 worker 上 DP=8 + Expert Parallel，通过 ComputeDomain 的 MNNVL（NVLink72） | 预构建 NGC 镜像（`...1.2.0-deepseek-v4-cuda13-dev.3`，multi-arch） |
| **sglang-agg**          | SGLang | 1 节点，8x B200 | [`sglang/agg/deploy.yaml`](sglang/agg/deploy.yaml)               | TP=8，通过 FlashInfer 的 MXFP4 MoE，EAGLE MTP 3/4                                   | 预构建 NGC 镜像；可选 [自定义构建](../container/) |

GB200 disagg 变体附带一个性能基准 Job：

| 性能 Job | 备注 |
|---|---|
| [`vllm/disagg/gb200/perf.yaml`](vllm/disagg/gb200/perf.yaml) | 对 `dsv4-pro-disagg-frontend:8000` 运行 `aiperf profile`，进行 8K 输入 / 1K 输出的并发扫描（256 / 512 / 1024）。可在 Job env 中覆盖 `CONCURRENCIES` 进行更小的冒烟测试。 |

状态：**实验性**（Day-0）。模态：仅文本。

## 前置条件

1. **已安装 Dynamo Platform** —— 参见 [Kubernetes 部署指南](../../../docs/kubernetes/README.md)。
2. **GPU 集群。** 硬件依变体而定：
   - **B200 变体**（`vllm-agg-b200`、`sglang-agg`）：单节点上 8 个可用 B200 GPU（x86_64）。TP=8 占满整机。
   - **GB200 变体**（`vllm-agg-gb200`、`vllm-disagg-gb200`）：**2 个 GB200 节点**，每节点 4 GPU（每节点单 NVL4 托盘），连接到 **同一 NVLink72 clique**。节点必须打 `nvidia.com/gpu.product=NVIDIA-GB200` label，并打 `kubernetes.io/arch=arm64:NoSchedule` taint。集群必须安装 **DRA / ComputeDomain controller**（用 `kubectl get crd | grep computedomain` 验证）；每个 manifest 的 `ComputeDomain` CR + `resourceClaims` 是 operator 把 worker pod 集合放置在同一 NVLink72 fabric 上的方式（agg 变体放置 2 个 pod，disagg 变体放置 4 个 pod）。
3. **HuggingFace token**，需具备访问 `deepseek-ai/DeepSeek-V4-Pro` 的权限。

## 快速开始

公共准备（执行一次——适用于全部变体）：

```bash
export NAMESPACE=dynamo-demo
kubectl create namespace ${NAMESPACE}

# HuggingFace token secret（被下载 Job 使用，方便起见 worker 也会挂载）
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="your-token-here" \
  -n ${NAMESPACE}

# 下载模型到 model-cache PVC。
# 编辑 model-cache/model-cache.yaml，将 storageClassName 设置为集群中的 RWX 类。
# PVC 申请 1500Gi；DeepSeek-V4-Pro 在磁盘上约 865 GB（64 个 safetensors 分片，
# FP4+FP8 混合），首次 apply 通常需要 1.5-3 小时。
kubectl apply -f model-cache/model-cache.yaml -n ${NAMESPACE}
kubectl apply -f model-cache/model-download.yaml -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=14400s
```

### 部署——vLLM B200（`vllm-agg-b200`）

```bash
kubectl apply -f vllm/agg/b200/deploy.yaml -n ${NAMESPACE}

# decode worker 首次启动最多约 90 分钟（TP=8 权重加载 +
# FlashInfer autotune + cudagraph 预热）。startup probe 已为此预留时间。
kubectl wait --for=condition=Ready pod \
  -l nvidia.com/dynamo-graph-deployment-name=dsv4-pro-agg \
  -n ${NAMESPACE} --timeout=5400s
```

### 部署——vLLM GB200 agg（`vllm-agg-gb200`）

```bash
kubectl apply -f vllm/agg/gb200/deploy.yaml -n ${NAMESPACE}

# decode worker 首次启动最多约 90 分钟（TP=8 权重加载
# + 跨节点 MNNVL 的 NCCL bring-up + 跨 2 节点 cudagraph 捕获）。
kubectl wait --for=condition=Ready pod \
  -l nvidia.com/dynamo-graph-deployment-name=dsv4-pro-agg \
  -n ${NAMESPACE} --timeout=5400s
```

### 部署——vLLM GB200 disagg（`vllm-disagg-gb200`）

```bash
kubectl apply -f vllm/disagg/gb200/deploy.yaml -n ${NAMESPACE}

# 每个 leader 首次启动最多约 90 分钟（DP=8 权重加载 +
# NIXL/UCX 设置 + 跨节点 MNNVL 的 NCCL bring-up + cudagraph 捕获）。
kubectl wait --for=condition=Ready pod \
  -l nvidia.com/dynamo-graph-deployment-name=dsv4-pro-disagg \
  -n ${NAMESPACE} --timeout=5400s

# 可选：运行性能基准 Job（8K 输入 / 1K 输出，c=256/512/1024 扫描）
# kubectl apply -f vllm/disagg/gb200/perf.yaml -n ${NAMESPACE}
```

### 部署——SGLang（`sglang-agg`）

```bash
kubectl apply -f sglang/agg/deploy.yaml -n ${NAMESPACE}

# decode worker 首次启动最多约 60 分钟（TP=8 权重加载 +
# DeepGEMM 预热 + cudagraph 预热）。startup probe 已为此预留时间。
kubectl wait --for=condition=Ready pod \
  -l nvidia.com/dynamo-graph-deployment-name=sglang-dsv4-pro \
  -n ${NAMESPACE} --timeout=3600s
```

## 测试部署

对你部署的变体进行端口转发：

```bash
# vLLM B200 agg 或 GB200 agg（DGD/service 名相同——同一 namespace
# 内一次只能部署其中一个变体）
kubectl port-forward svc/dsv4-pro-agg-frontend 8000:8000 -n ${NAMESPACE}

# vLLM GB200 disagg
kubectl port-forward svc/dsv4-pro-disagg-frontend 8000:8000 -n ${NAMESPACE}

# SGLang
kubectl port-forward svc/sglang-dsv4-pro-frontend 8000:8000 -n ${NAMESPACE}
```

请求形式相同。按上文的 Day-0 注意事项，对 vLLM 变体发送 `thinking: false`：

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-ai/DeepSeek-V4-Pro",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 100,
    "chat_template_kwargs": {"thinking": false}
  }'
```

## Recipe 详情

### vLLM B200 agg（`vllm/agg/b200/deploy.yaml`）

| Flag | 用途 |
|------|---------|
| `--tokenizer-mode deepseek_v4` | 选择 DeepSeek-V4 tokenizer |
| `--dyn-reasoning-parser deepseek_v4` | 把 chain-of-thought 提取为 `message.reasoning_content` |
| `--dyn-tool-call-parser deepseek_v4` | 输出兼容 OpenAI 的结构化 `tool_calls` |
| `--attention-config '{"use_fp4_indexer_cache":true}'` | 用于 CSA+HCA attention 的 Blackwell FP4 indexer 缓存 |
| `--kv-cache-dtype fp8` + `--block-size 256` | FP8 KV 缓存；block size 与上游 recipe 对齐 |
| `--tensor-parallel-size 8 --enable-expert-parallel` | 单节点 8 GPU 上的 TP=8，并对 MoE 专家启用 EP |
| `--compilation-config '{"mode":0,"cudagraph_mode":"FULL_DECODE_ONLY"}'` | 适合更大 Pro 模型的保守 cudagraph 模式（与上游 V4-Pro 示例一致） |
| `--no-enable-flashinfer-autotune` | 启动时跳过 per-shape FlashInfer autotune；dsv4 上需启用以保证精度 |
| `--max-num-seqs 256` | 并发上限 |

### vLLM GB200 agg（`vllm/agg/gb200/deploy.yaml`）

V4-Pro 在磁盘上约 865 GB，单个 GB200 NVL4 托盘（4 GPU 上约 768 GB HBM）放不下，所以 GB200 agg recipe 把一个 tensor-parallel 组横跨 **两个** 托盘——跨节点 TP all-reduce / all-gather 走 NVLink72（MNNVL），而不是 RoCE。两个 pod 由 DRA `ComputeDomain` controller 共置在同一 NVLink72 clique。

| Flag / 环境变量 | 用途 |
|---|---|
| `--tensor-parallel-size 8 --enable-expert-parallel` | 跨 2 节点（4 GPU/节点 × 2 节点）的 TP=8 + EP——无 DP。 |
| `--compilation-config '{"cudagraph_mode":"FULL_AND_PIECEWISE","custom_ops":["all"],"pass_config":{"fuse_allreduce_rms":false}}'` | FULL_AND_PIECEWISE cudagraph + 全部自定义 op；`fuse_allreduce_rms:false` 避免启动时的非致命 FlashInfer trtllm allreduce-norm workspace 警告。 |
| `--attention-config '{"use_fp4_indexer_cache":true}'` + `--moe-backend deep_gemm_mega_moe` | Blackwell FP4 indexer 缓存 + DeepGEMM "mega MoE" kernel——与 B200 agg 变体相同。 |
| `--no-enable-flashinfer-autotune` | 启动时跳过 per-shape FlashInfer autotune；dsv4 上需启用以保证精度 |
| `NCCL_MNNVL_ENABLE=1`、`UCX_CUDA_IPC_ENABLE_MNNVL=y`、`UCX_TLS=cuda_copy,cuda_ipc,tcp`、`NCCL_NVLS_ENABLE=1`、`NCCL_P2P_LEVEL=NVL` | 启用跨节点 NVLink72 / MNNVL fabric。因为 TP=8 进程组跨 2 节点，故必需。 |
| `ComputeDomain` CR + `resourceClaimTemplate`（manifest 顶部） | DRA 原语，请求调度器按需分配 MNNVL channel 并把 2-pod 集合共置在同一 NVLink72 clique。 |
| （无 `--data-parallel-rpc-port`） | 仅 TP——torch.distributed master 绑定 `MASTER_PORT`（29500）用于跨节点 rendezvous，这同时满足 operator 的 `wait-for-leader-mp` TCP 探测。 |

### vLLM GB200 disagg（`vllm/disagg/gb200/deploy.yaml`）

V4-Pro 在磁盘上约 865 GB，单个 GB200 NVL4 托盘（4 GPU 上约 768 GB HBM）放不下，所以 GB200 recipe 采用 **解耦的** prefill/decode 形态：一个 prefill 副本跨 2 个 GB200 节点（DP=8 + EP），一个 decode 副本跨 2 个 GB200 节点（DP=8 + EP），4 个 pod 全部由 DRA `ComputeDomain` controller 放置在同一 NVLink72 clique。

| Flag / 环境变量 | 用途 |
|---|---|
| `--data-parallel-size 8 --enable-expert-parallel --tensor-parallel-size 1` | 每个 worker 跨 2 节点（4 GPU/节点 × 2 节点）的 DP=8 + EP——TP=1 |
| `--data-parallel-rpc-port 29500` | 把 vLLM 的 DP coordinator 绑在 `:29500`。dynamo operator 的 `wait-for-leader-mp` init container 对 `<leader>:29500` 进行 TCP 探测并阻塞 worker 启动直到该端口可接收；把 DP coord 端口固定在 29500 让真实 RPC server 满足探测（比挂占位监听更干净）。 |
| `--disaggregation-mode prefill`（仅 prefill）+ `--kv-transfer-config '{"kv_connector":"NixlConnector","kv_role":"kv_both"}'` | prefill 通过 NIXL 写 KV block；decode 读取。NIXL 的 UCX active-messages 控制面走 TCP（`UCX_TLS=cuda_ipc,cuda_copy,tcp`），而批量 KV 走 MNNVL。 |
| `NCCL_MNNVL_ENABLE=1`、`UCX_CUDA_IPC_ENABLE_MNNVL=y`、`NCCL_NVLS_ENABLE=1`、`NCCL_P2P_LEVEL=NVL` | 启用跨节点 NVLink72 / MNNVL fabric。因为 prefill 与 decode worker 各自跨 2 节点，故必需。 |
| `ComputeDomain` CR + `resourceClaimTemplate`（manifest 顶部） | DRA 原语，请求调度器按需分配 MNNVL channel 并把 4-pod 集合共置在同一 NVLink72 clique。没有它，跨 pod 的 NCCL bring-up 会失败——TCP-only 回退对于跨 pod 的 DP=8 all-reduce 不可行。 |
| `--compilation-config '{"mode":0,"cudagraph_mode":"FULL_DECODE_ONLY"}'`（decode）、`--enforce-eager`（prefill） | 保守的编译/graph 配置——与 B200 agg 变体的 V4-Pro 调优一致。 |
| `--no-enable-flashinfer-autotune`（prefill + decode） | 启动时跳过 per-shape FlashInfer autotune；dsv4 上需启用以保证精度 |
| `--max-model-len 9280`、`--max-num-seqs 16`（prefill）/ `128`（decode） | 限制为 8K 输入 / 1K 输出基准形态。 |

### SGLang（`sglang/agg/deploy.yaml`）

| Flag | 用途 |
|------|---------|
| `--dyn-reasoning-parser deepseek_v4` | 把 chain-of-thought 提取为 `message.reasoning_content` |
| `--dyn-tool-call-parser deepseek_v4` | 输出兼容 OpenAI 的结构化 `tool_calls` |
| `--trust-remote-code` | V4 架构的自定义建模代码所必需 |
| `--tp 8` | 单节点全部 8 GPU 上的 tensor-parallel |
| `--moe-runner-backend flashinfer_mxfp4` | 通过 FlashInfer 为 V4 expert 权重提供的 MXFP4 MoE kernel |
| `--speculative-algo EAGLE` + `--speculative-num-steps 3` + `--speculative-eagle-topk 1` + `--speculative-num-draft-tokens 4` | EAGLE MTP 投机解码（3 个 draft step、EAGLE head 上 top-1、每步 4 个 draft token） |
| `--chunked-prefill-size 4096` | 以 4k tokens 为粒度对长 prompt 分块以保证稳态 decode 交错 |
| `--disable-flashinfer-autotune` | 启动时跳过 per-shape autotune；dsv4 base 自带预调优默认值 |

### 为什么 TP=8（而不是像 Flash 那样 DP=4）？

DeepSeek-V4-Pro 在磁盘上比 Flash 大约 5.5 倍（~865 GB vs. ~160 GB）。在 FP4+FP8 混合权重下，它在典型 batch shape 下放不进 4 个 rank，所以两端在 Pro 上的上游测试形态都是 **TP=8**——要么在一个 B200 节点的 8 个 GPU 上，要么跨两个由 NVLink72 连接的 GB200 NVL4 托盘。在 vLLM 上，Expert Parallel 叠加在 TP 之上——TP 切分稠密（attention/router/norm）权重，EP 切分专家。在 SGLang 上，MXFP4 MoE 后端在同一 TP=8 进程组内部处理专家切分。

## 模型详情

来源：[`deepseek-ai/DeepSeek-V4-Pro` 模型卡](https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro)（preview 发布）：

| | |
|---|---|
| **模型** | `deepseek-ai/DeepSeek-V4-Pro`（MoE，1.6T 总 / 每 token 49B 激活） |
| **上下文长度** | 1M tokens |
| **Checkpoint** | 混合精度——MoE 专家权重 FP4；其余大多数参数 FP8 |
| **Attention** | 混合 Compressed Sparse Attention（CSA）+ Heavily Compressed Attention（HCA）。vLLM 变体通过 `--attention-config '{"use_fp4_indexer_cache":true}'` 启用 Blackwell FP4 indexer 缓存 |
| **残差路径** | Manifold-Constrained Hyper-Connections（mHC） |
| **推理模式** | 通过 `chat_template_kwargs` 暴露三种思考强度：`{}`（Non-think）、`{"thinking":true,"reasoning_effort":"high"}`（Think High）、`{"thinking":true,"reasoning_effort":"max"}`（Think Max——需要 `--max-model-len >= 393216`） |
| **长上下文效率** | 模型卡所述：1M 上下文下 per-token 推理 FLOP 约为 DeepSeek-V3.2 的 27%，KV 缓存约为 10% |
| **License** | MIT |

Recipe 级（每变体）设置：

| | vLLM（`vllm-agg`） | SGLang（`sglang-agg`） |
|---|---|---|
| **后端镜像** | 预构建 `nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.2.0-deepseek-v4-cuda13-dev.3`（multi-arch） | 预构建 `nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.2.0-deepseek-v4-cuda12-dev.3` |
| **并行度** | TP=8，启用 Expert Parallel | TP=8 |
| **MoE 后端** | vLLM 的 V4 expert kernel（FP4） | FlashInfer MXFP4 |
| **KV 缓存** | FP8，block size 256 | engine 默认 |
| **投机解码** | — | EAGLE MTP（3 步 / 4 个 draft token） |

## 验证 Reasoning

两个变体上流程一致——同一模型、同一 `--dyn-reasoning-parser deepseek_v4`。在 vLLM 变体上，按 Day-0 注意事项省略 `chat_template_kwargs.thinking`（或设为 `false`）：

```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-ai/DeepSeek-V4-Pro",
    "messages": [{"role": "user", "content": "What is 2+2? Answer briefly."}],
    "max_tokens": 200
  }' | python3 -m json.tool
```

期望：

- `choices[0].message.reasoning_content` 包含模型的 chain-of-thought。
- `choices[0].message.content` 仅含最终答案。
- 两个字段中均无原始 `</think>` 标签。

如果 `reasoning_content` 为 `null` 且 `</think>` 出现在 `content` 中，则 reasoning parser 未接通——确认 worker 命令上有 `--dyn-reasoning-parser deepseek_v4`。

## 验证工具调用

两个变体上流程一致：

```bash
curl -s http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-ai/DeepSeek-V4-Pro",
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

如果 `tool_calls` 缺失而原始的 tool-call 标记出现在 `content`，确认 worker 命令上有 `--dyn-tool-call-parser deepseek_v4`。

## 备注

### 通用

- **Storage class。** 将 `model-cache/model-cache.yaml` 中的 `storageClassName` 更新为可向 Frontend 与 worker pod 提供 PVC 的 RWX 类。
- **模型大小。** `deepseek-ai/DeepSeek-V4-Pro` 在磁盘上约 865 GB（64 个 FP4+FP8 混合 safetensors 分片）。1500Gi PVC 留出约 1.7 倍余量给 HF 缓存元数据与一个备用 revision。
- **Parser 标志。** 在 worker 上使用 Dynamo 变体（`--dyn-reasoning-parser`、`--dyn-tool-call-parser`）。各引擎自带的 `--reasoning-parser` / `--tool-call-parser` 是引擎侧的，不会接入 Dynamo 的 OpenAI renderer。
- **离线模型缓存。** 两种 worker 都以 `HF_HUB_OFFLINE=1` 运行，因此引擎从 PVC 读取缓存权重，启动时不会联系 HF Hub。HF token secret 出于防御性挂载；下载 Job 完成后运行时不再需要。
- **首次启动较慢。** decode worker 在 8 个 TP rank 上加载权重并预热 CUDA graph / DeepGEMM kernel；manifest 的 startup probe 允许约 60–90 分钟才宣告失败。

### vLLM 专属

- **预构建镜像。** 三个 vLLM manifest（`vllm/agg/b200/`、`vllm/agg/gb200/`、`vllm/disagg/gb200/`）都引用 `nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.2.0-deepseek-v4-cuda13-dev.3`（multi-arch）。如需从源码重新构建（自定义 Dynamo 分支、不同的 vLLM base 等），参见 [`<repo_root>/container/README.md`](../../../container/README.md)。
- **Engine-ready 超时。** `VLLM_ENGINE_READY_TIMEOUT_S=5400` 与 startup probe 预算（`failureThreshold: 540`，`periodSeconds: 10`）一致。
- **FlashInfer autotune。** `--no-enable-flashinfer-autotune` 在每个 vLLM worker（prefill 与 decode）上设置以跳过启动时的 per-shape FlashInfer autotune。dsv4 上必需：当前 autotuner 产生的调优会让 GSM8k 精度回退。跳过它也缩短首次启动预热时间。
- **GB200：agg vs. disagg。** 两者都通过 MNNVL/ComputeDomain 把 V4-Pro 跨两个 GB200 NVL4 托盘展开。agg 变体在两节点上运行一个 TP=8 组（更低延迟、更简单拓扑、2 个 pod）；disagg 变体运行独立的 prefill 与 decode DP=8 worker（在高并发下更高的稳态吞吐，4 个 pod）。通用服务用 agg 变体；当工作负载从 prefill/decode 分离中受益时使用 disagg 变体。

### SGLang 专属

- **预构建镜像。** `sglang/agg/deploy.yaml` 已引用公共 NGC tag `nvcr.io/nvidia/ai-dynamo/sglang-runtime:1.2.0-deepseek-v4-cuda12-dev.3`。如需重新构建（自定义 Dynamo 分支、不同的 SGLang base 等），参见 [`recipes/deepseek-v4/container/README.md`](../container/README.md)。
- **DeepGEMM / FlashInfer 预热。** `SGLANG_JIT_DEEPGEMM_PRECOMPILE=0` + `SGLANG_JIT_DEEPGEMM_FAST_WARMUP=1` 跳过缓慢的预编译并使用快速预热路径。`--disable-flashinfer-autotune` 在启动时跳过 per-shape FlashInfer autotune；dsv4 base 自带预调优默认值。
- **NCCL / Gloo。** `NCCL_CUMEM_ENABLE=1` 为 Blackwell 上的 V4 NCCL 集合通信设置。`GLOO_SOCKET_IFNAME=eth0` 把 Gloo 绑定到标准 pod 网卡。

## 同系 Recipe

[DeepSeek-V4-Flash](../deepseek-v4-flash/) 是更小的同系 recipe（284B / 13B 激活、4x B200），共享相同的 dsv4 vLLM 与 SGLang 容器镜像。
