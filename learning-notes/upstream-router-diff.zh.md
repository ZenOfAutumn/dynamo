# 上游 Router 能力扩展详解（对比 friday 内部分支）

> 对比基线
> - **上游**：本仓库 `main`（`094aaca91`，2026-10-14，版本 `1.6.0`）
> - **friday 分支**：`/Users/wuliang/Workspace/friday/dynamo`（`069a83f4d`，2026-10-12，版本 `1.2.0`）
> - 两者共同祖先：`38782e0a0`（2026-05-12，`Support a single snapshot agent… (#8740)`）。分叉后上游 +2968 提交，friday +297 提交。
>
> 信息来源：上游 `docs/fern/pages/developer-guide/knowledge-base/modular-components/router/*`、
> `docs/fern/pages/reference/general/releases/dynamo-v1-{3,4,5}-0.mdx` 以及对应提交的源码注释。
> 标注 "v1.6 / main" 的条目尚无发版说明，依据代码与文档整理。

分叉后，上游在 router / kv-router / selection / epp 方向约有 90 个 feat 提交，按 7 个主题展开如下。

---

## 一、插件化：把"选哪个 worker"变成可替换组件

### 1.1 Worker 选择三段式：Filter → Scorer → Picker（v1.5，#12526 / #12536 / #12897）

**问题**：原来代价函数写死，想换策略只能改 Dynamo 源码。

**机制**：Dynamo 仍负责发现、资格校验、排队、预约、记账，只把"排序/挑选"交给插件。

```
请求 → Dynamo 资格过滤 → WorkerFilter(s) 硬性保留/剔除
     → WorkerScorer(s) 每个 worker 写一个有限 cost，多个 scorer 相加
     → WorkerPicker 选一行 → Dynamo 校验 → 记账/预约
```

| 阶段 | 输入 | 输出 | 用途 |
|---|---|---|---|
| `WorkerFilter` | 请求上下文 + 单个 worker | `true` 保留 / `false` 剔除 | 硬性要求（如最少 device 命中块数） |
| `WorkerScorer` | 全部存活 worker + 对齐的输出切片 | 每个 worker 一个有限 cost | 软偏好、惩罚项 |
| `WorkerPicker` | 全部候选 + 总 cost | 一个行号 | 采样、打破平局、策略内部状态 |
| Policy factory | 配置 + `WorkerType` + 路由分区 | 一个 `WorkerSelectionPolicy` | P / D / 不同模型用不同组件 |

- **按需计算信号**：组件通过 `required_worker_inputs()` 声明 `CACHE` / `LOAD` / `PREFERRED_TAINT`，Dynamo 只计算被声明的数据。
  - `CACHE`：`device/host/disk_overlap_blocks()`、`shared_hits()`
  - `LOAD`：`active_prefill_tokens()`、`decode_cost_blocks()`、`active_requests()`
- **请求上下文**：`session_context()`（session_id / parent / agent headers）、`affinity_target()`、`expected_output_tokens()`、`strict_priority()` 等。
- 所有 worker 都被 filter 剔除时返回 HTTP 503。
- **Java 类比**：像 Spring 里注入 `List<Filter>` + `List<Scorer>` + 一个 `Picker` 策略 Bean，通过 SPI 注册、由 YAML 选择实例。
- 代码：`lib/kv-router/src/plugins/worker_selection`；示例：`examples/router/custom-policy-example`。

### 1.2 插件注册表与 YAML 实例目录（v1.5 #12536；v1.6 统一为 `RouterPluginRegistry`，#14149）

- 插件是 Rust crate，编译期链接：自定义插件需用 `--features custom-policy` 重新构建 Python 扩展。启动时由 YAML 选择类型。
- 配置示例：

  ```yaml
  worker_selection:
    aggregated: least-busy
    prefill: prefill-least-busy
    decode: least-busy
    instances:
      - name: least-busy
        type: least-busy
        parameters: { min_device_overlap_blocks: 0 }
  ```

  - P / D / Agg / Encode 四个角色各自选择实例。
  - 某角色未配置时回退内置选择器，不继承其他角色的配置。
