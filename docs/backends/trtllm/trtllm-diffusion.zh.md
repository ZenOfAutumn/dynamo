---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 视频扩散支持（实验性）
---

有关 TensorRT-LLM 的通用特性与配置，请参阅[参考指南](trtllm-reference-guide.md)。

---

Dynamo 通过 `--modality video_diffusion` 标志支持使用扩散模型生成视频，
并通过 `--modality image_diffusion` 标志支持图像生成。

## 要求

- **带 visual_gen 的 TensorRT-LLM**：`visual_gen` 模块是 TensorRT-LLM 的一部分（`tensorrt_llm._torch.visual_gen`）。请按照[官方说明](https://github.com/NVIDIA/TensorRT-LLM#installation)安装 TensorRT-LLM。
- **带多模态 API 的 dynamo-runtime**：Dynamo 运行时必须支持 `ModelType.Videos` 或 `ModelType.Images`。请确保使用兼容版本。
- **视频扩散：带 ffmpeg 的 imageio**：用于将生成的帧编码为 MP4 视频：
  ```bash
  pip install imageio[ffmpeg]
  ```

## 支持的模型

| Diffusers Pipeline | 描述 | 示例模型 |
|--------------------|-------------|---------------|
| `WanPipeline` | Wan 2.1/2.2 文生视频 | `Wan-AI/Wan2.1-T2V-1.3B-Diffusers` |
| `FluxPipeline` | FLUX 文生图 | `black-forest-labs/FLUX.1-dev` |


Pipeline 类型会从模型的 `model_index.json` 中**自动检测**——无需 `--model-type` 标志。

## 快速开始

### 视频扩散

#### 启动 worker

```bash
python -m dynamo.trtllm \
  --modality video_diffusion \
  --model-path Wan-AI/Wan2.1-T2V-1.3B-Diffusers \
  --media-output-fs-url file:///tmp/dynamo_media
```

#### API 端点

视频生成使用 `/v1/videos` 端点：

```bash
curl -X POST http://localhost:8000/v1/videos \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "A cat playing piano",
    "model": "wan_t2v",
    "seconds": 4,
    "size": "832x480",
    "nvext": {
      "fps": 24
    }
  }'
```

### 图像扩散

#### 启动 worker

```bash
python -m dynamo.trtllm \
  --modality image_diffusion \
  --model-path black-forest-labs/FLUX.1-dev \
  --media-output-fs-url file:///tmp/dynamo_media
```

#### API 端点

图像生成使用 `/v1/images/generations` 端点：

```bash
curl -X POST http://localhost:8000/v1/images/generations \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "A cat playing piano",
    "model": "black-forest-labs/FLUX.1-dev",
    "size": "256x256"
  }'
```

## 配置项

| 标志 | 描述 | 默认值 |
|------|-------------|---------|
| `--media-output-fs-url` | 用于存储生成媒体的文件系统 URL | `file:///tmp/dynamo_media` |
| `--default-height` | 默认图像 / 视频高度 | `480` |
| `--default-width` | 默认图像 / 视频宽度 | `832` |
| `--default-num-frames` | 默认帧数 | `81` |
| `--default-num-images-per-prompt` | 每个 prompt 默认生成的图像数 | `1` |
| `--enable-teacache` | 启用 TeaCache 优化 | `False` |
| `--disable-torch-compile` | 禁用 torch.compile | `False` |

## 限制

- 扩散模型为实验性，不建议用于生产
- 当前版本仅支持文生视频与文生图（图生视频在计划中）
- 需要具备足够 VRAM 的 GPU 运行扩散模型
