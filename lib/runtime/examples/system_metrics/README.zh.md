# 组件端点（Component Endpoint）通用指标（Metrics）

本示例演示组件端点默认会自动获得的指标（metrics）能力。

## 概览

使用 DistributedRuntime 时，组件端点会被自动度量。DistributedRuntime 使用 `MetricsRegistry` trait，自动为所有组件端点提供度量能力。它会自动跟踪：

- **请求计数**：处理的请求总数
- **请求耗时**：处理每个请求所花的时间
- **请求/响应字节数**：接收与发送的总字节数
- **错误计数**：遇到的错误总数

此外，本示例还演示了如何添加带数据字节跟踪的自定义指标。

## 工作原理

**自动指标**：所有组件端点无需任何代码改动即可自动获得度量指标。

**自定义指标**：如果你希望在自动指标之外另外添加自定义指标，可使用 `add_metrics` 方法：

```rust
use dynamo_runtime::pipeline::network::Ingress;

// Automatic measurements - no code changes needed!
let ingress = Ingress::for_engine(my_handler)?;

// Optional: Add custom metrics IN ADDITION to automatic ones
ingress.add_metrics(&endpoint)?;
```

端点会为所有指标自动加上恰当的标签（dynamo_namespace、dynamo_component、dynamo_endpoint）。这些标签都以 "dynamo_" 为前缀，避免与 Kubernetes 及其他监控系统的标签冲突。

## 可用方法

`Ingress` 结构体提供与指标相关的方法：

- **自动**：所有组件端点自动获得度量指标
- `Ingress::add_metrics(&endpoint)` - 在自动指标基础上额外添加自定义指标（可选）

## 生成的指标

### 自动指标（无需代码改动）
以下 Prometheus 指标会自动为所有组件端点创建：

### Counters
- `dynamo_component_requests_total` - 处理的请求总数
- `dynamo_component_request_bytes_total` - 请求中接收的字节总数
- `dynamo_component_response_bytes_total` - 响应中发送的字节总数
- `dynamo_component_errors_total` - 遇到的错误总数（带 error_type 标签）

### 错误类型
`dynamo_component_errors_total` 指标包含以下错误类型：
- `deserialization` - 解析请求消息时出错
- `invalid_message` - 非预期的消息格式
- `response_stream` - 创建响应流出错
- `generate` - 请求处理过程中出错
- `publish_response` - 发布响应数据出错
- `publish_final` - 发布最终响应出错

### Histograms
- `dynamo_component_request_duration_seconds` - 请求处理时间

### Gauges
- `dynamo_component_inflight_requests` - 当前正在处理的请求数

### 自定义指标（可选）
- `dynamo_component_bytes_processed_total` - 系统处理器（system handler）处理的数据字节总数（示例）

### 标签
所有指标都会自动从端点带上以下标签：
- `dynamo_namespace` - namespace 名称
- `dynamo_component` - component 名称
- `dynamo_endpoint` - endpoint 名称

这些标签都带有 "dynamo_" 前缀，避免与 Kubernetes 及其他监控系统的标签冲突。

## 指标输出示例

系统运行时，你可以通过 http://<ip>:<port>/metrics 看到类似下面的指标：

