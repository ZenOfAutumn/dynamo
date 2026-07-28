---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 扩散（Diffusion）
---

Dynamo SGLang 支持三种基于扩散的生成：**LLM 扩散**（通过迭代细化生成文本）、**图像扩散**（文生图）以及**视频生成**（文生视频）。每种使用不同的 worker 标志和 handler，但都集成到 SGLang 的 `DiffGenerator`。

## 概述

| 类型             | Worker 标志                 | API 端点                              |
| ---------------- | --------------------------- | ----------------------------------------- |
| LLM Diffusion    | `--dllm-algorithm <algo>`   | `/v1/chat/completions`、`/v1/completions` |
| Image Diffusion  | `--image-diffusion-worker`  | `/v1/images/generations`                  |
| Video Generation | `--video-generation-worker` | `/v1/videos`                              |

<Note>
如果启动时遇到 CuDNN 版本不匹配错误（`cuDNN frontend 1.8.1 requires cuDNN lib >= 9.5.0`），请在启动前设置 `SGLANG_DISABLE_CUDNN_CHECK=1`。当 PyTorch 自带的 CuDNN 版本旧于 SGLang 所需的 Conv3d 操作版本时常见。
</Note>

## LLM 扩散

扩散语言模型通过迭代细化而非自回归逐 token 生成来产生文本。模型从带 mask 的 token 开始，逐步将其替换为预测，每一步细化置信度较低的 token。

LLM 扩散是自动检测的：当设置了 `--dllm-algorithm` 时，worker 会自动使用 `DiffusionWorkerHandler`，无需额外标志。关于扩散算法的更多细节，参见 [SGLang Diffusion Language Models 文档](https://github.com/sgl-project/sglang/blob/main/docs/supported_models/text_generation/diffusion_language_models.md)。

### 启动

```bash
cd $DYNAMO_HOME/examples/backends/sglang
./launch/diffusion_llada.sh
```

配置选项参见[启动脚本](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/sglang/launch/diffusion_llada.sh)。

### 测试

```bash
curl -X POST http://localhost:8001/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "inclusionAI/LLaDA2.0-mini-preview",
    "messages": [{"role": "user", "content": "Explain why Roger Federer is considered one of the greatest tennis players of all time"}],
    "temperature": 0.7,
    "max_tokens": 512
  }'
```

## 图像扩散

图像扩散 worker 使用 SGLang 的 `DiffGenerator` 从文本 prompt 生成图像。生成的图像以 URL（使用 `--media-output-fs-url` 存储时）或 base64 数据返回，遵循 OpenAI 兼容响应格式。

### 启动

```bash
cd $DYNAMO_HOME/examples/backends/sglang
./launch/image_diffusion.sh
```

支持本地存储（`--fs-url file:///tmp/images`）与 S3（`--fs-url s3://bucket`）。通过 `--http-url` 设置图像访问的基础 URL。所有配置选项参见启动脚本。

### 测试

```bash
curl http://localhost:8000/v1/images/generations \
  -H "Content-Type: application/json" \
  -d '{
    "model": "black-forest-labs/FLUX.1-dev",
    "prompt": "Explain why Roger Federer is considered one of the greatest tennis players of all time",
    "size": "1024x1024",
    "response_format": "url",
    "nvext": {
      "num_inference_steps": 15
    }
  }'
```

## 视频生成

视频生成 worker 使用 SGLang 的 `DiffGenerator` 从文本或图像 prompt 生成视频，并进行帧到视频的编码。支持文生视频（T2V）与图生视频（I2V）工作流。

### 启动

```bash
cd $DYNAMO_HOME/examples/backends/sglang
./launch/text-to-video-diffusion.sh
```

使用 `--wan-size 1b`（默认，1 GPU）或 `--wan-size 14b`（2 GPU）。所有配置选项参见启动脚本。

### 测试

```bash
curl http://localhost:8000/v1/videos \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Roger Federer winning his 19th grand slam",
    "model": "Wan-AI/Wan2.1-T2V-1.3B-Diffusers",
    "seconds": 2,
    "size": "832x480",
    "response_format": "url",
    "nvext": {
      "fps": 8,
      "num_frames": 17,
      "num_inference_steps": 50
    }
  }'
```

## 另请参阅

- **[示例](sglang-examples.md)**：所有部署模式的启动脚本
- **[参考指南](sglang-reference-guide.md)**：worker 类型与参数参考
- **[SGLang Diffusion LMs（上游）](https://github.com/sgl-project/sglang/blob/main/docs/supported_models/text_generation/diffusion_language_models.md)**：SGLang 扩散文档
