# 适用于 Dynamo 的 Triton Server 后端

> **⚠️ 进行中 / 概念验证（PoC）**
>
> 该示例演示了如何把 NVIDIA Triton Inference Server 集成为 Dynamo 的后端。
> 当前仅为概念验证，若要用于生产，可能还需要额外工作。

## 概览

该示例展示了如何通过 Dynamo 的分布式 runtime 运行 Triton Server 模型，并通过 KServe gRPC 协议对外暴露。该集成可让 Triton 模型享受到 Dynamo 在服务发现、路由与基础设施方面的能力。

**架构：**

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────────────────┐
│  Triton Client  │────▶│  Dynamo Frontend│────▶│       Dynamo Worker         │
│  (KServe gRPC)  │     │  (port 8787)    │     │  ┌───────────────────────┐  │
└─────────────────┘     └─────────────────┘     │  │    Triton Server      │  │
                              │                 │  │  (Python bindings)    │  │
                              ▼                 │  └───────────────────────┘  │
                    ┌─────────────────┐         └─────────────────────────────┘
                    │    KV Store     │
                    └─────────────────┘
```

## 前置条件

- 支持 CUDA 的 NVIDIA GPU
- 本地开发：Python 3.10+ 并已安装 Dynamo
- 容器部署：Docker 与 NVIDIA Container Toolkit

## 快速上手

### 选项 1：容器部署

#### 步骤 1：构建容器镜像

在 Dynamo 仓库根目录下：

```bash
# Build the base Dynamo image
python container/render.py --framework=dynamo --target=runtime --output-short-filename
docker build -f container/rendered.Dockerfile -t dynamo-base:latest .

# Build the Triton worker image
cd examples/backends/tritonserver
docker build -t dynamo-triton:latest .
```

#### 步骤 2：运行容器

```bash
docker run --rm -it --gpus all --network host \
  dynamo-triton:latest \
  ./examples/backends/tritonserver/launch/identity.sh
```

#### 步骤 3：测试部署

在另一个终端：

```bash
# Install client dependencies
pip install tritonclient[grpc]

# Test with the client
cd examples/backends/tritonserver
python src/client.py --port 8000
```

### 选项 2：本地开发

需要在本地安装 Dynamo。

```bash
# From the dynamo repo root
cd examples/backends/tritonserver

# Build Triton Server (first time only, ~30 minutes)
make all

# Install Python dependencies
pip install wheelhouse/tritonserver-*.whl
pip install tritonclient[grpc]

# Launch the server
./launch/identity.sh

# In another terminal, test with the client
python src/client.py
```

## 目录结构

```
tritonserver/
├── launch/
│   └── identity.sh      # Launch script (frontend + worker)
├── src/
│   ├── tritonworker.py  # Main Dynamo worker implementation
│   └── client.py        # Test client (KServe gRPC)
├── model_repo/
│   └── identity/        # Sample identity model
│       ├── config.pbtxt
│       └── 1/
├── backends/            # Triton backends (built by `make all`)
├── lib/                 # Triton libraries (built by `make all`)
├── wheelhouse/          # Python wheels (built by `make all`)
├── Dockerfile           # Triton worker container
└── Makefile             # Build Triton from source
```

## 配置

### 启动脚本选项

```bash
./launch/identity.sh --help

Options:
  --model-name <name>         Model name to load (default: identity)
  --model-repository <path>   Path to model repository
  --backend-directory <path>  Path to Triton backends
  --log-verbose <level>       Triton log verbosity 0-6 (default: 1)
  --discovery-backend <backend> Discovery backend: kubernetes, etcd, file, mem (default: file)
```

### 环境变量

| 变量 | 说明 | 默认值 |
|----------|-------------|---------|
| `BACKEND_DIR` | Triton 后端的路径。容器镜像将其设为 `/opt/tritonserver/backends`；本地源码构建则使用 `backends/`。 | `backends/` |
| `DYN_DISCOVERY_BACKEND` | 发现后端：`kubernetes`、`etcd`、`file` 或 `mem` | `file` |
| `DYN_LOG` | 日志级别（debug、info、warn、error） | `info` |
| `DYN_HTTP_PORT` | 前端 HTTP 端口 | `8000` |
| `ETCD_ENDPOINTS` | etcd 连接 URL（仅在 `--discovery-backend etcd` 时） | `http://localhost:2379` |
| `NATS_SERVER` | NATS 连接 URL（仅分布式模式下使用） | `nats://localhost:4222` |

## 添加你自己的模型

1. 在 `model_repo/` 下创建模型目录：

   ```text
   model_repo/
   └── my_model/
       ├── config.pbtxt
       └── 1/
           └── model.plan  # or other model file
   ```

2. 定义模型配置（`config.pbtxt`）：

   ```protobuf
   name: "my_model"
   backend: "tensorrt"  # or onnxruntime, python, etc.
   max_batch_size: 8

   input [
     {
       name: "input"
       data_type: TYPE_FP32
       dims: [3, 224, 224]
     }
   ]
   output [
     {
       name: "output"
       data_type: TYPE_FP32
       dims: [1000]
     }
   ]
   ```

3. 用你的模型启动：

   ```bash
   ./launch/identity.sh --model-name my_model
   ```

## 已知限制

- **单一模型**：当前一次仅加载一个模型
- **仅 identity 后端**：Makefile 默认只构建 identity 后端；其他后端需要修改构建配置

## 从源码构建 Triton

本地开发所必需。Makefile 会构建 Triton Server 与 identity 后端。

```bash
cd examples/backends/tritonserver

# Build Triton Server (~30 minutes, clones and builds from source)
make all

# Check build status
make status

# This produces:
#   lib/libtritonserver.so     - Core library
#   bin/tritonserver           - Server binary
#   backends/identity/         - Identity backend
#   wheelhouse/*.whl           - Python bindings

# Clean up build artifacts
make clean      # Remove installed artifacts
make distclean  # Remove everything including build cache
```

要添加其他后端（TensorRT、ONNX、Python 等），可编辑 Makefile 中 `build.py` 的调用，添加更多 `--backend=<name>` 标志。

## 故障排查

### "Model not found" 错误

- 确认模型存在于 `model_repo/<model_name>/` 下
- 确认 `config.pbtxt` 有效
- 确认 `backends/` 中存在对应后端

### Worker 启动失败

- 确认 `LD_LIBRARY_PATH` 包含 Triton 库
- 确认 GPU 可用：`nvidia-smi`
- 提高日志详细度：`--log-verbose 6`

## 相关文档

- [Dynamo Backend Guide](../../../docs/development/backend-guide.md)
- [Triton Inference Server](https://github.com/triton-inference-server/server)
- [KServe Protocol](https://kserve.github.io/website/latest/modelserving/data_plane/v2_protocol/)
