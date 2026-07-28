# Dynamo 代码地图

> NVIDIA Dynamo —— 开源、面向数据中心规模的 LLM 推理编排栈。
> Rust 内核（性能）+ Python 上层（可扩展性）。Workspace 版本：`1.2.0`。
>
> 代码体量（仅统计 `*.rs / *.py / *.go / *.cu / *.cuh / *.c / *.h`，已剔除 `target/.venv/__pycache__/build/dist`）：
> **lib ≈ 427K · components ≈ 109K · deploy ≈ 108K · tests ≈ 64K · benchmarks ≈ 13K · examples ≈ 2.4K · container ≈ 1.7K · docs ≈ 1.3K · fern ≈ 0.5K**。

Dynamo 是位于推理引擎（SGLang / TensorRT-LLM / vLLM）**之上**的编排层，提供：
prefill/decode 解耦、KV 感知路由、多层级 KV 缓存（KVBM）、SLA 驱动的弹性伸缩（Planner）、
极速冷启动（ModelExpress）、Kubernetes gang-scheduling（Grove）。

---

## 顶层目录结构

```
dynamo/
├── lib/            # Rust workspace —— runtime、LLM engine、KV router、KVBM、bindings
├── components/     # Python 服务 —— frontend、backends、planner、router、profiler
├── deploy/         # K8s operator（Go）、Helm charts、Docker compose、observability
├── container/      # runtime/test 镜像所用的 Dockerfile 与构建脚本
├── recipes/        # 按模型组织的 Kubernetes 生产部署 recipe
├── examples/       # 参考部署与自定义后端示例
├── benchmarks/     # 压测生成器与基准测试套件
├── tests/          # 跨模块的集成测试 / e2e 测试 / 故障容错测试
├── docs/           # 架构文档、设计文档、用户指南
├── fern/           # 基于 Fern 的文档站源码
├── Cargo.toml      # Rust workspace 清单（resolver = 3，edition 2024）
└── pyproject.toml  # `ai-dynamo` Python 包（extras：vllm | sglang | trtllm | mocker）
```

---

## `lib/` —— Rust Workspace

性能关键的核心。Workspace 成员在 `Cargo.toml` 中声明。

| Crate | 行数 | 角色 |
|------|-----:|------|
| `lib/llm` | 152,905 | 面向 LLM 的编排：HTTP/gRPC frontend、OpenAI 兼容 API、KV router、preprocessor、model card、agents、audit、LoRA、migration、mocker 集成、telemetry。Crate：`dynamo-llm`。 |
| `lib/runtime` | 57,178 | 基础分布式 runtime：components、endpoints、pipelines、transports（NATS/etcd/TCP）、metrics、health、system status server。Crate：`dynamo-runtime`。 |
| `lib/kv-router` | 35,173 | KV 感知请求路由：prefix tree、cost model、indexer、基于 NATS 协调的 cache 事件。Crate：`dynamo-kv-router`。 |
| `lib/bindings/{python,c,kvbm}` | 33,519 | PyO3 / C ABI / KVBM 多语言绑定（Python 入口：`maturin develop`）。 |
| `lib/mocker` | 30,090 | 用于测试/重放的推理引擎 mocker（vLLM/SGLang scheduler 模拟）。 |
| `lib/kvbm-engine` | 27,166 | KVBM engine 胶水：leader/worker、velo session、对象存储、pubsub、collectives。 |
| `lib/kvbm-logical` | 20,594 | KVBM 逻辑层：pools、blocks、events、manager、metrics、registry、sequence assignments。 |
| `lib/parsers` | 15,633 | Tool-call / chat-template / reasoning 解析器。Crate：`dynamo-parsers`。 |
| `lib/kvbm-physical` | 13,203 | KVBM 物理层：layout、manager、transfer executor、notifications。 |
| `lib/gpu_memory_service` | 9,617 | 可作为 Python 包安装的 GPU 显存核算服务（cli/client/server/snapshot/integrations）。 |
| `lib/bench` | 7,973 | 关键热路径的微基准（含 coding、kv_router 等）。 |
| `lib/memory` | 7,191 | KVBM 与 runtime 使用的内存抽象（nixl/numa/pool；二进制入口在 `lib/memory/bin`）。 |
| `lib/kvbm-kernels` | 3,369 | KVBM 用 CUDA kernel。 |
| `lib/protocols` | 3,260 | 跨组件的线缆协议（请求/响应、KV 事件）。Crate：`dynamo-protocols`。 |
| `lib/backend-common` | 3,216 | 后端集成共享的 trait/工具；附带 mocker 示例。 |
| `lib/tokens` | 2,855 | Token 工具与数据结构。 |
| `lib/kvbm-config` | 2,399 | KVBM 配置类型。 |
| `lib/tokenizers` | 1,783 | Tokenizer 封装。 |
| `lib/config` | 202 | 共享的配置类型。 |
| `lib/kvbm-common` | 28 | KVBM 公共定义（极小，仅常量/类型别名）。 |

