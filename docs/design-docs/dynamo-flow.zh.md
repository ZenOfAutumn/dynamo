---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 架构流程
---

下图展示了 NVIDIA Dynamo 的解耦推理（disaggregated inference）系统。不同颜色的连线代表不同类型的流程。

## 🔵 主请求流程（蓝色）
用户视角下贯穿整个系统的主链路：

1. **Request（S1）**：HTTP 客户端向 Frontend（OpenAI 兼容服务，端口 8000）发起 API 请求
2. **Preprocess（S2）**：Frontend 对请求做预处理（应用 chat template、分词）并校验
3. **Route to Prefill（S3）**：PrefillRouter 通过 KV 感知路由或负载均衡选择一个 prefill worker

## 🟢 Prefill 流程（绿色）
Prefill 处理流水线：

4. **Prefill（S4）**：Prefill worker 对输入 token 执行 prefill 计算，生成 KV cache
5. **Return Metadata（S5）**：Prefill worker 返回包含后端特定传输元数据的 `disaggregated_params`

## 🟠 Decode 路由流程（橙色）
Router 把请求编排到 decode 阶段：

6. **Route to Decode（S6）**：PrefillRouter 把 prefill 结果注入到 decode 请求中，并路由到一个 decode worker
7. **KV Transfer（S7）**：Decode worker 与 prefill worker 协调，通过 NIXL 进行直接的 GPU 到 GPU 的 KV cache 传输

## 🟣 完成流程（紫色）
响应生成与下发：

8. **Decode（S8）**：Decode worker 使用传输过来的 KV cache 生成 token
9. **Response（S9）**：生成的 token 流式回传至 Frontend，进行后处理（detokenize）后下发给客户端

## 🔗 基础设施连接（虚线）
负责协调与消息支撑：

### 服务发现
- **在 Kubernetes 上**（默认）：使用 K8s 原生资源（DynamoWorkerMetadata CRD、EndpointSlices）。**不需要** etcd。
- **在裸金属上**：通过 etcd 或文件系统进行服务发现与 endpoint 注册。

### 请求平面（Request Plane）
- **TCP**（默认）：Frontend 与 Worker 之间走直接 TCP 连接，用于请求/响应传输。
- **HTTP / NATS**：可通过 `DYN_REQUEST_PLANE` 切换的备选传输方式。

### NATS 连接（可选，仅 KV 路由需要）
- **KV Events**：用于 KV 感知路由的 cache 状态事件（可通过 `--no-router-kv-events` 关闭）

### Planning 连接（金色虚线）
- **Frontend → Planner**：上报指标，供自动伸缩决策
- **Planner → Workers**：向 worker 下发资源伸缩指令

## 技术实现细节

### PrefillRouter 编排
- `PrefillRouter` 位于 Frontend 与 worker 之间，负责编排解耦服务
- 通过 KV 感知路由（cache 重叠分 + 负载）或简单负载均衡选择 prefill worker
- 把传输元数据注入 decode 请求，用于 KV cache 协调

### NIXL（NVIDIA Interchange Library）
- 通过 NVLink、InfiniBand/UCX 或 PCIe 实现高速的 GPU 到 GPU 数据传输
- 通过 prefill 响应中的 `disaggregated_params` 交换传输元数据
- 后端各异的协调方式：SGLang 使用 bootstrap 连接，TRTLLM 使用不透明 state，vLLM 使用 block ID

### 解耦 KV Cache
- 每个 worker 在自身 GPU 显存中维护本地 KV cache
- **没有共享存储瓶颈** —— 传输直接在 worker 之间通过 NIXL 完成
- 非阻塞传输使得 GPU forward pass 可以在 KV 传输的同时继续进行

