<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# MM Router Worker

面向 TRT-LLM 后端（backend）的、具备多模态感知能力的 KV 缓存（KV cache）路由 worker。

## 概览

该 worker 位于 Dynamo 前端（frontend）与 TRT-LLM worker 之间，提供具备多模态感知能力的 KV 缓存路由：

1. **接收**前端发来的 OpenAI 格式请求
2. **下载**图像并计算 `mm_hash`（仅用于路由决策）
3. **构造**多模态路由元数据（`mm_routing_info`）
4. **使用** KvRouter 选择并将请求路由（routing）到最佳的 TRT-LLM worker
5. **流式**将响应返回前端

## 架构

```
Frontend (standard)      MM Router Worker (this)        TRT-LLM Worker (standard)
┌──────────────┐        ┌─────────────────────┐        ┌───────────────────┐
│              │───────>│ 1. Download images  │───────>│ python -m         │
│  round-robin │        │ 2. Compute mm_hash  │        │ dynamo.trtllm     │
│  to mm_router│<───────│ 3. Build routing    │<───────│ --modality mm     │
└──────────────┘        │ 4. KvRouter route   │        │ (processes images)│
                        └─────────────────────┘        └───────────────────┘
                                  │
                                  │ Subscribe KV events
                                  v
                            ┌──────────┐
                            │   NATS   │
                            └──────────┘
```

**注意**：图像会被下载两次 —— 一次在 MM Router 中（用于计算 mm_hash），一次在 TRT-LLM worker 中（用于实际处理）。这种设计省去了对 tensor 进行序列化的复杂度。

## 用法

### 快速开始

```bash
# Start all services
./launch.sh
```

### 手动启动

```bash
# 1. Start etcd and NATS
docker compose -f deploy/docker-compose.yml up -d

# 2. Start TRT-LLM worker(s)
python -m dynamo.trtllm \
    --model Qwen/Qwen2-VL-2B-Instruct \
    --namespace default \
    --component trtllm \
    --endpoint generate \
    --modality multimodal \
    --publish-events-and-metrics &

# 3. Start MM Router Worker
python -m examples.backends.trtllm.mm_router_worker \
    --model Qwen/Qwen2-VL-2B-Instruct \
    --model-type qwen2_vl \
    --namespace default \
    --component mm_router \
    --endpoint generate \
    --downstream-component trtllm \
    --downstream-endpoint generate &

# 4. Start Frontend
python -m dynamo.frontend \
    --http-port 8000 \
    --router-mode round-robin
```

### 测试请求

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2-VL-2B-Instruct",
    "messages": [{
      "role": "user",
      "content": [
        {"type": "text", "text": "Describe this image"},
        {"type": "image_url", "image_url": {"url": "https://example.com/image.jpg"}}
      ]
    }],
    "max_tokens": 100
  }'
```

## 配置

| 参数 | 默认值 | 描述 |
|----------|---------|-------------|
| `--model` | `Qwen/Qwen2-VL-2B-Instruct` | 模型路径或 HuggingFace ID |
| `--model-type` | `qwen2_vl` | TRT-LLM 用于多模态加载器的模型类型 |
| `--block-size` | `32` | KV 缓存块大小 |
| `--namespace` | `default` | Dynamo 命名空间 |
| `--component` | `mm_router` | 当前 worker 的组件名 |
| `--endpoint` | `generate` | 当前 worker 的 endpoint 名 |
| `--downstream-component` | `trtllm` | TRT-LLM worker 的组件名 |
| `--downstream-endpoint` | `generate` | TRT-LLM worker 的 endpoint 名 |

## 工作原理

### MM Hash 计算

worker 使用 TRT-LLM 的 `apply_mm_hashes()` 函数为每张图像的 tensor 表示计算哈希。
该哈希被纳入 block hash 的计算，确保：

- 相同图像 = 相同 mm_hash = 相同 block hash = 命中缓存
- 不同图像 = 不同 mm_hash = 不同 block hash = 不会假命中

### KV 感知路由（KV-Aware Routing）

worker 使用 `KvRouter.generate(...)` 并附带显式的多模态路由提示。
当请求到来时：

1. 为该请求构造路由 token（`routing_token_ids`）
2. 构造 `block_mm_infos`，携带每个 block 的图像 `mm_hash` 元数据
3. 把两者作为 `mm_routing_info` 传给 `KvRouter.generate(...)`
4. KvRouter 内部计算 overlap，并把请求路由到最优 worker

### Block MM Info 结构

对于每个包含图像 token 的 block，构造 `block_mm_infos`：

```python
block_mm_infos = [
    None,  # Block 0: no image
    {"mm_objects": [{"mm_hash": 12345, "offsets": [[32, 128]]}]},  # Block 1: has image
    {"mm_objects": [{"mm_hash": 12345, "offsets": [[32, 128]]}]},  # Block 2: same image
    None,  # Block 3: no image
]
```

它会被一并放入 `mm_routing_info`，从而让 KvRouter 能够计算多模态感知的 overlap。

## 文件

| 文件 | 描述 |
|------|-------------|
| `mm_router_worker.py` | 主 worker，含 `@dynamo_worker()` |
| `handler.py` | `MMRouterHandler` —— 路由逻辑 |
| `mm_processor.py` | 多模态处理工具 |
| `__main__.py` | 入口 |
| `launch.sh` | 启动脚本 |

## 依赖

- `tensorrt_llm >= 1.3.0rc5` —— 当前 `apply_mm_hashes()` 元组返回契约（`(mm_hashes_by_modality, uuids)`）所需，被本 worker 的路由哈希提取路径使用。
- `transformers` —— 用于 `AutoProcessor`
- `dynamo` —— 用于 runtime 与 KvRouter

## 已知限制

- **Qwen2-VL 专属**：`mm_processor.py` 中的 `_compute_tokens_per_image()` 逻辑当前只支持 `qwen2_vl` 模型类型。要支持其他多模态模型，需要补充其各自的 visual token 计算逻辑。
