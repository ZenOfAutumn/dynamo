# 阶段 0：Dynamo 整体认知 — 学习笔记

> 对应 [`LEARNING_PLAN.zh.md`](../LEARNING_PLAN.zh.md) 阶段 0 自检。
> 完成时间：2026-05

## 一、自检回答（我的理解）

**Frontend**：前端模块，负责接收 request、pre 处理和后处理、发送 response，是流量接入层。

**Router**：路由模块，负责根据负载或性能将请求转发到 prefill / decode 节点。最重要的策略是 **KV Cache Router**，使用加权公式综合考虑 KV cache 重叠率和系统负载来路由请求。

**Worker**：后端推理引擎节点，负责真正处理 prefill 和 decode 的前向传播。

**KVBM**：多级 KV cache 存储管理框架，负责管理 G1~G4 KV Cache 的 onboard、offload 和感知调度。

**Planner**：集群扩缩容决策中心，负责基于吞吐和基于负载两种算法模式下的集群扩缩决策。

---

## 二、复盘与修正

### ✅ Frontend（满分）

完全正确。补充：
- **还做 tokenization**（把字符串转成 token ids 发给 worker）
- **OpenAI 协议适配**（让 dynamo 看起来像 OpenAI API）
- 在 KV-router 模式下还**内嵌 router**

### ✅ Router（基本满分）

正确。两个小补充：
- 路由的是 **worker**，不是 "prefill/decode 节点"——这两个角色由 worker 自己的启动模式决定，router 只是按候选池调度
- 加权公式叫 **cost function**，大致是 `cost = α × (1 − prefix_hit_ratio) + β × queue_load`，谁 cost 低发给谁

### ⚠️ Worker（小瑕疵）

正确，但严格说 worker 是个**封装层**，真正跑前向的是它内部包的 **vLLM / SGLang / TRT-LLM / Mocker**。Worker 的职责是：
- 接 dynamo runtime 的请求
- 调用引擎做推理
- 把 KV 事件发回去（让 router 知道我现在缓存了什么）
- 把负载指标发回去（让 planner 决策）

### ⚠️ KVBM（需要修正一个误解）

正确，但要分清边界：
- KVBM **不做"感知调度"**（那是 router 的事）
- KVBM 做的是 **block 生命周期管理 + 跨层级 transfer + 事件广播**
- 它是**被动**的：引擎/router 说"我要这个 block"，KVBM 负责找到它（在哪一层）并送到 GPU；不是"主动决定该不该 cache"

简单说：**KVBM 是搬运工 + 仓库管理员，router 才是调度员**。

### ✅ Planner（满分）

完全正确。再补一句它的定位：**它只下决策，不真正扩缩**——真正的执行是通过 K8s Operator（KubernetesConnector）或外部系统（VirtualConnector）。

---

## 三、漏掉的三个关键模块

阶段 0 完整图里还有这三个：

| 模块 | 一句话 |
|---|---|
| **Discovery Plane**（etcd / file / K8s） | 服务注册与发现，让 frontend / router / worker 互相找得到 |
| **Event Plane**（NATS / ZMQ） | KV 事件、负载指标的 pub/sub 总线，是 KV-router 和 planner 的"耳朵" |
| **Request Plane**（TCP / HTTP / NATS） | RPC 请求传输层，决定 frontend → worker 的数据走哪条路 |

这三个"平面"是 Dynamo 区别于"传统推理服务"的核心架构特征——**没有一个 master，所有协调通过三个 plane 完成**。

---

## 四、整体架构图（自画，含通信协议与数据格式）

