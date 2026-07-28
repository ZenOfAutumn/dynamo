# 在 vLLM 后端上使用 LoRA

完整的 LoRA 集成指南（包含环境配置、使用方法、API 参考与故障排查），请参阅[共享 LoRA 指南](../../../../common/lora.md)。

## 快速开始

```bash
./setup_minio.sh    # 启动 MinIO，下载并上传 LoRA
./agg_lora.sh       # 启动 vLLM 前端（frontend）+ Worker（带 LoRA）
```

## vLLM 专属说明

- 默认 `--max-lora-rank 64`（与 SGLang 一致）
- 可通过环境变量覆盖：`MODEL`、`LORA_NAME`、`MAX_MODEL_LEN`、`MAX_CONCURRENT_SEQS`

### KV 缓存感知路由（KV-Aware Routing，2 GPU）

```bash
./agg_lora_router.sh
```

在一个 KV 感知路由（router）后面启动两个 vLLM Worker。将同一个 LoRA 同时加载到两个 Worker（端口 8081 与 8082），随后请求会按 KV 缓存（KV cache）亲和性进行路由，从而提升缓存命中率。
