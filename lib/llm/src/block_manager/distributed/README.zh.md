# 主动消息处理系统（Active Message Handling System）

本模块提供基于异步 future 的主动消息处理系统，具备完善的错误处理、响应通知与基于通道的通信。

## 关键特性

- **基于异步 Future**：handler 是 `Arc<dyn Future>`，可以捕获资源并异步运行
- **并发控制**：可配置的并发上限，基于信号量进行节流
- **响应通知**：可选的响应通知，格式为 `:ok` 或 `:err(<message>)`
- **基于通道的通信**：所有通信通过通道进行，保持职责清晰分离
- **错误处理**：完善的错误处理，附带日志与监控
- **资源捕获**：handler 可安全地捕获并共享资源

## 架构

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   通信层         │───▶│ ActiveMessage    │───▶│   Handler       │
│                 │    │    Manager       │    │   Futures       │
└─────────────────┘    └──────────────────┘    └─────────────────┘
         ▲                        │                       │
         │                        ▼                       ▼
         │              ┌──────────────────┐    ┌─────────────────┐
         └──────────────│   响应通知        │◀───│   异步任务池     │
                        │                  │    │                 │
                        └──────────────────┘    └─────────────────┘
```

## 使用方法

### 1. 初始化系统

```rust
use dynamo_llm::block_manager::distributed::worker::*;

// 创建 worker 并初始化主动消息管理器
let mut worker = KvBlockManagerWorker::new(config)?;
worker.init_active_message_manager(4)?; // 4 个并发 handler

// 创建 handler
let handlers = create_example_handlers();
worker.register_handlers(handlers)?;

// 获取通信通道
let message_sender = worker.get_message_sender()?;
let response_receiver = worker.get_response_receiver()?;
```

### 2. 创建自定义 Handler

```rust
#[derive(Clone)]
struct MyHandler {
    name: String,
    shared_resource: Arc<Mutex<SomeResource>>,
}

impl MyHandler {
    async fn handle_message(&self, data: Vec<u8>) -> Result<()> {
        // 异步处理消息
        let processed_data = self.process_data(data).await?;

        // 更新共享资源
        let mut resource = self.shared_resource.lock().await;
        resource.update(processed_data)?;

        Ok(())
    }
}

// 注册 handler
let handler = MyHandler::new("my_handler".to_string(), shared_resource);
let mut handlers = HashMap::new();
handlers.insert("my_message_type".to_string(), create_handler!(handler));
```

### 3. 发送消息

```rust
// 带响应通知的消息
let message = IncomingActiveMessage {
    message_type: "my_message_type".to_string(),
    message_data: b"Hello, World!".to_vec(),
    response_notification: Some("request_123".to_string()),
};

message_sender.send(message)?;
```

### 4. 处理响应

```rust
// 启动一个任务处理响应
tokio::spawn(async move {
    while let Some(response) = response_receiver.recv().await {
        match response.is_success {
            true => {
                info!("✅ 成功: {}", response.notification);
                // response.notification = "request_123:ok"
            }
            false => {
                warn!("❌ 错误: {}", response.notification);
                // response.notification = "request_123:err(Error message)"
            }
        }
    }
});
```

## 消息流程

1. **接收消息**：通信层收到字节流和可选的响应通知前缀
2. **通道发送**：消息通过通道发往主动消息管理器
3. **Handler 查找**：管理器为该消息类型寻找合适的 handler
4. **创建 Future**：handler 工厂创建一个捕获资源的异步 future
5. **异步执行**：future 在受并发控制的任务中被启动
6. **生成响应**：完成时生成响应通知（若请求时指定）
7. **发送响应**：响应通过响应通道返回

## 响应通知格式

- **成功**：`{prefix}:ok`
- **错误**：`{prefix}:err({error_message})`

示例：
- 带通知前缀的请求：`"user_request_456"`
- 成功响应：`"user_request_456:ok"`
- 错误响应：`"user_request_456:err(Invalid data format)"`

## 错误处理

系统提供多层错误处理：

1. **Handler 错误**：被捕获并转换为错误响应通知
2. **未知消息类型**：未注册的消息类型会生成错误响应
3. **通道错误**：被记录并优雅处理
4. **并发上限**：通过信号量管理，防止资源耗尽

## 测试

运行完整测试套件：

```bash
cargo test test_active_message_flow
cargo test test_resource_capturing_handler
cargo test test_communication_integration
cargo test test_concurrency_performance
```

## 性能特征

- **并发**：可配置的并发 handler 上限
- **内存**：基于通道的高效通信，最小化拷贝
- **延迟**：低延迟的消息分发，异步处理
- **吞吐量**：高吞吐量，配合合适的反压（backpressure）处理

## 最佳实践

1. **Handler 设计**：保持 handler 轻量、对异步友好
2. **资源管理**：共享资源使用 `Arc<Mutex<T>>`
3. **错误处理**：始终在 handler 中优雅处理错误
4. **并发**：根据负载设置合适的并发上限
5. **监控**：利用响应通知进行监控与调试
