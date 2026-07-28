---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Examples
---

如需快速上手，请参阅 [SGLang README](README.md)。本文档汇总了使用 Dynamo 运行 SGLang 的所有部署模式，包括 LLM、多模态、扩散（diffusion）模型以及 Kubernetes 部署。

## 基础设施搭建

对于本地/裸机开发，可使用 Docker Compose 启动 etcd（可选启动 NATS）：

```bash
docker compose -f deploy/docker-compose.yml up -d
```

<Note>
- **etcd** 是可选的，但是默认的本地服务发现（discovery）后端。你也可以使用 `--discovery-backend file` 切换为基于文件系统的服务发现。
- **NATS** 仅在使用带事件的 KV 路由（`--kv-events-config`）时才需要。如果想要基于预测的路由而不依赖 NATS，可在前端（frontend）使用 `--no-router-kv-events`。
- **在 Kubernetes 上**，使用 Dynamo Operator（`DYN_DISCOVERY_BACKEND=kubernetes`）时两者都不需要。
</Note>

<Tip>
每个启动脚本都会在同一终端里同时运行前端和 worker。你也可以为了便于测试，把每条命令分别放到不同的终端里运行。对于使用 Dynamo 的 AI agent 而言，你可以把启动脚本放到后台运行，然后用 `curl` 命令去测试这次部署。
</Tip>

## LLM 服务

### 聚合式服务（Aggregated Serving）

最简单的部署模式：单个 worker 同时处理预填充（prefill）与解码（decode）。

```bash
cd $DYNAMO_HOME/examples/backends/sglang
./launch/agg.sh
```

### 带 KV 路由的聚合式服务

两个 worker 位于一个 [KV 感知路由（KV-aware router）](../../components/router/README.md) 之后，最大化缓存复用：

```bash
cd $DYNAMO_HOME/examples/backends/sglang
./launch/agg_router.sh
```

该命令会以 `--router-mode kv` 启动前端，并启动两个开启基于 ZMQ 的 KV 事件发布的 worker。

### 解耦式服务（Disaggregated Serving）

将预填充与解码拆分为独立 worker，并通过 NIXL 进行 KV 缓存（KV cache）传输。需要 2 个 GPU。

```bash
cd $DYNAMO_HOME/examples/backends/sglang
./launch/disagg.sh
```

关于 SGLang 解耦在 Dynamo 上的工作机制，包括 bootstrap 机制和 RDMA 传输流程，详见 [SGLang Disaggregation](sglang-disaggregation.md)。

### 解耦服务 + KV 感知 Prefill 路由

在两个池上都启用 KV 感知路由，扩展为 2 prefill + 2 decode worker。需要 4 个 GPU。

```bash
cd $DYNAMO_HOME/examples/backends/sglang
./launch/disagg_router.sh
```

前端使用 `--router-mode kv`，并自动检测 prefill worker 以激活内部的 prefill 路由器（router）。每个 worker 在唯一端口上通过 ZMQ 发布 KV 事件。

## 多模态服务

### 聚合式多模态

使用 SGLang 内置的多模态支持来提供多模态模型的服务：

```bash
cd $DYNAMO_HOME/examples/backends/sglang
./launch/agg_vision.sh
```

<Accordion title="Verify the deployment">
```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-VL-8B-Instruct",
    "messages": [
      {
        "role": "user",
        "content": [
          {"type": "text", "text": "Explain why Roger Federer is considered one of the greatest tennis players of all time"},
          {"type": "image_url", "image_url": {"url": "https://media.newyorker.com/photos/63249cff39ac97c4c23ff5d0/master/w_2560%2Cc_limit/Marzorati%2520-%2520Federer%2520Retirement%25202.jpg"}}
        ]
      }
    ],
    "max_tokens": 50,
    "stream": false
  }' | jq
```
</Accordion>

### 解耦组件下的多模态

更进阶的多模态部署模式（如分别部署 encoder、prefill 与 decode worker，对应 E/PD 与 E/P/D 模式），请参阅专门的 [SGLang Multimodal](../../features/multimodal/multimodal-sglang.md) 文档。