- 配置入口：`--router-policy-config` / `DYN_ROUTER_POLICY_CONFIG`。
- 类型未知、重复注册或参数非法时直接启动失败，不会静默回退。

### 1.3 内置策略（v1.5，#14498；默认开启 `custom-policy`）

| 类型 | 行为 |
|---|---|
| `default` | 原内置选择器与代价函数 |
| `dynamo-default-cost-fn` | 原代价函数，但参数可在 YAML 中按角色单独配：`overlap_score_credit`、`overlap_score_credit_decay`、`prefill_load_scale`、`decode_active_request_weight`、`host/disk_cache_hit_weight`、`shared_cache_multiplier`、`router_temperature` |
| `dynamo-two-tier-cost-fn` | 两级排序（不再把各项加成一个总代价），移植自 SGLang router 的 `cache_aware_zmq` 策略，规则见下 |
| `thunderagent` | 与 ThunderAgent classifier 配对使用，见 1.6 |

`dynamo-two-tier-cost-fn` 的规则：
- 活跃请求数的差距 > 32，且最大值 > 最小值 × 1.1 时，进入负载层，选负载最低的 worker；
- 否则，若某 worker 的 device overlap 超过请求块数的 50%，选 overlap 最大者。
- 阈值可调：`cache_threshold`、`balance_abs_threshold`、`balance_rel_threshold`。

上游同时把 `--router-kv-overlap-score-credit`、`--router-temperature` 等 CLI 参数标记为 **v1.7 移除**，迁移到 YAML 的 `parameters`。

### 1.4 Request Classifier：准入控制插件（v1.6 / main，#14177 / #14149 / #15162）

**作用**：请求进入调度队列之前，决定放行、延后还是拒绝。

- **接口**：`RequestClassifier::classify(request) -> Future`。
  - 返回 `Ok(request)`：放行；
  - 在 future 里异步等待：延后；
  - 返回 `Err(DynamoError)`：拒绝。`CapacityExhausted` 默认 529，`RateLimited` 为 429。
- **放行前可修改**：
  - `set_policy_class`：指定进入哪个队列
  - `set_due_at`：排队截止时间
  - `set_scheduling_cost_tokens`：覆盖调度代价
  - `set_worker_selection_target`：软目标 worker
- **可读输入**：`request_id`、`session_context`、`input_tokens`、`ingress_at`、`progress().context_tokens()`（实时上下文高水位），以及 `sequence_hashes()`（#15162，准入前的 block hash 快照）。
- **生命周期事件** `on_event`：`Sent` / `Responding` / `Completed` / `Aborted`，用于释放容量、唤醒等待中的请求。
- **限制**：只作用于 aggregated / decode 路由，prefill hop 绕过 classifier；standalone selection / EPP 不支持 classifier。

### 1.5 Router flow control（v1.6 / main，#14123）

把 classifier 接入请求全生命周期。改动集中在 `request_classifier.rs`、`routing_host/kv.rs`、`request_guard.rs`、`migration.rs`：
- 新增 `RequestProgress`：宿主持续上报 prompt + 已生成 token，形成上下文高水位；
- 软目标 `set_worker_selection_target`；
- 请求迁移（migration）重试时保持 classifier 生命周期一致；
- 同一 request_id 被复用时，严格保证上一轮的终止事件先于下一轮 `classify` 送达（generation 隔离）。

### 1.6 ThunderAgent 内置插件（v1.3 实验服务 #9448 → v1.6 内置 #15164）

- 实现论文 ThunderAgent（arXiv 2602.13692）的"程序级调度"：把一个 agent 的多轮推理与工具调用视为一个 program，按 program 暂停/恢复，并按 worker 容量记账。
- 原独立服务 `dynamo.thunderagent_router`，现移入 `lib/router-plugins/builtin/src/thunderagent/`：
  - classifier：维护 program 表；session 忙碌或被暂停时延迟准入；放行时给出偏好的 worker/rank；
  - selector：目标合格则用，否则选负载最低者。