### `lib/llm/src` 关键模块
`http/`、`grpc/`、`kv_router/`、`preprocessor/`、`block_manager/`、`discovery/`、
`entrypoint/`、`agents/`、`audit/`、`lora/`、`protocols/`、`telemetry/`，
以及 `migration.rs`、`model_card.rs`、`mocker.rs`、`recorder.rs`、`engines.rs`。

### `lib/runtime/src` 关键模块
`component/`、`pipeline/`、`transports/`、`discovery/`、`metrics/`、`protocols/`、
`storage/`、`compute/`，以及 `distributed.rs`、`worker.rs`、`service.rs`、
`system_health.rs`、`system_status_server.rs`。

---

## `components/` —— Python 服务

通过 `components/src/dynamo/` 安装到 `dynamo.*` 命名空间下。
每个子包通常都会带 `__main__.py`，因此可以通过 `python3 -m dynamo.<svc>` 运行。

| Package | 行数 | 用途 |
|---------|-----:|------|
| `dynamo.vllm` | 21,534 | vLLM 后端 worker：`llm_engine.py`、`handlers.py`、`instrumented_scheduler.py`，以及多模态 handler/工具。 |
| `dynamo.planner` | 19,607 | SLA 驱动的弹性伸缩 —— `core/`、`connectors/`（k8s 等）、`monitoring/`、`offline/`，以及 profiling 结果 fixture。 |
| `dynamo.profiler` | 16,075 | Workload profiling —— `rapid.py`、`thorough.py`、`profile_sla.py`、`interpolation.py`，并附带 Web UI。 |
| `dynamo.common` | 14,396 | 跨后端共享代码：`backend/`、`protocols/`、`http/`、`lora/`、`memory/`、`multimodal/`、`forward_pass_metrics.py`。 |
| `dynamo.trtllm` | 13,469 | TensorRT-LLM 后端 worker：`engine.py`、`llm_engine.py`、`workers/`、`logits_processing/`、`multimodal/`。 |
| `dynamo.sglang` | 10,986 | SGLang 后端 worker：`unified_main.py`、`init_{llm,diffusion,embedding,multimodal}.py`、`request_handlers/`。 |
| `dynamo.frontend` | 7,408 | OpenAI 兼容的 HTTP frontend；面向 vLLM/SGLang 的预处理辅助。入口：`python3 -m dynamo.frontend`。 |
| `dynamo.global_router` | 2,036 | 跨 pool / 跨集群路由层。 |
| `dynamo.mocker` | 1,597 | 引擎 mocker 集成（与 `lib/mocker` 配套）。 |
| `dynamo.global_planner` | 998 | 集群级伸缩处理器（与 operator 协作）。 |
| `dynamo.tokenspeed` | 704 | Token 吞吐量测量工具。 |
| `dynamo.router` | 566 | 独立的 KV 感知 router 服务（薄封装，逻辑在 `lib/kv-router`）。 |

各后端（`vllm`、`sglang`、`trtllm`）遵循相同的模式：
`args.py` + `backend_args.py`（CLI/配置）→ `main.py` / `unified_main.py`（入口）→
`llm_engine.py`（engine 包装）→ `request_handlers/` → `health_check.py` + `publisher.py`。

---

## `deploy/` —— Kubernetes 与运维

| 路径 | 行数 | 内容 |
|------|-----:|------|
| `deploy/operator/` | 90,820 | **Go** 编写的 Kubernetes operator（Kubebuilder 布局：`api/`、`cmd/`、`internal/`、`config/`、`samples/`）。负责 reconcile `DynamoGraphDeployment` / `DynamoGraphDeploymentRequest` 两个 CRD。本地开发使用 Tiltfile。其中 `zz_generated*` 约占 ~3.6K，业务代码约 86K。 |
| `deploy/snapshot/` | 10,417 | Snapshot agent / `snapshotctl` CLI（Go），含 CUDA/DRA、runtime、kube 适配。 |
| `deploy/inference-gateway/` | 1,400 | K8s Inference Gateway 插件（在标准 gateway 内部实现 KV 感知路由）。 |
| `deploy/utils/` | 994 | 部署辅助脚本与工具。 |
| `deploy/observability/` | 523 | Grafana/Prometheus 仪表盘与配置。 |
| `deploy/helm/charts/` | — | 用于安装平台的 Helm charts（仅 YAML，未计入源码统计）。 |
| `deploy/docker-compose.yml` | — | 本地开发栈（etcd + NATS）。 |
| `deploy/pre-deployment/` | — | 预检脚本。 |
| `deploy/sanity_check.py` | — | 部署后的冒烟检查。 |