| 模式    | 脚本                              | 描述                                              |
| ------- | --------------------------------- | ------------------------------------------------- |
| E/PD    | `./launch/multimodal_epd.sh`      | 独立的 vision encoder + 合并的 PD worker          |
| E/P/D   | `./launch/multimodal_disagg.sh`   | 独立的 encoder、prefill 与 decode worker          |

## 扩散模型

### 扩散语言模型

运行 [LLaDA2.0](https://github.com/inclusionAI/LLaDA2.0) 等扩散语言模型：

```bash
cd $DYNAMO_HOME/examples/backends/sglang
./launch/diffusion_llada.sh
```

### 图像扩散

使用 [FLUX](https://huggingface.co/black-forest-labs/FLUX.1-dev) 等扩散模型从文本提示生成图像：

```bash
cd $DYNAMO_HOME/examples/backends/sglang
./launch/image_diffusion.sh
```

可选参数：`--model-path`、`--fs-url`（本地或 S3）、`--http-url`。

### 视频生成

使用 [Wan2.1](https://huggingface.co/Wan-AI) 系列模型从文本提示生成视频：

```bash
cd $DYNAMO_HOME/examples/backends/sglang
./launch/text-to-video-diffusion.sh
```

可选参数：`--wan-size 1b|14b`、`--num-frames`、`--height`、`--width`、`--num-inference-steps`。

各类扩散 worker（LLM、图像、视频）的完整说明，请参阅 [Diffusion](sglang-diffusion.md)。

### Kubernetes 部署

完整的 K8s 部署示例见：

- [SGLang K8s 部署指南](https://github.com/ai-dynamo/dynamo/tree/main/examples/backends/sglang/deploy)
- [SGLang 聚合路由 K8s 示例](https://github.com/ai-dynamo/dynamo/blob/main/examples/backends/sglang/deploy/agg_router.yaml)
- [Kubernetes 部署指南](../../kubernetes/README.md)

## 故障排查

### CuDNN 版本检查失败

```
RuntimeError: cuDNN frontend 1.8.1 requires cuDNN lib >= 9.5.0
```

启动前设置 `SGLANG_DISABLE_CUDNN_CHECK=1`。当 PyTorch 携带的 CuDNN 版本低于 SGLang 的 Conv3d 模型所需版本时常会出现此问题，影响 vision 与扩散模型。

### 模型注册失败并报 `config.json` 错误

```
unable to extract config.json from directory ...
```

此问题出现在使用 `model_index.json` 而非 `config.json` 的 diffusers 模型上（如 FLUX.1-dev、Wan2.1 等）。请确保使用了正确的 worker 标志（`--image-diffusion-worker` 或 `--video-generation-worker`），而不是标准的 LLM worker 模式。这些标志使用的是不需要 `config.json` 的注册路径。

### 启动时 GPU 显存不足（OOM）

如果上一次运行残留了孤儿 GPU 进程，下一次启动可能会 OOM。请检查僵尸进程：

```bash
nvidia-smi  # look for lingering sgl_diffusion::scheduler or python processes
kill -9 <PID>
```

### 解耦 worker 之间无法连接

请确保 prefill 与 decode worker 之间能通过 TCP 互相访问。bootstrap 机制使用 `--disaggregation-bootstrap-port`（默认：12345）。多节点部署时，请确保该端口跨主机可达，并设置 `--host 0.0.0.0`。

## 参见

- **[SGLang README](README.md)**：快速上手与功能概览
- **[Reference Guide](sglang-reference-guide.md)**：架构、配置与运行细节
- **[SGLang Multimodal](../../features/multimodal/multimodal-sglang.md)**：vision 模型部署模式
- **[SGLang HiCache](../../integrations/sglang-hicache.md)**：分层缓存集成
- **[Benchmarking](../../benchmarks/benchmarking.md)**：性能基准工具
- **[Tuning Disaggregated Performance](../../performance/tuning.md)**：P/D 调优指南
