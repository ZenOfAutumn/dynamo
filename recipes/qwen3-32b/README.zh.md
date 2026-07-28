# Qwen3-32B：聚合轮询 vs 解耦 KV 路由 对比

本配方展示了在使用 [Mooncake FAST25 论文](https://github.com/kvcache-ai/Mooncake)
所提供的真实对话轨迹数据集上，**聚合（aggregated，轮询）** 与
**解耦（disaggregated，KV-aware）** 路由之间的性能差异。

## 结果

https://github.com/user-attachments/assets/c425002b-4459-47c4-bfca-fd1e2620500c


## 实验概览

我们在 **2 节点共 16 张 H200 GPU** 上对比两种部署模式：

| 模式 | 路由 | 配置 |
|------|---------|---------------|
| **聚合（Aggregated）** | 轮询（Round-robin） | 8x TP2 worker |
| **解耦（Disaggregated）** | KV-aware | 6x prefill + 2x decode（TP2） |

## 数据集：Mooncake 对话轨迹

本基准测试使用一份具有显著前缀共享潜力的生产对话轨迹：

| 指标 | 取值 |
|--------|-------|
| 请求数 | 12,031 条，约 59 分钟（3.4 req/s） |
| 输入 token/秒 | 40,937 tok/s |
| 输入长度 | 平均 12,035 tokens（范围：891 - 126,195） |
| 输出长度 | 平均 343 tokens |

**缓存复用分析：**

| 指标 | 取值 | 含义 |
|--------|-------|------------------|
| 块复用率（Blocks reused） | 24.2% | 在 182,790 个唯一 block 中，44,144 个出现在多于一个请求中 |
| 缓存效率（Cache efficiency） | 36.64% | 在 288,500 次 block 引用中，105,710 次为重复（在无限缓存假设下可复用） |

*为何不同：* "块复用" 统计的是出现重复的唯一 block 数量，忽略重复频率；"缓存效率"
按出现频次加权 —— 一个被复用 12,031 次的 block 比只被复用 1 次的 block 贡献更大。

该负载非常适合 KV-aware 路由 —— 36.64% 的缓存效率意味着请求可以被路由到已经缓存
了相关 KV block 的 worker 上，从而显著降低 TTFT。

## 前置条件

1. **已安装 Dynamo Platform** —— 参见 [Kubernetes 部署指南](../../docs/kubernetes/README.md)
2. **2 节点共 16 张 H200 GPU**
3. **配置好 HuggingFace token**：
   ```bash
   export NAMESPACE=your-namespace
   kubectl create secret generic hf-token-secret \
     --from-literal=HF_TOKEN="your-token" \
     -n ${NAMESPACE}
   ```

## 快速开始

### 1. 创建存储

> **注意：** 请先编辑 `model-cache/cache.yaml`，将 `storageClassName` 改为你
> 集群上可用的值（运行 `kubectl get storageclass` 查看）。

```bash
kubectl apply -f model-cache/cache.yaml -n ${NAMESPACE}
```

### 2. 下载模型

```bash
kubectl apply -f model-cache/model-download.yaml -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=600s
```

### 3. 部署与基准测试

**方案 A：聚合（轮询基线）**

```bash
# 部署
kubectl apply -f vllm/agg-round-robin/deploy.yaml -n ${NAMESPACE}

# 等待就绪
kubectl wait --for=condition=ready pod -l nvidia.com/dynamo-graph-deployment-name=agg-8xtp2 \
  -n ${NAMESPACE} --timeout=1200s

# 运行基准测试
kubectl apply -f vllm/agg-round-robin/perf.yaml -n ${NAMESPACE}
```

**方案 B：解耦（KV-aware 路由）**

```bash
# 部署
kubectl apply -f vllm/disagg-kv-router/deploy.yaml -n ${NAMESPACE}

# 等待就绪
kubectl wait --for=condition=ready pod -l nvidia.com/dynamo-graph-deployment-name=disagg-router-6p-2d \
  -n ${NAMESPACE} --timeout=1200s

# 运行基准测试
kubectl apply -f vllm/disagg-kv-router/perf.yaml -n ${NAMESPACE}
```

### 4. 监控基准测试进度

基准测试运行在 tmux 会话中以方便监控：

```bash
# 找到 benchmark pod
kubectl get pods -n ${NAMESPACE} | grep benchmark

# attach 到 tmux 会话查看中间结果
kubectl exec -it -n ${NAMESPACE} <benchmark-pod-name> -- tmux a -t benchmark

# detach tmux：Ctrl+B，然后按 D
```

### 5. 查看结果

结果保存在 `perf-cache` PVC 中：

```bash
# 检查产物目录
kubectl exec -it -n ${NAMESPACE} <benchmark-pod-name> -- ls -la /perf-cache/artifacts/

# 拷贝结果到本地
kubectl cp ${NAMESPACE}/<benchmark-pod-name>:/perf-cache/artifacts ./benchmark-results
```

## 预期结果

由于基准测试使用 `--fixed-schedule`（按原始时间戳回放请求），**吞吐相关指标由
轨迹本身锁定** —— 我们真正要比较的是延迟相关指标：

| 指标 | 为何重要 |
|--------|----------------|
| **TTFT**（首 token 时间） | KV-aware 路由通过前缀缓存命中减少 prefill 计算量 |
| **ITL**（token 间延迟） | 解耦服务能够把解码（decode）从预填充（prefill）干扰中隔离开 |
| **整体请求延迟** | 上述两项优化的综合收益 |

**为什么 解耦 + KV-aware 路由 对该负载有帮助：**

1. **KV-aware 路由** 利用 36% 的缓存效率，把请求路由到已经缓存了相关 KV block
   的 worker 上，从而减少冗余的 prefill 计算并降低 TTFT。

2. **解耦服务** 将 prefill 与 decode worker 分离。考虑到平均输入长度约 12K
   tokens、平均输出约 343 tokens，把 decode 放在专用 worker 上，可以避免
   "prefill 注入"（prefill injection）现象 —— 即一个新到的长上下文请求中断了
   正在进行的 decode，从而引发 ITL 飙升。

## 清理

```bash
# 删除 benchmark pod
kubectl delete pod -l app=benchmark -n ${NAMESPACE}

# 删除部署
kubectl delete dynamographdeployment agg-8xtp2 -n ${NAMESPACE}
kubectl delete dynamographdeployment disagg-router-6p-2d -n ${NAMESPACE}
```

## 参考

- [Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving](https://github.com/kvcache-ai/Mooncake) —— FAST25 论文与轨迹数据