```prometheus
# HELP dynamo_component_inflight_requests Number of requests currently being processed by component endpoint
# TYPE dynamo_component_inflight_requests gauge
dynamo_component_inflight_requests{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace"} 0

# HELP dynamo_component_bytes_processed_total Example of a custom metric. Total number of data bytes processed by system handler
# TYPE dynamo_component_bytes_processed_total counter
dynamo_component_bytes_processed_total{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace"} 42

# HELP dynamo_component_request_bytes_total Total number of bytes received in requests by component endpoint
# TYPE dynamo_component_request_bytes_total counter
dynamo_component_request_bytes_total{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace"} 1098

# HELP dynamo_component_request_duration_seconds Time spent processing requests by component endpoint
# TYPE dynamo_component_request_duration_seconds histogram
dynamo_component_request_duration_seconds_bucket{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace",le="0.005"} 3
dynamo_component_request_duration_seconds_bucket{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace",le="0.01"} 3
dynamo_component_request_duration_seconds_bucket{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace",le="0.025"} 3
dynamo_component_request_duration_seconds_bucket{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace",le="0.05"} 3
dynamo_component_request_duration_seconds_bucket{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace",le="0.1"} 3
dynamo_component_request_duration_seconds_bucket{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace",le="0.25"} 3
dynamo_component_request_duration_seconds_bucket{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace",le="0.5"} 3
dynamo_component_request_duration_seconds_bucket{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace",le="1"} 3
dynamo_component_request_duration_seconds_bucket{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace",le="2.5"} 3
dynamo_component_request_duration_seconds_bucket{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace",le="5"} 3
dynamo_component_request_duration_seconds_bucket{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace",le="10"} 3
dynamo_component_request_duration_seconds_bucket{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace",le="+Inf"} 3
dynamo_component_request_duration_seconds_sum{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace"} 0.00048793700000000003
dynamo_component_request_duration_seconds_count{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace"} 3

# HELP dynamo_component_requests_total Total number of requests processed by component endpoint
# TYPE dynamo_component_requests_total counter
dynamo_component_requests_total{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace"} 3

# HELP dynamo_component_response_bytes_total Total number of bytes sent in responses by component endpoint
# TYPE dynamo_component_response_bytes_total counter
dynamo_component_response_bytes_total{dynamo_component="example_component",dynamo_endpoint="example_endpoint9881",dynamo_namespace="example_namespace"} 1917

# HELP uptime_seconds Total uptime of the DistributedRuntime in seconds
# TYPE uptime_seconds gauge
uptime_seconds{dynamo_namespace="system_status_server"} 1.8226759879999999
```

## 示例

### 自动度量并可选添加自定义指标的组件端点

```rust
struct RequestHandler {
    metrics: Option<Arc<CustomMetrics>>,
}

#[async_trait]
impl AsyncEngine<SingleIn<String>, ManyOut<Annotated<String>>, Error> for RequestHandler {
    async fn generate(&self, input: SingleIn<String>) -> Result<ManyOut<Annotated<String>>> {
        let (data, ctx) = input.into_parts();

        // Optional: Track custom metrics
        if let Some(metrics) = &self.metrics {
            metrics.data_bytes_processed.inc_by(data.len() as u64);
        }

        // Your business logic here...
        // No need to add any automatic measurement code!

        Ok(ResponseStream::new(Box::pin(stream), ctx.context()))
    }
}

// Create handler (with or without custom metrics)
let handler = if enable_custom_metrics {
    let custom_metrics = CustomMetrics::from_endpoint(&endpoint)?;
    RequestHandler::with_metrics(custom_metrics)
} else {
    RequestHandler::new()
};

// Automatic measurements - no additional code needed!
let ingress = Ingress::for_engine(handler)?;

// Optional: Add custom metrics IN ADDITION to automatic ones
if enable_custom_metrics {
    ingress.add_metrics(&endpoint)?;
}

// Endpoint code to add ingress to the handler below...
```

## 收益

1. **几乎/无需代码改动**：现有处理器自动获得度量指标，并可方便地为特定应用添加自定义指标。
2. **简单的 API**：只需将 Prometheus 构造器替换为端点的工厂方法之一。
3. **自动度量**：组件端点开箱即用地具备请求计数、耗时与错误跟踪。
4. **自动标签**：端点提供恰当的 namespace/component/endpoint 标签

## 运行示例

**重要**：必须设置 `DYN_SYSTEM_PORT` 环境变量来指定 system status server 监听的端口。

```bash
# Run the system metrics example
DYN_SYSTEM_PORT=8081 cargo run --bin system_server
```
服务端将在指定端口（本示例为 8081）启动一个 system status server，并通过 `/metrics` 暴露 Prometheus 指标端点。


如需运行真实 LLM frontend + server（聚合示例），请同时启动两者。前端默认监听 8000 端口。
```
python -m dynamo.frontend &

DYN_SYSTEM_PORT=8081 python -m dynamo.vllm --model Qwen/Qwen3-0.6B --enforce-eager --no-enable-prefix-caching &
```
然后向前端发起 curl 请求（参见 [main README](../../../../README.md)）

## 查询指标

启动后即可查询指标：

```bash
# Get all component endpoint metrics for components
curl http://localhost:8081/metrics | grep -E "dynamo_component"

# Get all frontend metrics
curl http://localhost:8000/metrics | grep -E "dynamo_frontend"
```
