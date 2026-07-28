# DGDR v1beta1 端到端测试套件

本目录包含 **DynamoGraphDeploymentRequest (DGDR) v1beta1** 的端到端测试套件。DGDR v1beta1 是 Dynamo 中用于部署推理（inference）模型的、面向 SLA 的高层 Kubernetes API。

## 测试覆盖范围

| 测试组 | Marker | GPU 需求 | 是否支持 mocker？ | 覆盖内容 |
|---|---|---|---|---|
| `TestDGDRValidation` | `gpu_0`、`pre_merge` | 无 | ✅ | Webhook 校验：被拒绝/接受的 spec、取值约束、storage version、shortname |
| `TestDGDRVersionConversion` | `gpu_0`、`pre_merge` | 无 | ✅ | v1alpha1 → v1beta1 conversion webhook |
| `TestDGDRMinimalDeployment` | `gpu_1`、`pre_merge`、`e2e` | 1+ | ⚠️ 见说明 | 完整的 Pending → Profiling → Ready → Deploying → Deployed 生命周期 |
| `TestDGDRBackendSelection` | `gpu_1`、`nightly`、`e2e` | 1+ | ⚠️ 仅 vllm+trtllm | vllm 与 trtllm 通过；sglang **被跳过**（mocker 默认 GPU SKU 缺少 sglang 的 AIC silicon 数据） |
| `TestDGDRSearchStrategies` | `gpu_1`/`gpu_8`、`e2e` | 1 或 8 | ⚠️ 仅 rapid | `rapid` 走 AIC，可正常工作；`thorough` 需要真正的 GPU 扫描 |
| `TestDGDRSLATargets` | `gpu_1`、`nightly`、`e2e` | 1+ | ✅ | ttft+itl、e2eLatency、optimizationType（latency/throughput） |
| `TestDGDRWorkloadPickingModes` | `gpu_1`、`nightly`、`e2e` | 1+ | ✅ | requestRate、concurrency、isl/osl |
| `TestDGDRFeatures` | `gpu_1`、`nightly`、`e2e` | 1+ | ⚠️ 见说明 | planner（rapid/none sweep）、mocker |
| `TestDGDRModelCache` | `gpu_1`、`nightly`、`e2e` | 1+ | ✅ | 基于 PVC 的模型缓存，缓存信息透传到 DGD |
| `TestDGDRHardwareOverride` | `gpu_1`、`pre_merge`、`e2e` | ✅ | ✅ | 手工指定 gpuSku/numGpusPerNode/totalGpus/vramMb |
| `TestDGDRAutoApply` | `gpu_1`、`pre_merge`、`e2e` | 1+ | ⚠️ 见说明 | mocker 中 autoApply=true **被跳过**（operator race）；autoApply=false 保持 Ready |
| `TestDGDROverrides` | `gpu_1`、`nightly`、`e2e` | 1+ | ✅ | Profiling job 的 toleration 注入；DGD 元数据 label 合并 **xfail**（operator 缺陷） |
| `TestDGDRStatusAndConditions` | `gpu_1`、`pre_merge`、`e2e` | 1+ | ⚠️ 见说明 | 所有 condition 设置正确、子 phase 跟踪、Pareto 配置；mocker 下 all-conditions **xfail**；mocker 下 pareto **被跳过** |
| `TestDGDRImmutability` | 混合 | 0–1 | ⚠️ 见说明 | 在 Profiling/Deployed 阶段 spec 被拒绝，metadata 始终允许 |
| `TestDGDRCleanup` | `gpu_1`、`pre_merge`、`e2e` | 1+ | ⚠️ 见说明 | DGDR 删除时 Job 被清理；DGD 被保留；ConfigMap 清理 **xfail**（operator 缺陷）；mocker 下 DGD-persistence 测试 **被跳过** |
| `TestDGDRMoEModels` | `gpu_8`、`nightly`、`e2e` | 8 | ❌ | DeepSeek-R1 MoE 跑在 SGLang 上 —— 需要真实的 8-GPU 节点 |

