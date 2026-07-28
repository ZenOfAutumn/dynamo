# LMCache Dynamo MMLU 测试套件

## 概述
通过比较启用与未启用 LMCache 时的 MMLU 基准测试结果，验证 Dynamo 与 LMCache 集成的正确性。

## 测试原理
在两种配置下比较 MMLU 测试结果：
- **基线测试**：未启用 LMCache 的 Dynamo
- **LMCache 测试**：启用 LMCache 的 Dynamo

如果两种配置产生相同的推理（inference）结果，则验证 LMCache 功能正确。

## 快速开始

### 前置条件
1. 确保 dynamo 及其依赖已正确安装（即 nats 与 etcd 已运行）
2. 将 MMLU 数据集下载到 `data` 目录
3. 确保可以访问 HuggingFace 模型

### 下载 MMLU 数据集

```bash
cd ./tests/lmcache

# Auto-download and organize data
python3 download_mmlu.py
```

### 运行单模型测试
修改脚本中的模型名以测试其他模型。
```bash
cd ./tests/lmcache

# 1. Baseline test (without LMCache)
./deploy-baseline-dynamo.sh Qwen/Qwen3-0.6B
# Wait for model to load, then run test in another terminal:
python3 mmlu-baseline-dynamo.py --model Qwen/Qwen3-0.6B --number-of-subjects 15
# Stop services with Ctrl+C in the deploy script terminal

# 2. LMCache test (with LMCache enabled)
./deploy-lmcache_enabled-dynamo.sh Qwen/Qwen3-0.6B
# Wait for model to load, then run test in another terminal:
python3 mmlu-lmcache_enabled-dynamo.py --model Qwen/Qwen3-0.6B --number-of-subjects 15
# Stop services with Ctrl+C in the deploy script terminal

# 3. Compare results
python3 summarize_scores_dynamo.py
```

## 文件说明

### 部署脚本
- **`deploy-baseline-dynamo.sh`**：部署未启用 LMCache 的 Dynamo（基线）
- **`deploy-lmcache_enabled-dynamo.sh`**：部署启用 LMCache 的 Dynamo（测试）

### 测试脚本
- **`mmlu-baseline-dynamo.py`**：在基线 Dynamo 上运行 MMLU 测试
- **`mmlu-lmcache_enabled-dynamo.py`**：在启用 LMCache 的 Dynamo 上运行 MMLU 测试
- **`summarize_scores_dynamo.py`**：比较并分析测试结果

## 架构差异

### 基线架构（deploy-baseline-dynamo.sh）
```
HTTP Request → Dynamo Ingress(8000) → Dynamo Worker → Direct Inference
```

### LMCache 架构（deploy-lmcache_enabled-dynamo.sh）
```
HTTP Request → Dynamo Ingress(8000) → Dynamo Worker → LMCache-enabled Inference
Environment:LMCACHE_CHUNK_SIZE=256
            LMCACHE_LOCAL_CPU=True
            LMCACHE_MAX_LOCAL_CPU_SIZE=1.0
```

## API 格式

测试脚本使用 Dynamo 的 Chat Completions API：

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": Qwen/Qwen3-0.6B,
    "messages": [{"role": "user", "content": "question content"}],
    "temperature": 0,
    "max_tokens": 3,
    "stream": false,
    "seed": 42
  }'
```


## 结果解读

测试完成后会生成以下文件：
- `dynamo-baseline-{model_name}.jsonl`：基线测试结果
- `dynamo-lmcache-{model_name}.jsonl`：LMCache 测试结果

如果两个结果文件中的准确率非常接近（差异 < 1%），则表明 LMCache 功能正确。

## 注意事项

1. **确定性保证**：所有测试均使用相同的随机种子（42）和零温度，以确保结果可复现
2. **前置条件**：确保 nats 和 etcd 正在运行。
3. **顺序执行**：必须先停止第一项测试再启动第二项，以避免端口冲突