```mermaid
%%{init: {'theme':'dark', 'themeVariables': {'primaryColor': '#f4f4f4', 'primaryTextColor': '#333333', 'primaryBorderColor': '#888888', 'lineColor': '#4A90E2', 'sectionBkgColor': '#f9f9f9', 'altSectionBkgColor': '#eeeeee', 'tertiaryColor': '#f0f0f0', 'background': '#ffffff', 'mainBkg': '#f8f8f8', 'secondaryColor': '#f4f4f4', 'nodeTextColor': '#333333'}, 'flowchart': {'htmlLabels': true, 'curve': 'basis'}, 'fontFamily': 'Inter, system-ui, -apple-system, "Segoe UI", Roboto, sans-serif', 'fontSize': '18px'}%%
graph TD
    %% 顶层 - 客户端与 Frontend
    Client["<b>HTTP 客户端</b>"]
    Frontend["<b>Frontend</b><br/><i>OpenAI 兼容 Server<br/>端口 8000</i>"]
    S1[["<b>1 REQUEST</b>"]]
    S2[["<b>2 PREPROCESS</b>"]]

    %% Router 层
    PrefillRouter["<b>PrefillRouter</b><br/><i>编排解耦服务</i>"]
    S3[["<b>3 ROUTE TO PREFILL</b>"]]

    %% 基础设施
    subgraph INF["<b>基础设施层</b>"]
        Discovery[("<b>Discovery</b><br/><i>服务注册<br/>(ETCD 或 K8s)</i>")]
        NATS[("<b>NATS</b><br/><i>KV 事件<br/>(可选)</i>")]
        Planner["<b>Planner</b><br/><i>自动伸缩</i>"]
    end

    %% Worker 层
    subgraph WL["<b>Worker 层</b>"]
        %% Prefill Worker
        PrefillWorker["<b>Prefill Worker</b><br/><i>计算 KV Cache</i>"]
        S4[["<b>4 PREFILL</b>"]]
        S5[["<b>5 RETURN METADATA</b>"]]

        %% Decode Worker
        DecodeWorker["<b>Decode Worker</b><br/><i>Token 生成</i>"]
        S6[["<b>6 ROUTE TO DECODE</b>"]]
        S7[["<b>7 KV TRANSFER</b>"]]
        S8[["<b>8 DECODE</b>"]]
        S9[["<b>9 RESPONSE</b>"]]

        %% KV Cache
        PrefillKVCache[("<b>Prefill KV Cache</b><br/><i>GPU VRAM</i>")]
        DecodeKVCache[("<b>Decode KV Cache</b><br/><i>GPU VRAM</i>")]
    end

    %% 主请求流程（蓝色）
    Client --> S1
    S1 -->|"<b>HTTP/1.1 + JSON</b><br/>POST /v1/chat/completions<br/>(OpenAI Chat Completions)"| Frontend
    Frontend --> S2
    S2 -->|"<b>本地函数调用</b><br/>分词 / chat template /<br/>校验 → PreprocessedRequest"| PrefillRouter
    PrefillRouter --> S3
    S3 -->|"<b>Request Plane</b><br/>TCP (默认) / HTTP2 / NATS<br/>编码：MessagePack 二进制<br/>payload：PreprocessedRequest"| PrefillWorker

    %% Prefill 流程（绿色）
    PrefillWorker --> S4
    S4 -->|"<b>引擎内部</b><br/>CUDA kernel 写入<br/>(GPU VRAM 中的 KV blocks)"| PrefillKVCache
    PrefillWorker --> S5
    S5 -->|"<b>Request Plane</b><br/>响应体：disaggregated_params<br/>(bootstrap host/port、<br/>block ids 等后端特定字段)"| PrefillRouter

    %% Decode 路由流程（橙色）
    PrefillRouter --> S6
    S6 -->|"<b>Request Plane</b><br/>注入 disaggregated_params<br/>payload：PreprocessedRequest +<br/>prefill 传输元数据"| DecodeWorker
    DecodeWorker --> S7
    S7 -->|"<b>NIXL (RDMA/UCX)</b><br/>底层：NVLink / IB / RoCE / PCIe<br/>(无 RDMA 时回退 TCP)"| PrefillKVCache
    PrefillKVCache -.->|"<b>裸 KV block 字节流</b><br/>GPU→GPU 零拷贝，<br/>无序列化"| DecodeKVCache

    %% 完成流程（紫色）
    DecodeWorker --> S8
    S8 -->|"<b>引擎内部</b><br/>逐 token 生成，<br/>读写 GPU 中的 KV blocks"| DecodeKVCache
    DecodeWorker --> S9
    S9 -->|"<b>Request Plane 流式响应</b><br/>chunk = 每 token<br/>{token_id, log_probs, finish_reason}"| Frontend
    Frontend -->|"<b>HTTP SSE</b><br/>data: {...}\\n\\n<br/>(detokenize 后)"| Client

    %% 基础设施连接
    Frontend -.->|"<b>etcd gRPC/HTTP2</b> 或 <b>K8s HTTPS</b><br/>watch 注册表，发现 worker"| Discovery
    PrefillRouter -.->|"<b>etcd gRPC/HTTP2</b><br/>watch worker endpoints"| Discovery
    PrefillWorker -.->|"<b>etcd lease + put</b><br/>注册 EndpointInfo<br/>{endpoint, addr, metadata}"| Discovery
    DecodeWorker -.->|"<b>etcd lease + put</b><br/>注册 EndpointInfo"| Discovery
    Planner -.->|"<b>etcd watch</b><br/>发现组件实例"| Discovery

    %% NATS（KV 事件，可选）
    PrefillWorker -.->|"<b>NATS Core 或 ZMQ PUB</b><br/>StoreEvent / RemoveEvent<br/>~100B/条，批量约 10s<br/>{sequence_hash, prefix_hash,<br/>block_size, event_type}"| NATS
    DecodeWorker -.->|"<b>NATS Core 或 ZMQ PUB</b><br/>KV 事件 + ForwardPassMetrics"| NATS

    %% Planning 连接
    Frontend -.->|"<b>Prometheus pull (HTTP)</b><br/>/metrics 端点<br/>TTFT / ITL / 请求数 / ISL / OSL"| Planner
    Planner -.->|"<b>K8s API (HTTPS)</b><br/>PATCH DGD CRD<br/>spec.replicas = N (JSON)"| PrefillWorker
    Planner -.->|"<b>K8s API (HTTPS)</b><br/>PATCH DGD CRD"| DecodeWorker

    %% 样式
    classDef client fill:#e8f5e8,stroke:#2E7D32,stroke-width:3px
    classDef frontend fill:#fff3e0,stroke:#F57C00,stroke-width:3px
    classDef router fill:#f3e5f5,stroke:#7B1FA2,stroke-width:3px
    classDef worker fill:#e3f2fd,stroke:#1565C0,stroke-width:3px
    classDef prefillWorker fill:#e8f5e9,stroke:#388E3C,stroke-width:3px
    classDef planner fill:#f1f8e9,stroke:#558B2F,stroke-width:3px
    classDef storage fill:#e0f2f1,stroke:#00695C,stroke-width:3px
    classDef discovery fill:#fff9c4,stroke:#F9A825,stroke-width:3px
    classDef nats fill:#ede7f6,stroke:#5E35B1,stroke-width:3px
    classDef infraLayer fill:#fff9c4,stroke:#FFC107,stroke-width:3px
    classDef workerLayer fill:#e3f2fd,stroke:#2196F3,stroke-width:3px

    class Client client
    class Frontend frontend
    class PrefillRouter router
    class DecodeWorker worker
    class PrefillWorker prefillWorker
    class Planner planner
    class PrefillKVCache,DecodeKVCache storage
    class Discovery discovery
    class NATS nats
    class INF infraLayer
    class WL workerLayer

    %% 连线颜色
    %% 主请求流程 - 蓝色
    linkStyle 0,1,2,3,4,5 stroke:#1565C0,stroke-width:4px

    %% Prefill 流程 - 绿色
    linkStyle 6,7,8,9 stroke:#2E7D32,stroke-width:4px

    %% Decode 路由流程 - 橙色
    linkStyle 10,11,12,13,14 stroke:#E65100,stroke-width:4px

    %% 完成流程 - 紫色
    linkStyle 15,16,17,18,19 stroke:#6A1B9A,stroke-width:4px

    %% 基础设施 - 灰色虚线
    linkStyle 20,21,22,23,24,25,26,27,28,29 stroke:#757575,stroke-width:2px,stroke-dasharray: 8 8

