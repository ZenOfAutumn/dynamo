#  Recipes 贡献指南

在新增模型 recipe 时，请确保它们遵循以下标准结构：
```text
<model-name>/
├── model-cache/
│   ├── model-cache.yaml
│   └── model-download.yaml
├── <framework>/
│   └── <deployment-mode>/
│       ├── deploy.yaml
│       └── perf.yaml (optional)
└── README.md (optional)
```

## 校验
`run.sh` 脚本要求严格遵循上述目录结构，并会在部署前校验目录与文件是否存在：
- 模型目录存在于 `recipes/<model>/`
- framework 是受支持的框架之一（vllm、sglang、trtllm）
- framework 目录存在于 `recipes/<model>/<framework>/`
- 部署目录存在于 `recipes/<model>/<framework>/<deployment>/`
- 部署目录中存在所需文件（`deploy.yaml`）
- 如果存在 `perf.yaml`，性能基准会被自动执行
