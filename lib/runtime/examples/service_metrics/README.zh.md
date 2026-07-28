# 服务指标（Service Metrics）

本示例在 hello_world 示例的基础上，针对客户端刚刚发出请求的服务名调用了
`scrape_service` 方法，从而获取该请求所对应服务的元信息。

```bash
DYN_LOG=debug cargo run --bin service_server
```

客户端可以观察到该服务每个实例的一些基础统计信息。

```bash
DYN_LOG=info cargo run --bin service_client
```

## 示例输出
```
Annotated { data: Some("h"), id: None, event: None, comment: None }
Annotated { data: Some("e"), id: None, event: None, comment: None }
Annotated { data: Some("l"), id: None, event: None, comment: None }
Annotated { data: Some("l"), id: None, event: None, comment: None }
Annotated { data: Some("o"), id: None, event: None, comment: None }
Annotated { data: Some(" "), id: None, event: None, comment: None }
Annotated { data: Some("w"), id: None, event: None, comment: None }
Annotated { data: Some("o"), id: None, event: None, comment: None }
Annotated { data: Some("r"), id: None, event: None, comment: None }
Annotated { data: Some("l"), id: None, event: None, comment: None }
Annotated { data: Some("d"), id: None, event: None, comment: None }
ServiceSet { services: [ServiceInfo { name: "dynamo_init_backend_720278f8", id: "eOHMc4ndRw8s5flv4WOZx7", version: "0.0.1", started: "2025-02-26T18:54:04.917294605Z", endpoints: [EndpointInfo { name: "dynamo_init_backend_720278f8-generate-694d951a80e06abf", subject: "dynamo_init_backend_720278f8.generate-694d951a80e06abf", data: Some(Metrics(Object {"average_processing_time": Number(53662), "last_error": String(""), "num_errors": Number(0), "num_requests": Number(2), "processing_time": Number(107325), "queue_group": String("q")})) }] }] }
```

如果同时启动两份 server，会看到两条对应的条目被发出。