- 需配合 `--router-session-affinity-mode soft` 才能把 session 迁到其他 worker（重新打包）；会话结束时发 `X-Dynamo-Session-Final: true` 释放状态。

**friday 分支现状**：1.1～1.6 全部缺失（无 `WorkerScorer`、`RequestClassifier`、`router-plugins`）。friday 的 length-aware、P-first、负载衰减 credit 是直接改在 `selector.rs` / `prefill_router` 里的。升级上游时适合改写为 scorer / classifier 插件，而非继续在核心代码上打补丁。

---

## 二、会话亲和（Session Affinity）

### 2.1 Header 识别 session + 兼容主流 coding agent（v1.3，#10875）

- 标准头：`X-Dynamo-Session-ID`、`X-Dynamo-Parent-Session-ID`、`X-Dynamo-Session-Final`，映射到 `agent_context.session_id` / `parent_session_id` / `session_final`。
- 自动识别并归一化 Claude Code（`x-claude-code-session-id` / `-agent-id` / `-parent-agent-id`）、Codex（`x-codex-parent-thread-id` 等）、OpenCode 的原生头。
- 开启：`--router-session-affinity-ttl-secs`（1～31536000，空闲多久过期）。只带 header、不配 TTL 时只有身份信息，不做亲和。

### 2.2 粒度细化到 DP rank（v1.3，#9920）

绑定从 `session → worker` 细化为 `session → (worker, dp_rank)`，多 DP rank 引擎上 session 能留在前缀已热的 rank。

### 2.3 hard / soft 两种模式（v1.5，#13907）

| 模式 | 行为 |
|---|---|
| `hard`（默认） | 精确派发到已绑定目标；目标失效则作废绑定，重新走一次正常选择 |
| `soft` | 已绑定目标作为建议（`affinity_target()`）进入选择流程。内置选择器在目标合格时保留它，自定义策略可改选；拿到响应流后才提交新绑定 |

- 绑定带版本号，防止较旧的并发请求覆盖较新的重绑定。
- 有活跃请求时不过期，请求结束后才重新计算空闲 TTL。

### 2.4 跨 router 副本同步（v1.4，#11079 → #11217 回滚 → #11750 再上线）

- 在 runtime event plane 上广播绑定：派发成功时发一次，请求租约结束时再发一次，让其他副本刷新空闲计时。
- "首个存活绑定生效"语义，尽力而为：事件丢失或乱序不影响正确性，只会暂时降低亲和效果。
- 若要求严格亲和，文档建议在 ingress / 负载均衡层按原始 session header 做一致性哈希。

### 2.5 子 agent 归入父 session 的组（v1.6，#14521）

- 带 parent session id 的请求，绑定 key 由 parent id 派生：同一父 agent 的子 agent 共享一个绑定，父 agent 保留自己的绑定。
- 已知问题：兄弟请求并发扇出时，要等第一个兄弟派发并提交绑定后，其余请求才能放置。

### 2.6 宿主可控的亲和时钟（v1.6 / main，#15631）

`SessionAffinity` 表支持宿主传入单调时钟，不依赖 tokio runtime 和后台清理任务，由宿主推进时间、触发清理。看起来是为 replay / 仿真、嵌入式宿主准备的。

### 2.7 SessionPrefixIndexer（v1.6，#13807）

- 新增 `lib/kv-router/src/session_prefix_index.rs`（1172 行），源码说明为"独立于物理缓存淘汰的会话谱系追踪"。
- 为每个 session 维护逻辑 block 链（森林），记录 route 时匹配与 store 事件，block 被删除时收缩 session 前沿。
- 通过 `router.enable_session_prefix_index` 开启（要求 `CACHE` 输入）。即使物理 KV 已被淘汰，策略也能知道某 session 的前缀谱系落在哪个 worker。

