# 兼容 S3 的存储后端 LoRA 集成指南

本指南介绍如何在 Dynamo 中通过 S3 兼容的存储后端（backend）（例如 MinIO、AWS S3、
GCS 等）来配置和使用 LoRA（Low-Rank Adaptation）适配器。

## 概述

本示例演示了如何：
1. 将 MinIO 作为本地的 S3 兼容存储启动
2. 从 Hugging Face Hub 下载 LoRA 适配器
3. 将 LoRA 适配器上传到 MinIO
4. 在 Dynamo 中加载并使用 LoRA 适配器
5. 用 LoRA 适配过的模型进行推理（inference）
6. 管理（加载 / 卸载）LoRA 适配器

## 前置条件

### 必备软件
- Docker（用于运行 MinIO）
- Python 3.10+
- AWS CLI：`pip install awscli`
- Hugging Face CLI：`pip install huggingface-hub[cli]`
- jq（可选，用于美化 JSON 输出）：`sudo apt install jq`

### Python 依赖
请确保已安装 Dynamo 与你选择的后端。环境配置说明请参见
[Dynamo quickstart guide](https://docs.nvidia.com/dynamo/getting-started/quickstart)。

## 快速开始

### 步骤 1：搭建 MinIO 并上传 LoRA

运行配置脚本启动 MinIO，并从 Hugging Face 下载 / 上传一个 LoRA 适配器：

```bash
./setup_minio.sh
```

该脚本会：
- 在 Docker 容器中启动 MinIO
- 从 Hugging Face Hub 下载一个 LoRA 适配器（默认：`codelion/Qwen3-0.6B-accuracy-recovery-lora`）
- 将该 LoRA 上传到 MinIO 的 `s3://my-loras/codelion/Qwen3-0.6B-accuracy-recovery-lora`

#### 脚本选项

该配置脚本支持多种模式：

```bash
# 完整流程（默认）—— 启动 MinIO，下载并上传 LoRA
./setup_minio.sh

# 仅启动 MinIO（不下载 / 上传）
./setup_minio.sh --start

# 停止 MinIO
./setup_minio.sh --stop

# 显示帮助
./setup_minio.sh --help
```

#### 自定义要下载的 LoRA

你可以指定不同的 LoRA 仓库与名称：

```bash
HF_LORA_REPO="username/lora-repo" \
LORA_NAME="my-lora" \
  ./setup_minio.sh
```

### 步骤 2：以 LoRA 支持启动 Dynamo

启动启用了 LoRA 支持的 Dynamo 前端（frontend）与 worker：

```bash
./agg_lora.sh
```

这会：
- 为 MinIO 配置 AWS 凭据
- 在端口 8000 上启动 Dynamo 前端
- 在端口 8081 上启动启用了 LoRA 支持的 Dynamo worker

等待服务启动完成（在日志中查看 "Application startup complete"）。

## 使用 LoRA

### 1. 查看可用模型

列出所有可用模型（一开始仅有基础模型）：

```bash
curl http://localhost:8000/v1/models | jq .
```

### 2. 加载 LoRA 适配器

从 S3 兼容的存储后端（例如 MinIO）加载一个 LoRA：

```bash
curl -X POST http://localhost:8081/v1/loras \
  -H "Content-Type: application/json" \
  -d '{
    "lora_name": "codelion/Qwen3-0.6B-accuracy-recovery-lora",
    "source": {
      "uri": "s3://my-loras/codelion/Qwen3-0.6B-accuracy-recovery-lora"
    }
  }' | jq .
```

预期响应：
```json
{
  "status": "success",
  "message": "LoRA adapter 'codelion/Qwen3-0.6B-accuracy-recovery-lora' loaded successfully",
  "lora_name": "codelion/Qwen3-0.6B-accuracy-recovery-lora",
  "lora_id": 1207343256
}
```

### 3. 列出已加载的 LoRA

查看当前已加载了哪些 LoRA：

```bash
curl http://localhost:8081/v1/loras | jq .
```

### 4. 在模型列表中确认 LoRA

加载完成后，该 LoRA 会出现在模型列表中：

```bash
curl http://localhost:8000/v1/models | jq .
```

你应该能同时看到基础模型与 LoRA 适配器。

### 5. 使用 LoRA 进行推理

#### 使用 LoRA 适配过的模型：

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "codelion/Qwen3-0.6B-accuracy-recovery-lora",
    "messages": [{
      "role": "user",
      "content": "What is good low risk investment strategy?"
    }],
    "max_tokens": 300,
    "temperature": 0.1
  }' | jq .
