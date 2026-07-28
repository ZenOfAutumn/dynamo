# DeepSeek V3.2 NVFP4：聚合（Aggregated）轮询路由 vs. 解耦（Disaggregated）KV 路由 + WideEP

本 **GB200 NVL72** recipe 针对 DeepSeek V3.2，演示在改编自 [Mooncake FAST25 论文](https://github.com/kvcache-ai/Mooncake) 的合成 trace 数据集上，**聚合（轮询）路由**与**解耦（KV 感知）路由 + WideEP** 之间的性能差异。

## 结果

https://github.com/user-attachments/assets/fcdb703c-7c1a-4109-a7ca-54196fcef885

## 实验概述

我们在 **8 节点共 32× GB200 GPU** 上对比两种部署模式：

| 模式 | 路由 | 配置 |
|------|---------|---------------|
| **Aggregated** | 轮询 | 4× DEP8 worker |
| **Disaggregated** | KV 感知 | 2× prefill + 2× decode，配合 WideEP（DEP8） |

## 数据集：基于 Mooncake 的合成 Coding Trace

benchmark 使用了一个模拟 coding 工作负载的 trace。我们通过提高原始 [Mooncake conversation trace](https://github.com/kvcache-ai/Mooncake/blob/main/FAST25-release/traces/conversation_trace.jsonl) 的输入序列长度和前缀复用率来合成该 trace。

要复现 benchmark，请在 Mooncake 的 `conversation_trace.jsonl` 上运行 Dynamo 的 [prefix data generator 工具](https://github.com/ai-dynamo/dynamo/tree/main/benchmarks/prefix_data_generator)：
```bash
datagen synthesize \
    --input-file conversation_trace.jsonl \
    --prefix-len-multiplier 16 \
    --prompt-len-multiplier 10 \
    --max-isl 110000 \
    --num-requests 10000
# synthesizes `conversation_trace_synth_16.00x1+10.00_speedup1_maxisl110000.jsonl`
```

我们这条 trace 的 ISL/OSL/缓存命中率统计如下。

<details>
<summary>数据集统计：基于 Mooncake 的合成 Trace</summary>

```
============================================================
  DATASET ANALYSIS: Mooncake-based Synthetic Trace
  ============================================================
  OVERVIEW
  ----------------------------------------
    Total Requests:      10,000
    Unique Hash Blocks:  430,838
    Total Hash Blocks:   770,934
  INPUT SEQUENCE LENGTH (ISL)
  ----------------------------------------
    Average:             39,186 tokens
    Maximum:             109,459 tokens
    Minimum:             12,801 tokens
  OUTPUT SEQUENCE LENGTH (OSL)
  ----------------------------------------
    Average:             344 tokens
    Maximum:             2,000 tokens
    Minimum:             1 tokens
  KV CACHE / PREFIX REUSE
  ----------------------------------------
    Block-level Hit Rate: 44.1%
    Token-level Hit Rate: 44.0%
    Avg Context (shared): 22,400 tokens/req
    Avg Unique Prompt:    16,786 tokens/req
    Shared Prefix Ratio:  57.2%
  ============================================================

  Summary:
  • ~44% KV cache hit rate (block/token level) based on hash_id overlap across requests
  • ~57% of input tokens come from shared context prefixes
  • Long-context workload: avg 39K input tokens, up to 109K max
```

</details>


## 前置条件

1. **已安装 Dynamo Platform** —— 参见 [Kubernetes 部署指南](../../docs/kubernetes/README.md)
2. **8 节点共 32× GB200 GPU**
3. **HuggingFace token** 已配置：
   ```bash
   export NAMESPACE=your-namespace
   kubectl create secret generic hf-token-secret \
     --from-literal=HF_TOKEN="your-token" \
     -n ${NAMESPACE}
   ```

## 快速开始

### 1. 创建存储

> **注意：** 请先编辑 `model-cache/model-cache.yaml`，将 `storageClassName` 改为与你的集群匹配的值（运行 `kubectl get storageclass` 可以查看可用选项）。

```bash
kubectl apply -f model-cache/model-cache.yaml -n ${NAMESPACE}
```

### 2. 配置 K8 benchmark 环境
对于多节点 Kubernetes 部署，集群可能要求在你的命名空间中存在一个 ComputeDomain，使 DRA 调度器能将多个 worker pod 共置（co-locate）在 MNNVL 互联的节点上（否则跨节点的 GPU peer memory 访问会失败）。
```bash
kubectl apply -f model-cache/compute-domain.yaml -n ${NAMESPACE}
```
确保对该文件做的任何名称修改也同步到部署 yaml 中的 `extraPodSpec.resourceClaims` 与 `mainContainer.resources.claims`。


### 3. 准备模型与数据
我们使用 NVIDIA 官方的 NVFP4 量化 checkpoint（[Huggingface](https://huggingface.co/nvidia/DeepSeek-V3.2-NVFP4)）。将其复制到 PVC 存储中：

```bash
kubectl apply -f model-cache/model-download.yaml -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=600s
```
类似地，把用于 benchmark 的 trace 文件也复制进 PVC：
```bash
# conversation_trace_synth_16.00x1+10.00_speedup1_maxisl110000.jsonl in our case
kubectl cp <local_trace.jsonl> your-namespace/<helper-pod>:/model-cache/traces/
```

### 4. 部署与 benchmark

**方案 A：聚合（轮询基线）**

```bash
# Deploy
kubectl apply -f trtllm/agg-round-robin/deploy.yaml -n ${NAMESPACE}

# Wait for ready
kubectl wait --for=condition=ready pod -l nvidia.com/dynamo-graph-deployment-name=agg-round-robin-dsv32-nvfp4 \
  -n ${NAMESPACE} --timeout=1200s

# Run benchmark
kubectl apply -f trtllm/agg-round-robin/perf.yaml -n ${NAMESPACE}
```

**方案 B：解耦（KV 感知路由）**

```bash
# Deploy
kubectl apply -f trtllm/disagg-kv-router/deploy.yaml -n ${NAMESPACE}

# Wait for ready
kubectl wait --for=condition=ready pod -l nvidia.com/dynamo-graph-deployment-name=disagg-kv-dsv32-nvfp4 \
  -n ${NAMESPACE} --timeout=1200s

# Run benchmark
kubectl apply -f trtllm/disagg-kv-router/perf.yaml -n ${NAMESPACE}
```

### 4. 监控 benchmark 进度

benchmark 运行在 tmux 会话中，便于监控：

```bash
# Find the benchmark pod
kubectl get pods -n ${NAMESPACE} | grep benchmark

# Attach to the tmux session to see intermediate results
kubectl exec -it -n ${NAMESPACE} <benchmark-pod-name> -- tmux a -t benchmark

# Detach from tmux: Ctrl+B, then D
```

### 5. 查看结果

结果保存到 `perf-cache` PVC：

```bash
# Check artifact directory
kubectl exec -it -n ${NAMESPACE} <benchmark-pod-name> -- ls -la /perf-cache/artifacts/

# Copy results to local machine
kubectl cp ${NAMESPACE}/<benchmark-pod-name>:/perf-cache/artifacts ./benchmark-results
```

## 预期结果

由于 benchmark 使用 `--fixed-schedule`（按原始时间戳重放请求），**吞吐指标由 trace 决定**——我们对比的是时延指标：

| 指标 | 重要性 |
|--------|----------------|
| **TTFT**（Time to First Token） | KV 感知路由通过前缀缓存命中减少预填充计算 |
| **ITL**（Inter-Token Latency） | 解耦服务将解码与预填充的相互干扰隔离开 |
| **总请求时延** | 上述两项优化的综合收益 |

在生产场景下，我们还可以用 **goodput**（满足预设服务级别协议（SLA）的请求速率）来评估部署。本实验的 SLA 设置为 TTFT=20s，ITL=50ms。

## 清理

```bash
# Delete benchmark pods
kubectl delete job agg-round-robin-dsv32-nvfp4-bench disagg-kv-dsv32-nvfp4-bench -n ${NAMESPACE}

# Delete deployments
kubectl delete dynamographdeployment agg-round-robin-dsv32-nvfp4 -n ${NAMESPACE}
kubectl delete dynamographdeployment disagg-kv-dsv32-nvfp4 -n ${NAMESPACE}
```

## 参考资料

- [Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving](https://github.com/kvcache-ai/Mooncake) —— FAST25 论文与 trace 数据
- [Optimizing DeepSeek-V3.2 on NVIDIA Blackwell GPUs](https://nvidia.github.io/TensorRT-LLM/blogs/tech_blog/blog15_Optimizing_DeepSeek_V32_on_NVIDIA_Blackwell_GPUs.html) —— TRTLLM 技术博客，介绍 DSV3.2 在 GB200 上的可用优化
