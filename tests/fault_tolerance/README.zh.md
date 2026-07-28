# 容错（Fault Tolerance）测试

## 迁移测试（Migration Tests）

migration 目录包含 worker 容错（带迁移支持）测试，覆盖多个后端（vLLM、SGLang、TRT-LLM）的聚合（aggregated）与解耦（disaggregated）模式。

### 测试参数化

所有迁移测试均按以下维度参数化：

| 参数 ID | 说明 |
|---------------|-------------|
| `migration_enabled`、`migration_disabled` | 控制是否允许迁移 |
| `worker_failure`（SIGKILL）、`graceful_shutdown`（SIGTERM） | worker 终止方式 |
| `chat`、`completion`（已跳过） | 待测试的 API 端点 |
| `stream`、`unary`（已跳过） | 流式响应 vs 单次响应 |
| `nats`、`tcp` | 请求面（request plane）传输 |

### 测试矩阵

每个后端（vLLM、SGLang、TRT-LLM）都包含以下测试类型：

| 测试 | 模式 | 配置 |
|------|------|-------|
| `test_request_migration_{backend}_aggregated` | 聚合 | 2 worker |
| `test_request_migration_{backend}_prefill` | 解耦 | 1 个 decode + 2 个 prefill |
| `test_request_migration_{backend}_kv_transfer` | 解耦 | 1 个 prefill + 2 个 decode |
| `test_request_migration_{backend}_decode` | 解耦 | 1 个 prefill + 2 个 decode |

其中 `{backend}` 取值为：`vllm`、`sglang`、`trtllm`

### 通用测试流程

1. 启动 Dynamo 前端（frontend），使用轮询路由（round-robin routing）
2. 启动 worker（按聚合或解耦模式不同而不同）
3. 在后台线程中发送请求（chat/completion，stream/unary）
4. 通过日志轮询确定哪个 worker 接收了该请求
5. 解码（decode）测试：在终止前先等待初始响应
6. 终止处理该请求的 worker（SIGKILL 或 SIGTERM）
7. 基于 `migration_limit` 校验请求结果：
   - `migration_limit > 0`：请求成功；若为流式则校验 TTFT/TPOT 与迁移指标（metrics）
   - `migration_limit = 0`：请求按预期失败
8. 在前端日志中校验迁移行为

**运行示例：**
```bash
# Run all vLLM migration tests
pytest tests/fault_tolerance/migration -m vllm -v -s

# Run aggregated or decode tests for SGLang
pytest tests/fault_tolerance/migration -m sglang -k "aggregated or decode" -v -s

# Run specific parameter combination
pytest tests/fault_tolerance/migration -m trtllm -k "aggregated and nats and stream and chat and worker_failure and migration_enabled" -v -s
```

## 取消（Cancellation）测试

cancellation 目录包含跨多种 API 端点、后端与部署配置的请求取消功能测试。

### 各后端测试概览

#### vLLM 取消测试

| 测试 | 模式 | 取消阶段 | 请求类型 | 配置 |
|------|------|-------------------|--------------|-------|
| `test_request_cancellation_vllm_aggregated` | 聚合 | 生成过程中 | 3 种场景：completion、chat、流式 chat | 1 worker |
| `test_request_cancellation_vllm_decode_cancel` | 解耦 | 远端 decode | 流式 chat（读取 5 条响应） | Prefill + Decode worker |
| `test_request_cancellation_vllm_remote_prefill_cancel` | 解耦 | 远端 prefill | Completion（长 prompt） | Prefill + Decode worker |

**运行示例：**
```bash
pytest tests/fault_tolerance/cancellation/test_vllm.py::test_request_cancellation_vllm_aggregated -v -s
pytest tests/fault_tolerance/cancellation/test_vllm.py::test_request_cancellation_vllm_decode_cancel -v -s
pytest tests/fault_tolerance/cancellation/test_vllm.py::test_request_cancellation_vllm_remote_prefill_cancel -v -s
```

#### TRT-LLM 取消测试

| 测试 | 模式 | 取消阶段 | 请求类型 | 配置 |
|------|------|--------------------|--------------|-------|
| `test_request_cancellation_trtllm_aggregated` | 聚合 | 生成过程中 | 3 种场景：completion、chat、流式 chat | 1 worker（prefill_and_decode） |
| `test_request_cancellation_trtllm_disagg_decode_cancel` | 解耦 | 远端 decode | 流式 chat（读取 5 条响应） | Prefill + Decode worker |
| `test_request_cancellation_trtllm_disagg_prefill_cancel` | 解耦 | 远端 prefill | Completion（长 prompt） | Prefill + Decode worker |

**运行示例：**
```bash
pytest tests/fault_tolerance/cancellation/test_trtllm.py::test_request_cancellation_trtllm_aggregated -v -s
pytest tests/fault_tolerance/cancellation/test_trtllm.py::test_request_cancellation_trtllm_disagg_decode_cancel -v -s
pytest tests/fault_tolerance/cancellation/test_trtllm.py::test_request_cancellation_trtllm_disagg_prefill_cancel -v -s
```

#### SGLang 取消测试

| 测试 | 模式 | 取消阶段 | 请求类型 | 配置 | 备注 |
|------|------|-------------------|--------------|-------|-------|
| `test_request_cancellation_sglang_aggregated` | 聚合 | 生成过程中 | 3 种场景：completion、chat、流式 chat（读取 1 条响应） | 1 worker | ⚠️ 不稳定：SGLang prefill 取消存在问题 |
| `test_request_cancellation_sglang_decode_cancel` | 解耦 | 远端 decode | 流式 chat（读取 1 条响应） | Decode + Prefill worker | 需要 2 个 GPU |

**运行示例：**
```bash
pytest tests/fault_tolerance/cancellation/test_sglang.py::test_request_cancellation_sglang_aggregated -v -s
pytest tests/fault_tolerance/cancellation/test_sglang.py::test_request_cancellation_sglang_decode_cancel -v -s
```

### 通用取消测试模式

1. 启动前端与 worker（具体配置因测试而异）
2. 发送请求（请求类型按测试场景而异）
3. 在 worker 日志中轮询请求 ID
4. 流式：在取消前读取 N 条响应
5. 通过 API 取消该请求
6. 在 worker 与前端日志中校验取消相关消息

**校验模式：**
- 聚合模式：worker 日志出现 "Aborted Request ID"
- 解耦 - prefill 取消：prefill worker 出现 "Aborted Request ID"（在 prefill 阶段取消）
- 解耦 - decode 取消：decode worker 出现 "Aborted Request ID"（在 decode 阶段取消）