**friday 分支现状**：只有较早的 `StickySessionRouter`（`lib/llm/src/kv_router/sticky_sessions.rs` + `agent_controller.rs`，基于 nvext 的 bind/open 动作）。header 识别、hard/soft、副本同步、子 agent 分组、SessionPrefixIndexer 均没有。

---

## 三、调度与排队

### 3.1 Policy-class 队列 + DRR + exact-worker 子队列（v1.3 #10744、v1.4 #11363 / #11434 / #11615）

- 客户端通过 `x-dynamo-meta-policy-class` 声明 family。router 计算 `uncached ISL = 原始 ISL − 所有合格 worker 中最好的缓存命中`，把 (family, cache bucket) 映射到一个物理 class。
- 每个 class 有一个共享堆（FCFS / WSPT）+ exact-worker 子队列，以及各自的 busy 阈值、quantum、deficit、计数器。
- 多个 class 之间用**加权 DRR（Deficit Round Robin，赤字轮询）**仲裁：
  - quantum 以未命中缓存的 token 计，按轮发放而非按请求发放；
  - 超大请求一次性计算所需轮数，避免空转。
- 队列上限按 worker 数缩放（`request_queue_limit_per_worker` 等），超限返回结构化 529。
- v1.4 新增公开准入策略接口 `PolicyClassAdmissionStrategy`（#11434）与实时 `RequestProgress`（#11615）。
- `--router-queue-threshold` / `--router-queue-policy` 在 v1.6 弃用，v1.7 移除。

**friday 分支现状**：#49 已手工迁移 #10744 和 #11363；`PolicyClassAdmissionStrategy`、`RequestProgress` 和后续的 deadline 排序未跟进。

### 3.2 严格优先级档（v1.3，#10638）

- 通过 `nvext.agent_hints.strict_priority` 或 `x-dynamo-request-strict-priority` 设置。高档位绝对优先，档内再按 FCFS / LCFS / WSPT 排序。
- 与 `priority_jump`（相对加权）不同。
- 队列完整排序键：`(strict_priority, due_at, policy_key)`。

### 3.3 Deadline-aware 排序（EDF）（v1.6，#14176）

- 同一档位内，带截止时间的请求按"截止最早优先"排在所有不带截止时间的请求之前，然后才比较 FCFS/WSPT 分数。源码注释的理由：必须按时完成的请求不能为了缓存亲和让出位置。
- 定时器到点时剔除已过期条目并计数。
- 实现细节：为让队列条目头部保持在 64 字节（一个 cache line）内，删掉了派生字段 `uncached_tokens`，否则 drain benchmark 退化 50～150%。

### 3.4 出队时刷新 overlap（v1.3，#9663）

排队超过 `DYN_ROUTER_OVERLAP_REFRESH_AFTER_SECS`（默认 10 秒）的请求，出队时重新查询缓存命中，避免用过期分数路由。

### 3.5 Predict-on-route（v1.3，#8276）

- **问题**：同批次兄弟请求（并行采样、best-of-N、agent 扇出）在引擎上报 "block stored" 之前都被评为零命中，被轮询分散，每个 worker 都重复 prefill 一遍。
- **做法**：`--router-predicted-ttl-secs 5` 开启短 TTL 的旁路近似索引，每次路由决策都写入；打分时对每个 worker 取主索引与旁路索引的较大值。
- 主索引不受影响：引擎 hash 带盐，router 无法复现，不能往主索引写。

### 3.6 Conditional disaggregation（条件式 PD 分离）（v1.4 #11718 内核 → v1.5 #11723 / #11725 编排）

- 每个请求单独决定：走 "prefill worker → decode worker"，还是直接发给 decode worker 在本地完成 prefill + decode。
- 判断依据是 decode 侧 KV 命中：`effective ISL = prompt 长度 − 选中 decode worker 已缓存的前缀`。

  | 策略 | 走本地 P+D 的条件 |
  |---|---|
  | `isl_bounding`（默认） | effective ISL < `eff_isl_threshold`（默认 2048）且 effective/raw < 0.7 |
  | `prefill_load` | 选中的 prefill worker 超过忙碌阈值 |
  | `isl_or_load` | 满足任一 |

