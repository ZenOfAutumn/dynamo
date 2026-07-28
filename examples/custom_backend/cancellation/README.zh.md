# 请求取消（Request Cancellation）演示

该演示展示了如何在 Dynamo 中实现请求取消，使用：
- 客户端（Client）：通过 `context.stop_generating()` 取消请求
- 中间服务器（Middle Server）：转发请求并透传 context（可选）
- 后端服务器（Backend Server）：通过 `context.is_stopped()` 检查取消状态

## 架构

该演示支持两种架构：

**直连模式（默认）：**
```
Client -> Backend Server
```

**带中间服务器：**
```
Client -> Middle Server -> Backend Server
```

中间服务器作为代理：
1. 接收来自客户端的请求
2. 转发到后端服务器
3. 透传原始 context 以支持取消
4. 将响应流式回传给客户端

## 用法

### 选项 1：直连（简单）

1. 启动后端服务器：
```bash
python3 server.py
```

2. 运行客户端（直连后端）：
```bash
python3 client.py
```

### 选项 2：带中间服务器（代理）

1. 启动后端服务器：
```bash
python3 server.py
```

2. 启动中间服务器：
```bash
python3 middle_server.py
```

3. 运行客户端（通过中间服务器连接）：
```bash
python3 client.py --middle
```

## 发生了什么

### 直连模式：
1. 后端服务器以 0.1 秒的延迟生成数字 0-999
2. 客户端直接从后端接收前 3 个数字（0、1、2）
3. 客户端调用 `context.stop_generating()` 取消请求
4. 后端服务器通过 `context.is_stopped()` 检测到取消并停止
5. 客户端与服务器都优雅地处理了取消

### 带中间服务器：
1. 后端服务器以 0.1 秒的延迟生成数字 0-999
2. 中间服务器转发请求并透传 context
3. 客户端通过中间服务器接收前 3 个数字（0、1、2）
4. 客户端调用 `context.stop_generating()` 取消请求
5. context 取消逐级传播：Client → Middle Server → Backend Server
6. 后端服务器通过 `context.is_stopped()` 检测到取消并停止
7. 所有组件都优雅地处理了取消

## 关键组件

- **后端服务器**：在每次 yield 前检查 `context.is_stopped()`
- **中间服务器**：转发请求并透传 context（在使用时）
- **客户端**：使用 `Context()` 对象并调用 `context.stop_generating()`
- **优雅关闭**：所有组件都处理 `asyncio.CancelledError`

## 备注

- 客户端默认使用直连模式以简化使用
- 使用 `--middle` 标志可测试代理场景
- 两种模式展示的取消行为相同
- 中间服务器示例展示了在代理场景下如何正确转发 context

关于请求取消架构的更多细节，请参阅[架构文档](../../../docs/fault-tolerance/request-cancellation.md)。