```
                       ┌──────────────┐
                       │  HTTP Client │
                       └──────┬───────┘
                              │
              ① HTTP/1.1 + JSON（OpenAI Chat Completions）
                              │  POST /v1/chat/completions
                              ▼
                       ┌──────────────┐
                       │   Frontend   │  ← tokenize、协议适配
                       └──────┬───────┘
                              │
              ② Request Plane（默认 TCP，可选 HTTP/2 / NATS）
                              │  payload：PreprocessedRequest
                              │   {token_ids, sampling_params,
                              │    stop_conditions, output_options}
                              │  序列化：MessagePack / 二进制
                              ▼
                       ┌──────────────┐
                       │    Router    │  ← cost = α·(1-hit) + β·load
                       └──────┬───────┘
                              │
              ③ Request Plane（同上 TCP/HTTP/NATS）
                              │  payload：含 disaggregated_params
                              │  （prefill 元数据，给 decode worker）
                  ┌───────────┼───────────┐
                  │           │           │
                  ▼           ▼           ▼
              ┌──────┐    ┌──────┐    ┌──────┐
              │  W1  │    │  W2  │    │  W3  │   vLLM / SGLang / TRT-LLM
              │ KVBM │    │ KVBM │    │ KVBM │
              └──┬─┬─┘    └──┬─┬─┘    └──┬─┬─┘
                 │ │         │ │         │ │
        ┌────────┘ │         │ │         │ └────────┐
        │          │         │ │         │          │
        │   ④ NIXL (RDMA / NVLink / IB / TCP)       │
        │          KV cache 直传，二进制 KV blocks   │
        │          (Prefill Worker ↔ Decode Worker)  │
        │          │         │ │         │          │
        │          ▼         ▼ ▼         ▼          │
        │  ⑤ Response 流式回 Frontend（沿 ② 反向）  │
        │     SSE 风格 chunk：每生成一个 token       │
        │     立刻回传（token_id + log_probs）        │
        │                                            │
        │  ⑥ Event Plane（NATS Core / ZMQ PUB）       │
        │     KV 事件：StoreEvent / RemoveEvent       │
        │     {sequence_hash, prefix_hash, block_size,│
        │      storage_location, event_type}          │
        │     每条 ~100B，批量发布（约 10s）          │
        │                                            │
        │  ⑦ Event Plane                              │
        │     Worker 指标：ForwardPassMetrics(FPM)    │
        │     {wall_time, scheduled_requests,         │
        │      queued_requests, kv_usage...}           │
        ▼                                            ▼
   ┌──────────┐                                 ┌──────────┐
   │  Router  │ ◄── 订阅 KV 事件，更新 prefix  │ Planner  │
   │ (副本同步)│     树和 worker 负载视图       │          │
   └──────────┘                                 └────┬─────┘
                                                     │
                                  ⑧ K8s API (HTTPS) 或 VirtualConnector
                                     PATCH DGD：replicas=N
                                     {prefill: 3, decode: 5}
                                                     │
                                                     ▼
                                            （K8s Operator 扩缩 Pod）

   ──────────────────────────────────────────────────────────────
   ⑨ Discovery Plane（全员都连，注册自己 + 发现别人）
      - etcd：gRPC over HTTP/2，KV 注册：
          /dynamo/namespaces/<ns>/components/<comp>/instances/<id>
          value = {endpoint, address, metadata, lease_id}
      - K8s：DynamoWorkerMetadata CRD + EndpointSlices（HTTPS）
      - file/mem：本地开发用，文件 JSON / 内存 map
```

### 通信协议速查表