- `decode_busy_threshold` 保护：decode worker 的 KV 占用过高时仍走远程 prefill。
- 开启：`--router-conditional-disagg` + `--router-conditional-disagg-config '{...}'`。要求 decode worker 发布 KV 事件；支持 vLLM / TRT-LLM，不支持 SGLang。
- 同时把 `--enforce-disagg` 改为接受但不生效的参数。

**friday 分支现状**：没有。friday 的 P-first / D-first / length-aware 是另一种思路：按长度分档决定先选 P 还是先选 D。上游方案的核心是"decode 侧有没有缓存命中"，两者可互补。

### 3.7 代价函数增强

| 特性 | 说明 |
|---|---|
| overlap credit 衰减（v1.3，#10574） | `overlap_score_credit_decay`：worker 比最空闲者多出 prefill 负载时，device credit 乘以 `1/(1+decay·超出量)`。friday #30 参考它自研了一版 |
| 放大的 overlap credit（v1.4，#11714） | `overlap_score_credit` 允许 > 1，取消调整后 prefill blocks 的 0 下限，让缓存命中可抵消 decode 负载 |
| 活跃请求数计入 decode 代价（v1.4，#12158） | `decode_active_request_weight`，适合 decode 受 batch size 影响大（接近计算瓶颈）的模型，如部分 MLA / MTP 配置 |
| decode affinity scoring（v1.4，#11720） | 选择器新增 decode 亲和打分路径 |
| host/disk 缓存层权重（v1.3，#10157） | `host_cache_hit_weight`（默认 0.75）、`disk_cache_hit_weight`（默认 0.25）。friday #24 只有 host 一项 |
| AIS prefill 负载模型 | `--router-prefill-load-model ais`：用 AISimulate 预测每个请求的 prefill 时长，按时间衰减最老的活跃 prefill，不再"首 token 前一直按静态负载计" |
| 已计算 prefill 时间可读（v1.3，#10190） | slot tracker 暴露每个 worker 预计剩余的 prefill 毫秒数 |

最终打分公式（上游文档原文）：
`prefill_load_scale × adjusted_prefill_blocks + potential_decode_blocks + decode_active_request_weight × active_requests`

### 3.8 容量受限的近似 LRU（v1.5，#13701）

- 不用 KV 事件（`--no-router-kv-events`）时，原近似索引只能靠 TTL 过期。
- 新增 `--router-approximate-cache-policy lru`：按每个 DP rank 上报的 `total_kv_blocks` 模拟物理容量，压力下淘汰最久未用、无引用的副本。
- 代码：`lib/kv-router/src/indexer/approximate_lru.rs`（1858 行）。

### 3.9 其他

- **活跃请求过期时间可配**（v1.4，#11303）：`DYN_ROUTER_ACTIVE_REQUEST_EXPIRY_SECS`（默认 300 秒），避免长上下文 agent 请求未结束就被清掉活跃记录。
- **负载感知内置策略**（v1.5，#13513）：`power-of-two` / `least-loaded` 改为基于共享、原子化的占用记账（`lib/runtime/src/routing_policy/occupancy.rs`）。worker 被移除后计数器保留到最后一个预约释放，重新加入时能看到遗留负载。friday 有这两个 mode，但是旧实现。
- **队列深度背压**（v1.3，#8144）：按请求代价（未命中 token 数）限制每个 worker 的队列深度，超限立即返回 503。

---

## 四、拓扑与约束

### 4.1 RoutingConstraints + worker taints（v1.3，#9558）

- 请求侧可带：
  - `required_taints`：硬约束，缺少该 taint 的 worker 直接排除；
  - `preferred_taints`：软约束，命中的 worker 代价乘以系数。
- worker 在 `ModelRuntimeConfig` 声明自己的 taints。
- 类比 Kubernetes 的 taint/toleration 与 node affinity 的 required/preferred。

### 4.2 动态更新 taint（v1.5，#12620）