```

#### 作为对比，使用基础模型：

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [{
      "role": "user",
      "content": "What is good low risk investment strategy?"
    }],
    "max_tokens": 300
  }' | jq .
```

### 6. 卸载 LoRA

不再使用某个 LoRA 时，卸载它以释放资源：

```bash
curl -X DELETE http://localhost:8081/v1/loras/codelion/Qwen3-0.6B-accuracy-recovery-lora | jq .
```

预期响应：
```json
{
  "status": "success",
  "message": "LoRA unloaded successfully"
}
```

卸载之后，该 LoRA 会从 `/v1/loras` 与 `/v1/models` 接口中同时被移除。

## 配置

### 环境变量

可配置的环境变量如下：

```bash
# S3 兼容存储后端配置
export AWS_ENDPOINT=http://localhost:9000
export AWS_ACCESS_KEY_ID=minioadmin
export AWS_SECRET_ACCESS_KEY=minioadmin
export AWS_REGION=us-east-1

# Dynamo LoRA 配置
export DYN_LORA_ENABLED=true
export DYN_LORA_PATH=/tmp/dynamo_loras_minio
```

### MinIO 控制台

通过 `http://localhost:9001` 访问 MinIO Web 控制台：
- 用户名：`minioadmin`
- 密码：`minioadmin`

## 故障排查

### MinIO 无法启动
- 检查 9000 与 9001 端口是否被占用
- 确认 Docker 处于运行状态
- 查看 Docker 日志：`docker logs dynamo-minio`
- 尝试停止已有的 MinIO 容器：`./setup_minio.sh --stop`
- 重新启动 MinIO：`./setup_minio.sh --start`

### LoRA 加载失败
- 确认 LoRA 已上传到 MinIO：`aws --endpoint-url=http://localhost:9000 s3 ls s3://my-loras/`
- 检查 AWS 凭据是否正确设置
- 确认 LoRA 文件与基础模型兼容
- 查看 worker 日志中的详细错误信息

### 推理失败
- 确认模型名完全一致（区分大小写）
- 检查 LoRA 是否已加载：`curl http://localhost:8081/v1/loras`
- 确认基础模型支持该 LoRA 的 rank
- 确认 worker 配置中的 max_lora_rank >= 该 LoRA 的 rank

### 缓存问题
- 检查缓存目录：`ls -la /tmp/dynamo_loras_minio/`
- 必要时清空缓存：`rm -rf /tmp/dynamo_loras_minio/*`
- 确认缓存目录可写

## 进阶用法

### 同时加载多个 LoRA

可以同时加载多个 LoRA 适配器：

```bash
# 加载第一个 LoRA
curl -X POST http://localhost:8081/v1/loras \
  -H "Content-Type: application/json" \
  -d '{"lora_name": "lora1", "source": {"uri": "s3://my-loras/lora1"}}'

# 加载第二个 LoRA
curl -X POST http://localhost:8081/v1/loras \
  -H "Content-Type: application/json" \
  -d '{"lora_name": "lora2", "source": {"uri": "s3://my-loras/lora2"}}'
```

### 使用其他基础模型

要使用不同的基础模型，请修改 `MODEL` 环境变量：

```bash
MODEL=meta-llama/Llama-2-7b-hf ./agg_lora.sh
```

请确保你的 LoRA 与所选基础模型兼容。

## 清理

### 停止服务

在运行 `agg_lora.sh` 的终端中按 `Ctrl+C` 停止 Dynamo 服务。

### 停止 MinIO

```bash
# 使用配置脚本（推荐）
./setup_minio.sh --stop

# 或手动使用 Docker
docker stop dynamo-minio
docker rm dynamo-minio
```

### 清理数据

```bash
# 移除 MinIO 数据
rm -rf ~/dynamo_minio_data

# 移除 LoRA 缓存
rm -rf /tmp/dynamo_loras_minio
```

## API 参考

### 加载 LoRA
- **Endpoint**：`POST /v1/loras`
- **Body**：`{"lora_name": "string", "source": {"uri": "string"}}`
- **Response**：`{"status": "success", "lora_id": int}`

### 列出 LoRA
- **Endpoint**：`GET /v1/loras`
- **Response**：已加载 LoRA 的数组

### 卸载 LoRA
- **Endpoint**：`DELETE /v1/loras/{lora_name}`
- **Response**：`{"status": "success", "message": "string"}`

### 列出模型
- **Endpoint**：`GET /v1/models`
- **Response**：兼容 OpenAI 的模型列表

### Chat Completions
- **Endpoint**：`POST /v1/chat/completions`
- **Body**：兼容 OpenAI 的 chat completion 请求
- **Response**：兼容 OpenAI 的 chat completion 响应