---

## `container/` —— 构建与运行时镜像

`Dockerfile.template`（多后端通用）、`Dockerfile.docs`、`Dockerfile.test`、
`build_trtllm_wheel.sh`、`run.sh`、`render.py`（基于 Jinja 的 Dockerfile 渲染器），
以及 `deps/`（各后端依赖安装器）、`templates/`、`compliance/`。

---

## `recipes/` —— 按模型的生产部署

为以下模型提供 K8s manifest 与调优说明：`deepseek-r1`、`deepseek-v32-fp4`、`deepseek-v4`、
`gpt-oss-120b`、`kimi-k2.5`、`llama-3-70b`、`nemotron-3-{nano-omni,super-fp8}`、
`glm-5-nvfp4`、`qwen3-{32b,32b-fp8,235b-a22b-fp8,vl-30b}`。每个 recipe 面向 vLLM / SGLang /
TensorRT-LLM 中的一个或多个，并支持聚合（aggregated）或解耦（disaggregated）拓扑。

---

## `examples/` —— 参考部署

`backends/`（vLLM/SGLang/TRT-LLM 示例配置）、`deployments/`（EKS、GKE）、
`custom_backend/`（新后端模板）、`diffusers/`、`global_planner/`、
`chat_templates/`、`common/`。

---

## `benchmarks/` —— 压测与性能分析

`agent_trace/`、`burstgpt_loadgen/`、`frontend/`、`incluster/`、`llm/`、`multimodal/`、
`nat_trace/`、`omni/`、`prefix_data_generator/`、`router/`、`sin_load_generator/`。
独立的 `pyproject.toml`。

---

## `tests/` —— 跨模块测试

`basic/`、`benchmarks/`、`deploy/`、`dgdr/`、`fault_tolerance/`（含硬件故障注入服务）、
`frontend/`、`gpu_memory_service/`、`kvbm_integration/`、`lmcache/`、
`mm_router/`、`parity/`、`router/`、`serve/`、`vllm_self_benchmark/`。
pytest 配置位于根目录 `pyproject.toml` 的 `[tool.pytest.ini_options]` 中——
为 GPU / 并行度 / 后端 / 发布流水线定义了大量 marker。

---

## `docs/` —— 文档

架构与设计（`design-docs/`、`proposals/`、`digest/`）、按组件分类的指南
（`components/`、`backends/`、`agents/`、`mocker/`）、运维相关
（`kubernetes/`、`observability/`、`fault-tolerance/`、`performance/`）、参考
（`api/`、`reference/`），以及 `getting-started/`、`development/`、`integrations/`、
`templates/`、`assets/`。`fern/` 存放发布版文档站使用的源文件。

---

## 服务发现与消息通信

- 组件间传输：**TCP**（请求平面）。
- 服务发现可选：基于文件（本地开发）、Kubernetes 原生（CRD + EndpointSlice）、`etcd`。
- KV 感知路由的前缀协调需要 **NATS**（JetStream）。

---

## 构建与入口

| 操作 | 命令 |
|------|------|
| 构建 Rust 并以可编辑模式安装 Python | 在 `lib/bindings/python` 目录执行 `maturin develop --uv`，随后 `uv pip install -e .` |
| 生成 frontend OpenAPI spec | `cargo run -p dynamo-llm --bin generate-frontend-openapi` |
| 启动 frontend | `python3 -m dynamo.frontend --http-port 8000` |
| 启动一个后端 worker | `python3 -m dynamo.{vllm,sglang,trtllm} --model-path …` |
| 本地开发基础设施（etcd + NATS） | `docker compose -f deploy/docker-compose.yml up -d` |

---

## 高层数据流

```
HTTP/gRPC 客户端
   │
   ▼
dynamo.frontend  ──►  KV 感知 router（lib/kv-router）
   │                       │
   │                       ├─► 前缀索引（基于 NATS 协调的 KV 事件）
   │                       └─► worker 选择（综合负载与 cache 命中重叠度）
   ▼
后端 worker（dynamo.vllm | dynamo.sglang | dynamo.trtllm）
   │
   ├─► engine（vLLM / SGLang / TRT-LLM）
   ├─► KVBM（lib/kvbm-*） —— GPU↔CPU↔SSD↔remote KV 多层
   └─► publisher → 将 metrics / KV 事件回传 router

Planner（dynamo.planner） ◄─ metrics ─┘
   │
   └─► 通过 K8s operator（deploy/operator）伸缩 prefill/decode 池