新增 Python API `update_model_taints` 与 `POST /engine/update/model_taints`（vLLM / SGLang / TRT-LLM），worker 不重启即可修改路由元数据，并唤醒挂起的调度请求。

### 4.3 拓扑感知 KV 传输（v1.3，#9767 / #9878 / #9893）

- worker 发布 `topology_domains`（如 `{"zone":"us-east-1a","rack":"rack-22"}`），每项自动生成 taint `dynamo.topology/<domain>=<value>`。
- 选定 prefill worker 后，按它的 `kv_transfer_domain`（如 zone）生成约束并合并进 decode 请求：
  - `required`：decode 必须与 prefill 同 zone，否则失败。约束在派发 prefill 前生成，保证 fail-closed；
  - `preferred`：仅加权倾斜，权重 0～1。
- K8s 用 DGD 的 `spec.experimental.kvTransferPolicy`，operator 注入拓扑文件；非 K8s 用 `DYN_TOPOLOGY_*` / `DYN_KV_TRANSFER_*` 环境变量。

### 4.4 worker 自报路由配置（v1.5，#13192 / #13197）

- worker 在模型部署卡（MDC）中声明本 worker set 的路由参数，整体替换 frontend 默认值（不逐项合并）。
- prefill hop 改用 prefill worker 自己声明的配置；此前会错误继承 decode set 的 mode。
- 配套 **worker-set 准入**：同一 worker set 内所有副本的 MDC checksum 必须一致。首个被看到的卡占住配置；checksum 不同的副本不接流量，也无法扰动现有配置。滚动升级时用版本化 namespace 区分新旧 worker set。

**friday 分支现状**：4.1～4.4 均没有。friday 的 worker-tag PD 配对（`support worker-tag-pd-pairing`）与 taint 部分重叠，但只是内部实现，没有通用约束模型。

---

## 五、独立服务与网关（脱离 Dynamo runtime 的选择能力）

### 5.1 Standalone selection service（v1.3 #10641，v1.4 #11416 / #12335）

- 启动：`python -m dynamo.select_service --port 8092`。通过 HTTP 提供选择与活跃负载记账，**不转发推理请求**。
- 适用于 Ray 等外部运行时：外部系统注册 worker 目录、请求选择、自行连接选中的 worker，再回报预约生命周期。
- 能力：
  - 从 ZMQ 事件构建 KV 索引；
  - 选择与预约可分两步，也可原子化一次完成；
  - selector 副本间尽力同步；
  - 启动时从其他 selector 或 standalone indexer 恢复索引。
- `SelectionCache` 以 `(model, routing_group, selection_id)` 为 key，后续 `POST /reservations` 复用缓存的预约，不必重传长 prompt 再做 hash。
- Rust 可通过 `SelectionServiceBuilder` 嵌入。

### 5.2 Standalone slot tracker（v1.3，#10291 / #10394）

- 启动：`python -m dynamo.slot_tracker --port 8091`，只负责活跃请求记账。
- 调用方传每个 block 的链式 hash（不传 token），返回仅供参考的负载快照。
- 可选：多个 slot tracker 之间通过 ZMQ 直接复制生命周期事件。

### 5.3 统一 frontend 与 standalone 的选择内核（v1.6，#14570）

把 frontend 内嵌 router 与 standalone selection 的核心合并到 `lib/kv-router/src/services/selection/core` 和 `services/indexer/{backend,lookup}.rs`（单次改动约 1 万行），session affinity 也移到 `services/selection/affinity.rs`。

### 5.4 Rust EPP（Gateway API Inference Extension）（v1.3 #8783 → v1.4 standalone 模式 → v1.5 Go 版下线）

- Envoy 通过 ext_proc 调用 Rust 实现的 EPP（Endpoint Picker）做 KV 感知选择，调用链从 Envoy → Go → Python 三层语言缩减为 Envoy → Rust 一层。
- 两种模式：
  - runtime 模式：嵌入完整 Dynamo KV router，P/D 角色从服务发现自动判断；
  - `DYN_EPP_MODE=standalone`：内嵌 `SelectionService`，从 GAIE 的 `InferencePool` 发现 worker，`DYN_EPP_PEER_SERVICE` 做多 EPP 副本同步。