| 路径 | 协议 | 编码 | 触发频率 | 典型 payload |
|---|---|---|---|---|
| ① Client → Frontend | HTTP/1.1 或 HTTP/2 | JSON | 每请求 | OpenAI Chat/Completions |
| ② Frontend → Router | Request Plane（TCP 默认） | MessagePack 二进制 | 每请求 | PreprocessedRequest |
| ③ Router → Worker | Request Plane（同上） | MessagePack 二进制 | 每请求 | 含 disaggregated_params |
| ④ Worker ↔ Worker（KV 传输） | NIXL（RDMA / NVLink / IB / TCP fallback） | 裸 KV block 字节流 | 每个 prefill→decode 切换 | KV cache blocks |
| ⑤ Worker → Frontend（响应流） | 沿 ② 反向 + Frontend 再转 SSE 给 client | 二进制流 → SSE/JSON | 每生成一个 token | token_id + log_probs |
| ⑥ Worker → Router（KV 事件） | Event Plane（NATS Core 或 ZMQ PUB） | Protobuf / JSON | 批量约 10s | StoreEvent / RemoveEvent |
| ⑦ Worker → Planner（指标） | Event Plane（同上） | Protobuf | 每个 forward pass | ForwardPassMetrics |
| ⑧ Planner → K8s | Kubernetes API（HTTPS） | JSON（CRD） | 每个 adjustment_interval | DGD spec.replicas |
| ⑨ 全员 ↔ Discovery | etcd（gRPC/HTTP2）/ K8s API（HTTPS） | Protobuf / JSON | 注册一次 + 监听 watch | EndpointInfo |

### 三个平面的协议选项

| 平面 | 默认 | 备选 | 控制开关 |
|---|---|---|---|
| **Request Plane** | TCP | HTTP/2、NATS JetStream | `DYN_REQUEST_PLANE` |
| **Event Plane** | ZMQ（本地）/ NATS（分布式） | 仅二选一 | `DYN_EVENT_PLANE` |
| **Discovery Plane** | etcd（生产）/ file（本地） | K8s CRD / mem | `--discovery-backend` |

**整体逻辑**：
Frontend 收 HTTP 请求 → tokenize → 通过 Request Plane 发给 Router → Router 按 KV+负载选 worker → Worker 推理（PD 分离时走 NIXL 直传 KV）→ token 沿原路流式回传 → KV 事件/指标通过 Event Plane 广播给 Router/Planner → Planner 通过 K8s API 调整副本数 → Discovery Plane 让全员互相找到。

---

## 五、本阶段沉淀的核心概念

| 概念 | 关键点 |
|---|---|
| **TTFT** | Time To First Token，首 token 延迟 |
| **TPOT / ITL** | 每输出 token 时长 / token 间间隔，TPOT ≈ mean(ITL) |
| **ISL / OSL** | Input / Output Sequence Length |
| **PD 分离** | Prefill / Decode 分离式服务，分别优化两阶段的算力特性 |
| **KV-aware routing** | 按 prefix 命中率 + 负载选 worker，复用历史 KV |
| **stutter** | 流式输出 token 不匀速（看 ITL 分布的 p95/p99 才能暴露） |
| **三个平面** | Discovery / Request / Event 各司其职，无中心 master |
| **四级 KV 层级（G1~G4）** | GPU → CPU pinned → NVMe SSD → 远程 / 对象存储 |
| **adjustment_interval** | Planner 扩缩间隔，必须 > 端到端启动时间（含模型分发） |
| **cost function** | Router 选 worker 的核心公式，调 α/β 权衡 KV 命中 vs 负载 |

---

## 六、未解问题（留待后续阶段）

- [ ] cost function 的 α/β 默认值是多少？怎么调？→ 阶段 2 router
- [ ] KVBM 的 NIXL/RDMA 在没有 RDMA 网卡时怎么 fallback？→ 阶段 4 KVBM
- [ ] Planner 的 ARIMA/Prophet/Kalman 预测器实际生产用哪个？→ 阶段 5 planner
- [ ] 多 frontend 实例之间如何避免重复决策？→ 阶段 2 router 副本同步
- [ ] mocker 跟真实 vllm 的 KV 事件能否互通？→ 阶段 3 worker

---

## 七、阶段 0 → 阶段 1 的衔接

下一阶段重点：**Discovery / Request / Event 三个平面的实现细节**

- 三个平面的代码入口在哪？
- etcd / NATS / ZMQ 各自被封装在哪个 crate / module？
- 本地开发用 `--discovery-backend file` 时，文件结构是什么样？
- KV 事件的具体 schema？

