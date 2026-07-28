---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Metrics Developer Guide
---

本指南介绍如何使用 Dynamo 指标 API 在 Dynamo 组件中创建和使用自定义指标（metrics）。

## 指标暴露

通过 Dynamo 指标 API 创建的所有指标，在设置以下环境变量后会自动以 Prometheus Exposition Format 文本形式暴露在 `/metrics` HTTP 端点：

- `DYN_SYSTEM_PORT=<port>` - 指标端点端口（设置为正值以启用，默认 `-1` 禁用）

示例：
```bash
DYN_SYSTEM_PORT=8081 python -m dynamo.vllm --model <model>
```

Prometheus Exposition Format 文本指标可在 `http://localhost:8081/metrics` 获取。

## 指标名常量

[prometheus_names.rs](https://github.com/ai-dynamo/dynamo/tree/main/lib/runtime/src/metrics/prometheus_names.rs) 模块提供了集中式的指标名常量与名称清洗函数，确保所有 Dynamo 组件之间的一致性。

---

## Rust 中的指标 API

指标 API 通过 runtime、namespace、component 和 endpoint 对象上的 `.metrics()` 方法访问。层级结构详情参见 [Runtime Hierarchy](metrics.md#runtime-hierarchy)。

### 可用方法

- `.metrics().create_counter()`：创建一个计数器指标
- `.metrics().create_gauge()`：创建一个 gauge 指标
- `.metrics().create_histogram()`：创建一个直方图指标
- `.metrics().create_countervec()`：创建带标签的计数器
- `.metrics().create_gaugevec()`：创建带标签的 gauge
- `.metrics().create_histogramvec()`：创建带标签的直方图

### 创建指标

```rust
use dynamo_runtime::DistributedRuntime;

let runtime = DistributedRuntime::new()?;
let endpoint = runtime.namespace("my_namespace").component("my_component").endpoint("my_endpoint");

// 简单指标
let requests_total = endpoint.metrics().create_counter(
    "requests_total",
    "Total requests",
    &[]
)?;

let active_connections = endpoint.metrics().create_gauge(
    "active_connections",
    "Active connections",
    &[]
)?;

let latency = endpoint.metrics().create_histogram(
    "latency_seconds",
    "Request latency",
    &[],
    Some(vec![0.001, 0.01, 0.1, 1.0, 10.0])
)?;
```

### 使用指标

```rust
// Counter
requests_total.inc();

// Gauge
active_connections.set(42.0);
active_connections.inc();
active_connections.dec();

// Histogram
latency.observe(0.023);  // 23ms
```

### 带标签的向量指标

```rust
// 创建带标签名的向量指标
let requests_by_model = endpoint.metrics().create_countervec(
    "requests_by_model",
    "Requests by model",
    &["model_type", "model_size"],
    &[]
)?;

let memory_by_gpu = endpoint.metrics().create_gaugevec(
    "gpu_memory_bytes",
    "GPU memory by device",
    &["gpu_id", "memory_type"],
    &[]
)?;

// 使用具体标签值
requests_by_model.with_label_values(&["llama", "7b"]).inc();
memory_by_gpu.with_label_values(&["0", "allocated"]).set(8192.0);
```

### 高级特性

**自定义直方图分桶：**
```rust
let latency = endpoint.metrics().create_histogram(
    "latency_seconds",
    "Request latency",
    &[],
    Some(vec![0.001, 0.01, 0.1, 1.0, 10.0])
)?;
```

**常量标签：**
```rust
let counter = endpoint.metrics().create_counter(
    "requests_total",
    "Total requests",
    &[("region", "us-west"), ("env", "prod")]
)?;
```

---

## 相关文档

- [指标概览](metrics.md)
- [Prometheus 与 Grafana 配置](prometheus-grafana.md)
- [分布式运行时架构](../design-docs/distributed-runtime.md)

