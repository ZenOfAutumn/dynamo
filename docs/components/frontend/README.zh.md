---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Frontend
---

Dynamo Frontend 是为 LLM 推理请求提供服务的 API 网关。它提供 OpenAI 兼容的 HTTP 端点和 KServe gRPC 端点，处理请求预处理、路由与响应格式化。

## 功能矩阵

| 特性 | 状态 |
|---------|--------|
| OpenAI Chat Completions API（`/v1/chat/completions`） | ✅ 支持 |
| OpenAI Completions API（`/v1/completions`） | ✅ 支持 |
| OpenAI Embeddings API（`/v1/embeddings`） | ✅ 支持 |
| OpenAI Responses API（`/v1/responses`） | ✅ 支持 |
| OpenAI Models API（`/v1/models`） | ✅ 支持 |
| 图像生成（`/v1/images/generations`） | ✅ 支持 |
| 视频生成（`/v1/videos/generations`） | ✅ 支持 |
| Anthropic Messages API（`/v1/messages`） | 🧪 实验性 |
| KServe gRPC v2 API | ✅ 支持 |
| 流式响应（SSE） | ✅ 支持 |
| 多模型服务 | ✅ 支持 |
| 集成的 KV 感知路由 | ✅ 支持 |
| 工具调用 | ✅ 支持 |
| TLS（HTTPS） | ✅ 支持 |
| Swagger UI（`/docs`） | ✅ 支持 |
| NVIDIA 请求扩展（`nvext`） | ✅ 支持 |

## 快速开始

### 前置条件

- 已安装 Dynamo 平台
- `etcd` 与 `nats-server -js` 已运行
- 至少有一个后端 worker 已注册

### HTTP Frontend

```bash
python -m dynamo.frontend --http-port 8000
```

这会启动一个 OpenAI 兼容的 HTTP 服务器，集成预/后处理与路由。后端调用 `register_model` 时会被自动发现。

前端负责预处理与后处理。为此它需要访问模型的配置文件：`config.json`、`tokenizer.json`、`tokenizer_config.json` 等。它不需要权重。

前端会从 Hugging Face 下载所需文件，无需额外设置。但我们建议搭配 [modelexpress-server](https://github.com/ai-dynamo/modelexpress) 与共享目录（例如 Kubernetes PVC）使用。这样可确保整个集群只下载一次模型。

如果模型不在 Hugging Face 上（例如私有或定制模型），你需要在前端的本地路径上提供这些模型文件，路径与后端一致。后端的 `--model-path <here>` 必须在前端存在并至少包含配置（JSON）文件。

### KServe gRPC Frontend

```bash
python -m dynamo.frontend --kserve-grpc-server
```

KServe 特定的配置与消息格式请参阅 [Frontend 指南](frontend-guide.md)。

### Kubernetes

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: frontend-example
spec:
  graphs:
    - name: frontend
      replicas: 1
      services:
        - name: Frontend
          image: nvcr.io/nvidia/dynamo/dynamo-vllm:latest
          command:
            - python
            - -m
            - dynamo.frontend
            - --http-port
            - "8000"
```

## 配置

| 参数 | 默认值 | 描述 |
|-----------|---------|-------------|
| `--http-port` | 8000 | HTTP 服务器端口 |
| `--kserve-grpc-server` | false | 启用 KServe gRPC 服务器 |
| `--router-mode` | `round-robin` | 路由策略：`round-robin`、`random`、`kv`、`direct`、`least-loaded`、`device-aware-weighted`（在分离式 prefill 模式下，`power-of-two` 与 `least-loaded` 使用同步的 prefill 回退） |

完整配置选项请参阅 [Frontend 指南](frontend-guide.md)。

## 后续步骤

| 文档 | 描述 |
|----------|-------------|
| [配置参考](configuration.md) | 所有 CLI 参数、环境变量与 HTTP 端点 |
| [Frontend 指南](frontend-guide.md) | KServe gRPC 配置与集成 |
| [NVIDIA 请求扩展（nvext）](nvext.md) | 用于路由提示与缓存控制的自定义请求字段 |
| [Router 文档](../router/README.md) | KV 感知路由配置 |
