# Qwen3-32B：聚合（Aggregated）+ KVBM（单 GPU）

`Qwen/Qwen3-32B` 在单 GPU 上的聚合部署，并启用 KV Block Manager
（KVBM）。KVBM 会将冷的 KV 缓存（KV cache）块卸载到主机内存，因此
有效缓存容量可超出 GPU HBM，从而在长 prompt 或重复 prompt 场景下提升
前缀复用命中率，而无需增加 GPU。

## 硬件

- **1x NVIDIA H200（141 GB）或 B200（192 GB）**。Qwen3-32B 的 BF16 权重约 64 GB，
  再加上 KV 缓存与激活，80 GB H100 几乎没有余量，在真实负载下很可能 OOM。
  如果你只有 H100 80 GB，请参见 `../../../qwen3-32b-fp8/` 中的 FP8 变体。
- 节点（node）上 **≥ 约 150 GiB 的主机内存**。`DYN_KVBM_CPU_CACHE_GB=100` 会作为
  page-locked 主机内存固定下来，作为 KVBM 的 G2 层。Worker 声明
  `resources.requests.memory: 150Gi` 与 `resources.limits.memory: 200Gi`
  （100 GiB 的 pinned KV 池 + 约 50 GiB 给 Python、权重加载器工作内存以及
  CUDA/NCCL 缓冲的余量）。如果你提高 `DYN_KVBM_CPU_CACHE_GB`，请按大致相同的增量
  上调这些数值。

## 先决条件

与本目录下的同级 recipe 相同：

1. **已安装 Dynamo 平台** —— 参见 [Kubernetes 部署指南](../../../../docs/kubernetes/README.md)。
2. **已存在 `model-cache` 与 `compilation-cache` PVC** —— 参见
   [`../../model-cache/cache.yaml`](../../model-cache/cache.yaml) 与
   [`../../model-cache/model-download.yaml`](../../model-cache/model-download.yaml)。
3. 命名空间中存在名为 `hf-token-secret` 的 **HuggingFace token Secret**。

```bash
export NAMESPACE=your-namespace
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="your-token" \
  -n ${NAMESPACE}
```

## 部署

```bash
kubectl apply -f deploy.yaml -n ${NAMESPACE}

kubectl wait --for=condition=ready pod \
  -l nvidia.com/dynamo-graph-deployment-name=agg-kvbm-qwen3-32b \
  -n ${NAMESPACE} --timeout=1200s
```

## 验证

```bash
kubectl port-forward svc/agg-kvbm-qwen3-32b-frontend 8000:8000 -n ${NAMESPACE}

curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-32B",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 64
  }'
```

## KVBM 配置

通过 worker 的 `--kv-transfer-config` 选择 connector：

```json
{"kv_connector":"DynamoConnector","kv_role":"kv_both","kv_connector_module_path":"kvbm.vllm_integration.connector"}
```

本 recipe 设置的 worker 环境变量：

| 变量 | 本 recipe 默认值 | 说明 |
|---|---|---|
| `DYN_KVBM_CPU_CACHE_GB` | `100` | 为卸载的 KV 块预留的 CPU 内存。更长上下文或更高复用时可调高；如果改动该值，也请按大致相同的增量同步上调 worker 的 `resources.requests.memory` / `limits.memory`。 |

### （可选）Prometheus 指标（metrics）

指标默认 **关闭**。要暴露指标，请在 `deploy.yaml` 的 worker 的 `env` 与
`mainContainer.ports` 中添加：

```yaml
env:
  - name: DYN_KVBM_METRICS
    value: "true"
  - name: DYN_KVBM_METRICS_PORT
    value: "6880"
ports:
  - name: kvbm
    containerPort: 6880
```

启用后，可抓取 `:6880/metrics` 获取诸如
`kvbm_offload_blocks_d2h`、`kvbm_onboard_blocks_h2d`、`kvbm_matched_tokens`、
`kvbm_host_cache_hit_rate` 以及各路径的传输计数等。如果你运行了
Prometheus Operator，请添加一个 `PodMonitor`，选中带有
`nvidia.com/dynamo-component-type: worker` 标签且端口为 `kvbm` 的 pod。

## 清理

```bash
kubectl delete dynamographdeployment agg-kvbm-qwen3-32b -n ${NAMESPACE}
```
