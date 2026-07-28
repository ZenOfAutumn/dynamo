# Qwen3-VL-30B-A3B-Instruct-FP8：嵌入缓存（Embedding Cache）开关对比

本 recipe 演示在多模态负载下启用嵌入缓存（embedding cache）所带来的性能差异。它包含构造具有用户自定义图像复用率的人造数据集的指引，以及面向生产的 `Qwen/Qwen3-VL-30B-A3B-Instruct-FP8` 部署示例。

## 结果

| 指标               | Cache ON | Cache OFF | Delta  |
|----------------------|---------:|----------:|-------:|
| Output TPS (tok/s)           |   3575.6 |    3072.3 | +16.4% |
| TTFT avg (ms)        |    526.0 |     727.5 | -27.7% |
| TTFT p50 (ms)        |    356.8 |     510.8 | -30.1% |
| ITL avg (ms)         |     14.1 |      15.5 |  -8.8% |
| Req Latency avg (ms) |   2630.0 |    3035.7 | -13.4% |

**在 GB200 上以 vLLM 后端的单聚合（aggregated）副本运行 `Qwen3-VL-30B-A3B-Instruct-FP8` 时，启用嵌入缓存平均带来 +16% 的吞吐提升、-28% 的 TTFT 降低以及 -13% 的请求时延降低。**

## 前置条件

要复现表格中的结果，需要满足：

1. **已安装 Dynamo Platform** —— 参见 [Kubernetes 部署指南](../../docs/kubernetes/README.md)
2. **GB200**
3. **HuggingFace token** 已配置：
   ```bash
   export NAMESPACE=your-namespace
   kubectl create secret generic hf-token-secret \
     --from-literal=HF_TOKEN="your-token" \
     -n ${NAMESPACE}
   ```

## 数据集生成

`data-gen/generate-datasets-job.yaml` 会生成一个图像重叠率为 80% 的合成文本 + 图像数据集。脚本通过控制"总槽位数（total slots）"和"图像池（image pool）"来实现。

总槽位数计算公式为 `num_requests*images/request`，表示 benchmark 总共会迭代多少张图。图像池则是 benchmark 可以从中挑选图像附加到请求上的图像数量。

`data-gen/generate-datasets-job.yaml` 脚本生成 1000 条请求，每条请求 1 张图像，图像池为 200。每个请求从池中无放回地选取图像，池中图像用完后再从头循环。因此 1000 条请求的前 200 条会带有唯一图像，而其余的 800 条则会重复推理引擎此前已经见过的图像。关于数据生成的更多细节，请参考 jsonl [文档](https://github.com/ai-dynamo/dynamo/tree/main/benchmarks/multimodal/jsonl)。

每个数据集都硬编码为包含 400 个 token 的用户输入文本。

要生成数据集，运行：

```bash
kubectl apply -f data-gen/generate-datasets-job.yaml -n ${NAMESPACE}
```

## 注意事项

1. 由于嵌入缓存可能采用 LRU 等驱逐策略，无法通过数据集精确控制命中率；但减小图像池相对于请求数的比例，可以按比例提高重复图像与缓存命中的概率。增大嵌入缓存容量也能提升命中率，因为驱逐会更少。

**2. 聚合（agg）模式的嵌入缓存使用 vLLM 原生的 `ec_both` ECConnector 角色，vLLM 0.17+ 支持，无需打补丁。详情参见 [multimodal-vllm.md](https://github.com/ai-dynamo/dynamo/blob/main/docs/features/multimodal/multimodal-vllm.md#embedding-cache)。**

3. 运行前请替换 `*.yaml` 中的占位符：
   - `model-cache/model-cache.yaml` 中的 `storageClassName: "your-storage-class-name"`
   - 所有 `vllm/*/deploy.yaml` 文件中的 `image: <your-dynamo-image>`
   - 安装命令中的 `NAMESPACE=your-namespace` 与 `HF_TOKEN="your-token"`

## 目录结构

本 recipe 包含三个顶层组件：用于 PVC/模型准备的 `model-cache/`、用于数据集生成的 `data-gen/`，以及用于部署和使用 [AIPerf](https://github.com/ai-dynamo/aiperf) 进行 benchmark 的 `vllm/agg-embedding-cache/`。

```text
qwen3-vl-30b/
├── data-gen/
│   └── generate-datasets-job.yaml
├── model-cache/
│   ├── model-cache.yaml
│   └── model-download.yaml
└── vllm/
    └── agg-embedding-cache/
        ├── deploy.yaml
        ├── perf.yaml
        └── run-benchmark.sh
```

`deploy.yaml` 默认设置 `DYN_MULTIMODAL_EMBEDDING_CACHE_GB=10`，对应嵌入缓存**开启**配置。要关闭它，可将该环境变量设为 0。

类似地，每个 `perf.yaml` 暴露了 `CACHE_MODE` 环境变量，用于控制 AIPerf 把结果写入到哪里。根据部署情况设为 `cache_on` 或 `cache_off`。

## 快速开始

### 1. 设置命名空间并创建存储

```bash
export NAMESPACE=your-namespace
kubectl apply -f model-cache/model-cache.yaml -n ${NAMESPACE}
kubectl get pvc -n ${NAMESPACE}
```

### 2. 下载模型并生成数据集

```bash
kubectl apply -f model-cache/model-download.yaml -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=3600s

kubectl apply -f data-gen/generate-datasets-job.yaml -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/qwen3-vl-30b-generate-datasets -n ${NAMESPACE} --timeout=3600s
kubectl logs job/qwen3-vl-30b-generate-datasets -n ${NAMESPACE}
```

### 3. 部署并 benchmark（`agg-embedding-cache`）

```bash
# deploy.yaml defaults to cache ON (DYN_MULTIMODAL_EMBEDDING_CACHE_GB=10)
kubectl apply -f vllm/agg-embedding-cache/deploy.yaml -n ${NAMESPACE}
kubectl wait --for=condition=Ready dynamographdeployment/qwen3-vl-agg -n ${NAMESPACE} --timeout=900s

kubectl apply -f vllm/agg-embedding-cache/perf.yaml -n ${NAMESPACE}
kubectl wait --for=condition=Ready pod/qwen3-vl-agg-benchmark -n ${NAMESPACE} --timeout=300s
```

可选：要运行 cache OFF，请在应用前将 `vllm/agg-embedding-cache/deploy.yaml` 中的 `DYN_MULTIMODAL_EMBEDDING_CACHE_GB` 改为 `0`，并在 `vllm/agg-embedding-cache/perf.yaml` 中将 `CACHE_MODE` 改为 `cache_off`。

### 4. 监控 benchmark 进度

```bash
kubectl get pods -n ${NAMESPACE} -l app=benchmark

# Follow benchmark logs in real time
kubectl logs -f qwen3-vl-agg-benchmark -n ${NAMESPACE}

# Wait for completion
kubectl wait --for=jsonpath='{.status.phase}'=Succeeded pod/qwen3-vl-agg-benchmark -n ${NAMESPACE} --timeout=7200s
```

等待出现 `Run complete. Artifacts in /perf-cache/artifacts/qwen3_vl_30b_embedding_cache/agg/<cache_mode>`。

`vllm/agg-embedding-cache/run-benchmark.sh` 也作为辅助脚本提供，用于一键启动 cache-on/cache-off 两种运行。