- 多种 tokenizer 提供方（vLLM `/render`、SGLang tokenizer）、优雅退出（SIGTERM 时先报 NOT_SERVING）、缓存命中 token 指标。
- 1.5.0 起 Rust EPP 二进制随 frontend 镜像发布（`/epp`），**Go EPP 镜像不再发布**。

### 5.5 其他对外 API

- overlap 分数 API（v1.3，#9538）：按 worker 返回分层 KV 命中。
- standalone router 把 `prefill_time_ms` / `ttft_ms` / `kv_hit_rate` 等时序回传 frontend（v1.3，#10182）。
- potential loads API；`markPrefill` / `markFree` 不再强制要求 context id（v1.3，#10412）。

**friday 分支现状**：有 standalone indexer 和 `deploy/inference-gateway/epp`，但该 EPP 是 **Go 版**（`main.go`、`dynamo_kv_scorer/plugin.go`），即上游已停止发布的版本。selection service、slot tracker、统一选择内核都没有。

---

## 六、多模态 KV 路由（mm-routing）

- **原理**：frontend 为每个图片/视频计算稳定 identity，在"仅用于路由的 token 视图"中用它占位，并对齐后端的缓存 identity 与 token 布局；router 把多模态 overlap 纳入打分。选中的 worker 收到同一个 hash，其 KV 事件也使用该 identity。
- **图片**：默认对 URL 字节做 hash（同图换 URL 视为不同图）；开启 `--frontend-decoding` 后对解码后内容做 hash。
- **视频**：必须 frontend 解码。对采样帧 + 元数据计算 XXH3-64，由模型适配器复现 worker 的 resize / patch / 时间分组布局。worker 启动时上报 processor 契约，与 frontend 不一致则关闭精确视频路由。
- **覆盖模型**：Qwen2-VL / 2.5-VL / 3.5 / 3.6、Kimi K2 / K3、Nemotron 3 Nano Omni（图像 + 视频）、Qwen 视频（vLLM / SGLang）。
- **其他**：TRT-LLM 多模态路由迁到 Rust frontend 原生实现，去掉 Python sidecar（v1.4，#11737）；`/inference/v1/generate` 支持精确多模态路由（v1.5，#11588）。

**friday 分支现状**：没有。friday 的 `lightseek_mm.rs` 是多模态预处理，不涉及路由侧多模态 identity。

---

## 七、KV 事件来源扩展与多租户隔离

### 7.1 cache_salt 多租户隔离 + routing group（v1.4，#8197 / #11471）

- 租户标识来自 `x-tenant-id`、`nvext.cache_salt` 或请求体 `cache_salt` 字段。带盐请求在 router 与 vLLM / TRT-LLM KV 事件中独立计算 hash，从根本上杜绝跨租户缓存复用。
- `tenant_id` 更名为 `routing_group`：indexer 按 `(model_name, routing_group)` 各自维护一棵 radix tree，相互隔离。

### 7.2 Provider-keyed tracking hash（v1.4，#11548）

- `--router-tracking-hash keyed-xxh3-v1`，需要 32 字节密钥文件与 key id。
- 用 BLAKE3 从（算法版本、key id、模型、routing group、block size、cache_salt、LoRA、Eagle）派生 XXH3 种子，只用于活跃序列追踪；主索引仍用公开 hash。
- 目的：防止外部伪造 hash 污染负载记账。轮换密钥需所有生产者同时重启。

### 7.3 下层缓存与共享缓存事件

| 来源 | 版本 / PR | 说明 |
|---|---|---|
| vLLM CPU offload 事件 → HostPinned 层 | v1.3，#10368 | 统计下层命中 |
| vLLM STORAGE 事件 → Disk 层 | v1.5，#11571 | 带 locality gating，不确定时 fail-closed |
| Mooncake 共享缓存 | v1.5，#11239 | 为 SGLang HiCache 建立事件驱动的 Mooncake 共享索引 |
| 原生 offload 容量 | v1.4，#11321 | `native_offloading_capacity.total_tokens`，避免 router 预算重复计算 GPU 容量 |

