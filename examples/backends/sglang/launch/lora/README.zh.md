# 在 SGLang 后端上使用 LoRA

完整的 LoRA 集成指南（环境准备、用法、API 参考、问题排查）请参见 [共享 LoRA 指南](../../../../common/lora.md)。

## 快速开始

```bash
./setup_minio.sh    # 启动 MinIO，下载并上传 LoRA
./agg_lora.sh       # 启动 SGLang 前端（frontend）+ 带 LoRA 的 worker
```

## SGLang 相关注意事项

- 启动脚本默认使用 `--lora-target-modules all` 与 `--max-lora-rank 64`
- 可以通过环境变量进行覆盖：`MODEL`、`LORA_NAME`、`DYN_SYSTEM_PORT`、`DYN_HTTP_PORT`
- SGLang 的 LoRA 加载会通过 `engine.tokenizer_manager.load_lora_adapter()` 进行
