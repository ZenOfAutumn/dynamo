# KServe gRPC 示例

本目录包含一个最小化的 Dynamo worker，它对外提供一个兼容 KServe 的 gRPC 端点（`server.py`），以及一个使用 Triton `tritonclient.grpc` API 调用该端点的 Python 客户端（`test_client.py`）。

## 先决条件

- 已安装 Dynamo Python 绑定
- 客户端依赖：
  - `numpy`
  - `tritonclient[grpc]`

可以使用以下命令将 Python 依赖安装到当前激活的环境中：

```bash
uv pip install numpy tritonclient[grpc]
```

## 运行 mock 服务器

1. 在仓库根目录下，设置 `PYTHONPATH` 以便 Python 能找到本地的 Dynamo 包：

   ```bash
   export PYTHONPATH=$(pwd)
   ```

2. 启动 worker：

   ```bash
   python lib/bindings/python/examples/kserve_grpc_service/server.py
   ```

   服务器会注册一个名为 `mock_model` 的 mock completions 模型，并监听
   `0.0.0.0:8787`。在测试端点期间请保持该进程运行。

## 使用 Triton 客户端发送请求

服务器运行后，在另一个终端中调用示例客户端：

```bash
python lib/bindings/python/examples/kserve_grpc_service/test_client.py \
  --model mock_model \
  --prompt "Hello from Dynamo!"
```


你可以根据需要覆盖 `--host`、`--port` 和 `--prompt` 选项。该脚本通过 gRPC 使用 `InferenceServerClient` 发送一次推理请求，并打印解码后的 `ModelInferResponse` 负载。你应当能看到 prompt `Hello from Dynamo!` 被服务器成功接收并打印出来。

## 替代工具

为了便于调试，你仍然可以使用 [`grpcurl`](https://github.com/fullstorydev/grpcurl) 直接调用该端点，方法是运行本目录下的 `grpcurl.sh`。
