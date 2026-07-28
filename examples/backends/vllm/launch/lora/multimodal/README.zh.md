# 多模态 LoRA 服务指南

使用 Dynamo 的聚合架构为视觉-语言模型（VLM）提供服务，并支持动态加载 LoRA 适配器。

## 前置条件

- **GPU**：具有足够显存的 NVIDIA GPU（2B 模型 8 GB+，7B 模型 24 GB+）
- **Dynamo**：已安装并启用 vLLM 支持（`pip install dynamo[vllm]`）
- **jq**（可选，用于美化 JSON 输出）：`sudo apt install jq`
- **hf CLI**（可选，用于下载适配器）：`pip install huggingface-hub`

## 快速开始

### 1. 启动服务端

```bash
cd examples/backends/vllm/launch/lora/multimodal
./lora_agg.sh
```

这会启动前端（端口 8000）与 vLLM worker（端口 8081），以 `Qwen/Qwen3-VL-2B-Instruct` 作为基础模型。

等待两个服务在日志中报告就绪（查找 `Application startup complete`）。

### 2. 验证服务端运行中

```bash
curl http://localhost:8000/v1/models | jq .
```

你应能看到列出的基础模型。

### 3. 下载 LoRA 适配器

将一个兼容的视觉 LoRA 下载到本地文件系统：

```bash
export HF_TOKEN=<your-huggingface-token>

hf download Chhagan005/Chhagan-DocVL-Qwen3 --local-dir /tmp/my-vlm-lora
```

### 4. 加载 LoRA 适配器

```bash
curl -s -X POST http://localhost:8081/v1/loras \
  -H "Content-Type: application/json" \
  -d '{
    "lora_name": "my-vlm-lora",
    "source": {"uri": "file:///tmp/my-vlm-lora"}
  }' | jq .
```

期望响应：
```json
{
  "status": "success",
  "message": "LoRA adapter 'my-vlm-lora' loaded successfully",
  "lora_name": "my-vlm-lora",
  "lora_id": 1207343256
}
```

### 5. 使用 LoRA 适配器进行推理

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "my-vlm-lora",
    "messages": [{"role": "user", "content": [
      {"type": "text", "text": "Describe this image in detail"},
      {"type": "image_url", "image_url": {"url": "https://vllm-public-assets.s3.us-west-2.amazonaws.com/vision_model_images/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg"}}
    ]}],
    "max_tokens": 300,
    "temperature": 0.0
  }' | jq .
```

### 6. 与基础模型对比

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-VL-2B-Instruct",
    "messages": [{"role": "user", "content": [
      {"type": "text", "text": "Describe this image in detail"},
      {"type": "image_url", "image_url": {"url": "https://vllm-public-assets.s3.us-west-2.amazonaws.com/vision_model_images/2560px-Gfp-wisconsin-madison-the-nature-boardwalk.jpg"}}
    ]}],
    "max_tokens": 300,
    "temperature": 0.0
  }' | jq .
```

### 7. 卸载 LoRA 适配器

```bash
curl -X DELETE http://localhost:8081/v1/loras/my-vlm-lora | jq .
```

### 8. 停止服务端

在运行 `lora_agg.sh` 的终端中按 `Ctrl+C`。trap 处理器会清理子进程。

## 配置

### 命令行选项

```bash
./lora_agg.sh --model llava-hf/llava-1.5-7b-hf            # 使用其他基础模型
./lora_agg.sh -- --enforce-eager                            # 传入额外的 vLLM 参数
./lora_agg.sh -- --mm-processor-kwargs '{"max_pixels": 1003520}'  # 限制图像分辨率
```

### 环境变量

| 变量 | 默认值 | 描述 |
|---|---|---|
| `DYN_MODEL_NAME` | `Qwen/Qwen3-VL-2B-Instruct` | 基础 VLM 模型 |
| `DYN_HTTP_PORT` | `8000` | 前端 HTTP 端口 |
| `DYN_SYSTEM_PORT` | `8081` | worker 系统/管理端口 |
| `DYN_LORA_PATH` | `/tmp/dynamo_loras_multimodal` | 本地 LoRA 适配器缓存 |
| `DYN_MAX_LORA_RANK` | `64` | 支持的最大 LoRA rank |
| `CUDA_VISIBLE_DEVICES` | `0` | GPU 设备索引 |

