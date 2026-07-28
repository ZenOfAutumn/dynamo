# TensorRT-LLM 引擎配置（Engine Configurations）

本目录包含适用于多种模型部署的 TensorRT-LLM 引擎配置文件。


## 用法

可将这些 YAML 配置文件通过 `--extra-engine-args` 参数传递给 TensorRT-LLM Worker：

```bash
python3 -m dynamo.trtllm \
    --extra-engine-args "${ENGINE_ARGS}" \
    ...
```

其中 `ENGINE_ARGS` 指向本目录下的某个配置文件。

## 配置类型

### 聚合（Aggregated，agg/）
将预填充（prefill）与解码（decode）合并执行的单节点配置：
- **simple/**：基础聚合部署
- **mtp/**：多 token 预测（multi-token prediction）配置
- **wide_ep/**：宽专家并行（wide expert parallel）配置

### 解耦（Disaggregated，disagg/）
预填充与解码 Worker 各自独立的配置：
- **simple/**：基础的预填充/解码拆分
- **mtp/**：多 token 预测且预填充/解码分离
- **wide_ep/**：带专家负载均衡器（expert load balancer）的宽专家并行

## 关键配置参数

- **并行**：`tensor_parallel_size`、`moe_expert_parallel_size`、`pipeline_parallel_size`
- **显存**：`kv_cache_config.free_gpu_memory_fraction`、`kv_cache_config.dtype`
- **批处理**：`max_batch_size`、`max_num_tokens`、`max_seq_len`
- **调度**：`disable_overlap_scheduler`、`cuda_graph_config`

## 注意事项

- 对于解耦部署，请确保预填充与解码配置中的 `kv_cache_config.dtype` 保持一致
- WideEP 配置需要专家负载均衡器配置（`eplb.yaml`）
- 请根据工作负载与 attention DP 设置调整 `free_gpu_memory_fraction`