### 7.4 KV DC Relay（跨数据中心）（v1.4 #11793 / #12199，v1.5 #13471 / #13539）

- 每个数据中心把每个 KV pool 压缩为 Cuckoo filter（CKF）投影对外发布，**不在 WAN 上复制每个 worker 的完整事件流**；精确的 block 归属只保留在本地。
- 另外发布服务拓扑（按 namespace + 模型给出 READY / UNAVAILABLE / UNKNOWN）和 pool 负载。
- 通过 gRPC/Protobuf 流式提供给消费方。Relay 不做路由决策，"去哪个数据中心"由消费方决定。
- 独立组件：`components/src/dynamo/kv_dc_relay`。

### 7.5 KV state-agent（v1.5，#13110 / #13237）

- 把 vLLM 的 KV 事件状态从引擎进程剥离到可外置、长期存活的 agent。
- 每个 agent 绑定一个稳定 DP slot，内含本地恢复索引、语义事件发布器、出站游标。引擎每次启动实例（incarnation）可 attach / detach；独立 host（`state_agent_host.rs`）可在一个进程内复用多个 slot。
- 目标：KV 索引身份不受引擎重启影响。源码注释写明跨 incarnation 的连续性尚未实现（TODO #13044）。
- 配套组件：`components/src/dynamo/kv_state_agent`。

### 7.6 远程 KV 复用提示 router_hint（v1.5；v1.6 #14890）

- router 发现另一个非目标 worker 上有更好的前缀时，在发给后端的请求里附带 `router_hint`，供 KVCR（KV Cache Runner）等远程拉取组件从对端拉取 KV block。
- v1.6 把它转换为 vLLM 原生 `KvHintsEnvelope`；PD 分离下只发给 prefill worker。
- standalone selection 中，worker 需注册 `router_hint_worker_type` / `router_hint_source_control_endpoints`。

### 7.7 worker 上报的实际复用量（v1.6，#15443）

- 新增三组指标（标签 phase / model）：
  - 选择时最佳候选 worker 的缓存前缀 token 数；
  - 选择时实际选中 worker 的缓存前缀 token 数；
  - 后端实际上报的复用 token 数。
- vLLM / SGLang / TRT-LLM 均上报，用于**校准 router 预估与真实复用的偏差**。

**friday 分支现状**：
- cache_salt 只零星出现在 vLLM handler 与 `zmq_wire/extra_keys.rs`，router 侧没有 routing group 隔离；
- 只有 host 层权重（#24，`host_cache_hit_weight`）；
- 7.2 以及 7.4～7.7 均没有。

---

## 汇总：迁移价值排序（建议）

| 优先级 | 特性 | 理由 |
|---|---|---|
| 高 | 插件化（1.1～1.4） | friday 的定制路由（length-aware、P-first、负载衰减）可改写为 scorer / classifier，此后跟随上游时冲突大幅减少 |
| 高 | Conditional disagg（3.6）、predict-on-route（3.5） | 直接针对 PD 与 agent 场景的 TTFT 和缓存复用，与 friday 的 PD 路由方向一致 |
| 高 | Session affinity hard/soft + 子 agent 分组（第二节） | agent 类负载收益明显；friday 的 StickySessionRouter 已落后 |
| 中 | Deadline / 严格优先级（3.2、3.3） | 在已迁移的 policy queue 上补齐，成本低 |
| 中 | Taints / 拓扑约束（第四节） | 可替代 worker-tag 配对这类内部实现 |
| 中 | 复用遥测（7.7） | 用于验证 friday 自研 credit 衰减的实际效果 |
| 低（视场景） | EPP / standalone selection / DC Relay / state-agent / 多模态路由 | 取决于是否接入 Gateway、是否跨机房、是否有多模态流量 |