## 前置条件

1. 一个带 GPU 节点的 Kubernetes 集群（cluster），或参见下方的 [GPU-free 模式](#gpu-free-mocker-mode)
2. 已安装 Dynamo operator（含 CRD 与 webhook）
3. 已配置 `kubectl` 并指向该集群
4. Python 3.10+ 并安装 `pytest` 与 `pyyaml`：
   ```bash
   pip install pytest pyyaml
   # or, from the repo root:
   pip install -e ".[test]"
   ```

## 一次性集群准备

在运行任何测试之前，请确保集群中具备以下条件。即便使用 GPU-free（mocker）模式也是必需的。

### 1. 安装 Dynamo operator

```bash
cd deploy/operator
helm install dynamo-operator helm/dynamo-operator -n dynamo-system --create-namespace
```

### 2. 部署 NATS

Mocker worker（以及真实 worker）通过 NATS 进行组件间通信。
operator 期望 NATS 位于 `nats://dynamo-operator-nats.dynamo-system.svc.cluster.local:4222`。

```bash
helm repo add nats https://nats-io.github.io/k8s/helm/charts/
helm repo update
helm install dynamo-operator-nats nats/nats -n dynamo-system --create-namespace
```

### 3. 创建 HuggingFace token Secret

Profiling job 从名为 `hf-token-secret` 的 Secret 中读取 HF token，键名为 `HF_TOKEN`（不是 `HUGGING_FACE_HUB_TOKEN`）。

```bash
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=<your-hf-token> \
  -n default
# If running in a non-default namespace, adjust -n accordingly
```

> **重要：** 键名必须为 `HF_TOKEN`，Secret 名必须为 `hf-token-secret`。
> 用其他键名会让 profiling job 静默失败。

## 运行测试

根据是否拥有 GPU 硬件，主要有两种方式运行测试套件。

---

### GPU-free（mocker 模式）—— 推荐用于本地开发与 CI

无需 GPU 节点。使用 AIC 模拟做 profiling，用 mock 推理 worker 做部署。覆盖所有 `gpu_0` 与 `gpu_1` 的测试（约 45 个）；`gpu_8` 测试即便在 mocker 模式下也需要真实的 8-GPU 节点，因此被排除。

```bash
python3 -m pytest tests/dgdr/ -m "gpu_0 or gpu_1" -v \
  --dgdr-namespace=default \
  --dgdr-image=<your-image>
```

预期结果：37 通过，6 跳过（2 个 model-cache PVC；sglang 后端；mocker 下的 pareto；mocker 下的 DGD-persistence；mocker 下的 auto-apply-true），4 个 xfail（DGD label 合并；all-conditions 需要 Deployed 阶段；dry-run 不可变性需要 Deployed 阶段；删除时的 ConfigMap 清理）。
`test_backend[sglang]` 是上述 6 个跳过用例之一（mocker 模式下没有 sglang 的 AIC silicon 数据）。

---

### 在真实 GPU 上运行完整套件 —— 用于生产/夜间验证

需要带 GPU 节点的 Kubernetes 集群。设置 `--dgdr-no-mocker` 关闭 mocker 注入并跑在真实硬件上。`gpu_8` 测试还需要一个 8-GPU 节点。

```bash
# gpu_0 + gpu_1 tests on real GPUs (single-GPU node sufficient)
python3 -m pytest tests/dgdr/ -m "gpu_0 or gpu_1" -v \
  --dgdr-namespace=dynamo-test \
  --dgdr-image=<your-image> \
  --dgdr-no-mocker \
  --dgdr-profiling-timeout=3600 \
  --dgdr-deploy-timeout=1800

# Full nightly suite including 8-GPU tests
python3 -m pytest tests/dgdr/ -v \
  --dgdr-namespace=dynamo-test \
  --dgdr-image=<your-image> \
  --dgdr-no-mocker \
  --dgdr-pvc-name=model-cache \
  --dgdr-profiling-timeout=14400 \
  --dgdr-deploy-timeout=3600
```

预期结果（gpu_0 + gpu_1，配合 `--dgdr-pvc-name`）：**约 43 通过，0 跳过，2 个 xfail**（DGD label 合并 operator 缺陷；ConfigMap 清理 operator 缺陷）。
不带 `--dgdr-pvc-name` 时，会再多 2 个跳过（model-cache 测试）。

> **说明：** 有两个 xfail 是 mocker 与 GPU 模式下都存在的**永久 operator 缺陷**：
> - `test_dgd_override_injects_custom_labels` —— operator 暂未把 `spec.overrides.dgd.metadata.labels` 合并到所创建的 DGD 上。
> - `test_deletion_removes_output_configmap` —— operator 的 `FinalizeResource` 是空操作，不会在 DGDR 删除时清理输出的 ConfigMap。
> 其他 mocker 模式下的 xfail/skip 在 GPU 模式下都会消失，预期都能通过。

---

### 其他常用调用方式

```bash
# Validation + conversion tests only (no cluster setup required beyond CRDs)
python3 -m pytest tests/dgdr/ -m "gpu_0" -v \
  --dgdr-namespace=default \
  --dgdr-image=<your-image>

# Pre-merge gate (GPU-free)
python3 -m pytest tests/dgdr/ -m "pre_merge" -v \
  --dgdr-namespace=default \
  --dgdr-image=<your-image>

# Single test class
python3 -m pytest tests/dgdr/test_dgdr_v1beta1.py::TestDGDRAutoApply -v \
  --dgdr-namespace=default \
  --dgdr-image=<your-image>
```

## CLI 选项

| 选项 | 默认值 | 描述 |
|---|---|---|
| `--dgdr-namespace` | _(必填)_ | 测试资源所在的 Kubernetes 命名空间 |
| `--dgdr-image` | _(必填)_ | profiling 与推理 worker 使用的容器镜像 |
| `--dgdr-model` | `Qwen/Qwen3-0.6B` | 大多数测试使用的 HuggingFace 模型 ID |
| `--dgdr-backend` | `vllm` | DGDR 测试默认使用的后端（backend） |
| `--dgdr-pvc-name` | _(空)_ | 已预下载模型权重的 PVC 名称（未设置时 PVC 测试会被跳过） |
| `--dgdr-profiling-timeout` | `3600` | 等待 profiling 完成的秒数 |
| `--dgdr-deploy-timeout` | `600` | 等待 DGD 进入 Deployed 阶段的秒数 |
| `--dgdr-no-mocker` | `false` | 关闭 mocker 模式（要求有真实 GPU 节点） |

## DGDR v1beta1 功能覆盖矩阵

下列 spec 字段至少被一个测试覆盖：

| 字段 | 覆盖该字段的测试 |
|---|---|
| `spec.model` | 所有测试 |
| `spec.backend`（auto/vllm/sglang/trtllm） | `TestDGDRBackendSelection`、`TestDGDRValidation` |
| `spec.image` | 所有测试 |
| `spec.searchStrategy`（rapid/thorough） | `TestDGDRSearchStrategies` |
| `spec.sla.ttft` + `spec.sla.itl` | `TestDGDRSLATargets::test_sla_ttft_and_itl` |
| `spec.sla.e2eLatency` | `TestDGDRSLATargets::test_sla_e2e_latency` |
| `spec.sla.optimizationType` | `TestDGDRSLATargets::test_sla_optimization_type_*` |
| `spec.workload.isl` + `spec.workload.osl` | `TestDGDRWorkloadPickingModes` |
| `spec.workload.requestRate` | `TestDGDRWorkloadPickingModes::test_request_rate_picking` |
| `spec.workload.concurrency` | `TestDGDRWorkloadPickingModes::test_concurrency_picking` |
| `spec.features.planner`（不透明配置） | `TestDGDRFeatures::test_planner_enabled_*` |
| `spec.features.mocker.enabled` | `TestDGDRFeatures::test_mocker_enabled` |
| `spec.modelCache.pvcName` | `TestDGDRModelCache` |
| `spec.hardware.gpuSku` | `TestDGDRHardwareOverride::test_hardware_manual_override` |
| `spec.hardware.numGpusPerNode` | `TestDGDRHardwareOverride` |
| `spec.hardware.totalGpus` / `spec.hardware.vramMb` | `TestDGDRHardwareOverride::test_hardware_total_gpus_and_vram` |
| `spec.autoApply` | `TestDGDRAutoApply` |
| `spec.overrides.profilingJob` | `TestDGDROverrides::test_profiling_job_toleration_override` |
| `spec.overrides.dgd` | `TestDGDROverrides::test_dgd_override_injects_custom_labels` |
| `status.phase` | 所有生命周期测试 |
| `status.profilingPhase` | `TestDGDRStatusAndConditions::test_profiling_sub_phase_tracked` |
| `status.profilingJobName` | `TestDGDRStatusAndConditions::test_profiling_job_name_populated` |
| `status.dgdName` | `TestDGDRAutoApply`、`TestDGDRMinimalDeployment` |
| `status.profilingResults.selectedConfig` | 多处 |
| `status.profilingResults.pareto` | `TestDGDRStatusAndConditions::test_pareto_configs_in_profiling_results` |
| `status.deploymentInfo` | `TestDGDRMinimalDeployment` |
| `status.conditions`（所有类型） | `TestDGDRStatusAndConditions` |
| `status.observedGeneration` | `TestDGDRStatusAndConditions::test_observed_generation_tracks_spec` |

## GPU-free 模式（默认）

测试套件默认通过组合两个模拟特性，**完全无需 GPU 节点**就能跑完完整的 DGDR 生命周期：

| 特性 | 启用方式 | 影响阶段 |
|---|---|---|
| **AIC（AI Configurator）** | `searchStrategy: rapid`（默认） | **Profiling** —— profiler 走纯 CPU 模拟，而非在线 GPU 扫描 |
| **Mocker** | 默认开启（用 `--dgdr-no-mocker` 关闭） | **Deployment** —— DGD 使用 mock 推理 worker（不申请 GPU 资源） |

**工作原理：**

- v1beta1 DGDR 默认 `searchStrategy: rapid`。Profiler 在 rapid 下会自动使用 AI Configurator（AIC）模拟 —— 无需额外配置。
- Mocker 模式**默认开启**。`dgdr_factory` fixture 会自动给每个 DGDR 注入 `spec.features.mocker.enabled: true` 与默认的 `spec.hardware` 配置。
- AIC profiling 创建一个纯 CPU 运行的 Kubernetes Job（job 前缀 `profile-aic-`）。该 profiling pod 不申请 GPU 资源。
- Mocker 部署选择 profiler 输出的 `mocker_config_with_planner.yaml` 而非真实部署配置，从而让 DGD pod 不申请 GPU。
- 传入 `--dgdr-no-mocker` 即可关闭 mocker 模式，跑在真实 GPU 硬件上。

> **说明：** 一些断言（如 status.deploymentInfo.gpuCount、pareto 配置）在 mocker 与真实 GPU profiling 下取值可能不同。
> 测试主要验证结构与阶段切换，而非精确的 profiling 数值，因此两种模式下都能正常工作。

> **说明：** 即便配合 mocker，`searchStrategy: thorough` 仍需要在线（GPU）profiling，因为 thorough 会做真实的基准测量。请使用 rapid 进行 GPU-free 测试。

> **说明：** mocker 模式下 `TestDGDRFeatures::test_planner_enabled_with_rapid_sweep` 以 `auto_apply=False` 运行（与下方说明同因——operator 在 profiling 输出之后预先设置 `Status.DGDName`，然后由于找不到该 DGD 立即触发 `handleDGDDeleted`）。mocker 模式下该测试只验证 spec 生成成功（等到 `PHASE_READY` 后检查 `dgdName` + `selectedConfig`）。带 rapid sweep 的完整部署在非 mocker 模式下验证。`test_planner_enabled_no_pre_deployment_sweep` 与 `test_mocker_enabled` 在 mocker 模式下也只跑到 `PHASE_READY`。

> **说明：** mocker 模式下 `auto_apply=True` 一直会触发 `handleDGDDeleted`。Operator 的 `generateDGDSpec` 在 DGD 真正创建之前，已经把 `Status.DGDName` 预填为 profiling 输出（如 `mocker-disagg`）；然后 `handleDeployingPhase` 检查到 `DGDName != ""`，立即去 GET 那个 DGD，由于尚不存在，就触发 `handleDGDDeleted`，DGDR 进入 Failed。
> 因此所有在 mocker 模式下会进入 Deploying 阶段的测试都改用 `auto_apply=False`/`PHASE_READY`（包括最小生命周期、后端选择、mocker 特性、planner-no-sweep、planner-rapid-sweep、DGD label 覆盖）。
> 那些**仅**为了验证 `auto_apply=True` 自动创建 DGD 的测试，在 mocker 模式下被跳过（`test_auto_apply_true_creates_dgd_automatically`、`test_deletion_does_not_remove_created_dgd`）。
> 非 mocker 模式（真实 GPU 集群）不受影响。

> **说明：** mocker 模式下 `TestDGDRImmutability::test_spec_immutable_in_deployed_via_dry_run` 是 **xfail**。该测试依赖会话级 fixture `deployed_dgdr`，而它在 mocker 模式下停留在 `PHASE_READY` 而非 `PHASE_DEPLOYED`。Webhook 的 `ValidateUpdate` 不可变性强制只在 DGDR 处于 `Deployed` 阶段时才生效，因此 server-dry-run 的修改会被接受而非拒绝。

> **说明：** `gpu_8` 测试无法配合 mocker 运行，需要真实的 8-GPU 节点。
> `TestDGDRSearchStrategies::test_thorough_strategy_completes` 使用 `searchStrategy: thorough`，会做真实 GPU 基准扫描；`TestDGDRMoEModels`（DeepSeek-R1）真实推理负载需要 8 GPU。GPU-free 运行请用 `-m "gpu_0 or gpu_1"` 把它们排除。

### AIC silicon 数据可用性

AIC 工作在 **silicon 模式**：它会按 `{gpu_sku}/{backend}/{backend_version}/` 组织结构，从随 `aiconfigurator` Python 包附带的预录每算子性能数据文件中查找。Mocker fixture 会向每个 DGDR 注入 `gpuSku: a100_sxm` —— 但该包对该 SKU 仅附带了 vllm 数据：

| Backend | a100_sxm 数据？ | mocker 结果 |
|---|---|---|
| `vllm` | ✅ 有 | profiling 成功 |
| `trtllm` | ✅ 有 | profiling 成功 |
| `sglang` | ❌ 缺失 | 测试 **自动跳过**（`a100_sxm` 没有 `sglang/0.5.8` 性能数据） |

要测试 sglang/trtllm，请在真实 GPU 集群上跑（`--dgdr-no-mocker`），AIC 可使用拥有相应数据文件的 GPU SKU。

## 清理

测试通过 `dgdr_factory` fixture 自行清理它们创建的 DGDR。如果某个测试被中断，可手动清理：

```bash
# Delete all DGDRs created by the test suite (they are labelled automatically)
kubectl delete dgdr -n default -l "test.dynamo/managed=true"

# If you used a custom namespace:
kubectl delete dgdr -n <namespace> -l "test.dynamo/managed=true"
```

## 架构说明

- 所有测试**仅通过 `kubectl` 子进程调用**与集群交互，与 Dynamo 测试套件其他部分一致。
- `dgdr_factory` fixture 通过 `yield` 保证无论测试结果如何都会清理 DGDR。
- 依赖可选 PVC（`--dgdr-pvc-name`）的测试在未提供该选项时会自动跳过。
- 各超时值可调，以适配不同 profiling 速度的集群。