### 此脚本支持的基础模型

| 模型 | 备注 |
|---|---|
| `Qwen/Qwen3-VL-2B-Instruct` | 默认。适合单 GPU 测试。 |
| `Qwen/Qwen2.5-VL-7B-Instruct` | 质量更高，需 24 GB+ 显存。 |
| `llava-hf/llava-1.5-7b-hf` | LLaVA 架构，最大上下文 4096。 |

## LoRA 管理 API

所有管理端点都在系统端口（默认 8081）上提供。

### 加载 LoRA

```
POST /v1/loras
```

```json
{
  "lora_name": "my-adapter",
  "source": {
    "uri": "file:///path/to/adapter"
  }
}
```

支持的 URI 协议：
- `file://` —— 本地文件系统路径
- `s3://` —— S3 兼容存储（需要 `AWS_ENDPOINT`、`AWS_ACCESS_KEY_ID`、`AWS_SECRET_ACCESS_KEY`）

### 列出已加载的 LoRA

```
GET /v1/loras
```

### 卸载 LoRA

```
DELETE /v1/loras/{lora_name}
```

### 列出所有模型（基础 + LoRA）

```
GET /v1/models        (在前端端口，默认 8000)
```

## 运行验证脚本

提供了一个验证脚本，用于针对运行中的服务端测试 LoRA 端点：

```bash
# 在一个终端启动服务端
./lora_agg.sh

# 在另一个终端下载 LoRA 适配器
hf download Chhagan005/Chhagan-DocVL-Qwen3 --local-dir /tmp/my-vlm-lora

# 运行完整测试套件（端到端 LoRA 加载/推理/卸载）
./validate_lora_agg.sh --lora-path /tmp/my-vlm-lora

# 或仅运行错误处理与基础模型测试（无需适配器）
./validate_lora_agg.sh
```

验证脚本覆盖：
- 前端健康与基础模型发现
- LoRA 加载/卸载错误处理（缺失字段、不存在的适配器）
- 端到端 LoRA 生命周期：加载、在 `/v1/models` 中验证、推理、卸载（提供 `--lora-path` 时）
- 基础模型多模态推理

## 故障排查

### 前端无法启动
- 检查端口 8000 是否已被占用：`lsof -i :8000`
- 设置不同端口：`DYN_HTTP_PORT=8001 ./lora_agg.sh`

### 推理时 OOM
- 通过额外参数降低 `--max-model-len`：`./lora_agg.sh -- --max-model-len 4096`
- 限制图像分辨率：`./lora_agg.sh -- --mm-processor-kwargs '{"max_pixels": 1003520}'`
- 降低 GPU 显存利用率：`./lora_agg.sh -- --gpu-memory-utilization 0.80`

### LoRA 加载失败
- 验证适配器路径存在并包含 `adapter_config.json` 与 `adapter_model.safetensors`
- 确保适配器与基础模型架构兼容
- 检查 `max-lora-rank`（默认 64）>= 适配器的 rank
- 查看 worker 日志获取详细错误信息

### 加载 LoRA 后推理报错
- 验证 LoRA 已加载：`curl http://localhost:8081/v1/loras | jq .`
- 确认请求中的模型名与 `lora_name` 完全一致（区分大小写）
- 检查适配器训练时使用的基础模型一致

### 缓存问题
- 检查缓存：`ls -la /tmp/dynamo_loras_multimodal/`
- 清理缓存：`rm -rf /tmp/dynamo_loras_multimodal/*`

## 清理

```bash
# 移除 LoRA 缓存
rm -rf /tmp/dynamo_loras_multimodal

# 移除已下载的适配器
rm -rf /tmp/my-vlm-lora
```
