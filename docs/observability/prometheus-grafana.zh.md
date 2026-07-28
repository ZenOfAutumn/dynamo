---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Prometheus + Grafana 配置
---

## 概述

本指南介绍如何在单台机器上为 Demo 目的搭建 Prometheus 与 Grafana 以可视化 Dynamo 指标。

![Grafana Dynamo Dashboard](../assets/img/grafana-dynamo-composite.png)

**组件：**
- **Prometheus Server** - 从 Dynamo 服务采集并存储指标
- **Grafana** - 通过查询 Prometheus Server 提供仪表盘

**指标参考**见 [Metrics 文档](metrics.md)。

## 环境变量

| 变量 | 描述 | 默认值 | 示例 |
|----------|-------------|---------|---------|
| `DYN_SYSTEM_PORT` | 系统指标 / 健康端口 | `-1`（禁用） | `8081` |

## 快速开始

这是一个单机示例。

### 启动可观测性栈

启动可观测性栈（Prometheus、Grafana、Tempo、各 exporter）。说明与前置条件见 [可观测性快速开始](README.md#getting-started-quickly)。

### 启动 Dynamo 组件

启动前端与 worker（一个简单的单 GPU 示例）：

```bash
# Start frontend (default port 8000, override with --http-port or DYN_HTTP_PORT env var)
python -m dynamo.frontend &

# Start vLLM worker with metrics enabled on port 8081
DYN_SYSTEM_PORT=8081 python -m dynamo.vllm --model Qwen/Qwen3-0.6B --enforce-eager
```

worker 运行后，发送一些测试请求以填充指标：

```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [{"role": "user", "content": "Hello"}],
    "max_completion_tokens": 100
  }'
```

发送几次请求后，Prometheus Exposition Format 文本指标可在以下位置访问：
- 前端：`http://localhost:8000/metrics`
- 后端 worker：`http://localhost:8081/metrics`

**注意：** 带标签的序列（例如 `...{model="..."}`）只有在第一次匹配请求被处理后才会出现。详情见[可用指标](metrics.md#available-metrics)。

### 访问 Web 界面

Dynamo 组件运行后：

1. 打开 **Grafana**：`http://localhost:3000`（用户名：`dynamo`，密码：`dynamo`）
2. 在左侧栏点击 **Dashboards**
3. 选择 **Dynamo Dashboard** 查看指标和 trace

其他界面：
- **Prometheus**：`http://localhost:9090`
- **Tempo**（追踪）：通过 Grafana 的 Explore 视图访问。详见[Tracing 指南](tracing.md)。

**注意：** 如果从其他机器访问，请将 `localhost` 替换为机器主机名或 IP 地址，并确保防火墙规则允许访问这些端口（3000、9090）。

---

## 配置

### Prometheus

Prometheus 配置位于 [prometheus.yml](https://github.com/ai-dynamo/dynamo/tree/main/deploy/observability/prometheus.yml) 中。该文件已配置为从指标聚合服务端点采集指标。

请注意你可能需要根据具体主机配置与网络环境修改 target 设置。

修改 prometheus.yml 后，重启 Prometheus 服务。Docker Compose 命令见[可观测性快速开始](README.md#getting-started-quickly)。

### Grafana

Grafana 已预先配置：
- Prometheus 数据源
- 用于可视化服务指标的示例仪表盘

### 故障排查

1. 使用 `docker compose ps` 验证服务在运行

2. 使用 `docker compose logs` 检查日志

3. 在 `http://localhost:9090/targets` 检查 Prometheus targets 以验证指标采集

4. 如果遇到陈旧数据或配置问题，停止服务并使用 `docker compose down -v` 擦除卷，然后重启

  **注意：** `-v` 标志会移除命名卷（grafana-data、tempo-data），将重置仪表盘和已存储的指标。

具体的 Docker Compose 命令见[可观测性快速开始](README.md#getting-started-quickly)。

## 开发者指南

关于如何在 Dynamo 组件中创建自定义指标的详细信息，请参阅：

- [Metrics 开发指南](metrics-developer-guide.md)
