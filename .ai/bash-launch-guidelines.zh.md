# Bash 启动脚本规范

本仓库中用于启动推理引擎（vLLM、SGLang、TensorRT-LLM）的 bash 脚本所遵循的规则与约定。这些规则适用于 `examples/backends/*/launch/`、`tests/serve/launch/`，以及任何启动 `dynamo.frontend`、`dynamo.vllm`、`dynamo.sglang` 或 `dynamo.trtllm` 的脚本。

## 这些规范为何存在

启动脚本是测试框架与推理引擎之间的接口。当它们遵循一致模式时，许多原本不可能的事情才得以成立：

- **GPU 测试并行执行。** 测试框架在同一台机器上并发运行多个 GPU 测试。这只有当每个测试的启动脚本能从环境接受 VRAM 预算（通过 `gpu_utils.sh`）和唯一端口（`DYN_HTTP_PORT`、`DYN_SYSTEM_PORT`）时才成立。硬编码端口或让引擎抢占全部 VRAM 的脚本无法参与并行运行。

- **故障即时检测。** 推理栈运行多个协作进程（前端、worker、路由）。如果其中之一崩溃而脚本未察觉，测试会一直挂到全局超时被杀 —— 浪费 GPU 时间且产生无用日志。`wait_any_exit` 立即检测第一个子进程的故障并撤销所有其他进程，让故障在数秒而非数分钟内浮现。

- **一致、可调试的日志。** 当所有脚本都输出相同的启动横幅（模型、端口、GPU 内存参数、示例 curl）时，从 CI 日志定位失败测试就直截了当。否则每个脚本各自打印不同的内容（或什么都不打），需要逆向推断当时使用的配置。

- **降低重复与漂移。** 共享工具（`gpu_utils.sh`、`launch_utils.sh`）在一处维护。bug 修复和新功能（例如对新引擎内存控制 flag 的支持）会自动传播到所有脚本。当脚本各自重复实现这些逻辑时，它们会随时间漂移并悄然损坏。

- **降低贡献者门槛。** 一个新启动脚本基本是模板代码 —— source 两个文件、设置一个模型、后台启动进程、调用 `wait_any_exit`。这让新增部署配置变得简单，无需理解进程管理、VRAM 预算或端口分配的内部细节。

## 关键规则

这些是最重要的约定。例外应该极少且在代码注释中给出充分理由。

### Source 共享工具库

代码库中的启动脚本都 source `gpu_utils.sh` 与 `launch_utils.sh`，从单一可维护的位置共享进程管理、VRAM 预算与横幅打印逻辑。新启动脚本应遵循同一约定：

```bash
SCRIPT_DIR="$(dirname "$(readlink -f "$0")")"
source "$SCRIPT_DIR/../../../common/gpu_utils.sh"   # build_vllm_gpu_mem_args、build_sglang_gpu_mem_args 等
source "$SCRIPT_DIR/../../../common/launch_utils.sh" # print_launch_banner、wait_any_exit
```

对于位于 examples 树之外的测试脚本，使用 `DYNAMO_HOME`：

```bash
export DYNAMO_HOME="${DYNAMO_HOME:-/workspace}"
source "${DYNAMO_HOME}/examples/common/gpu_utils.sh"
source "${DYNAMO_HOME}/examples/common/launch_utils.sh"
```

**始终标记**那些重新实现 GPU 内存参数构造、`wait_any_exit` 或横幅打印，而不是 source 共享库的启动脚本。

**始终标记**那些手动检查 `_PROFILE_OVERRIDE_*` 环境变量、而非调用相应 `build_*_gpu_mem_args` 函数的启动脚本。

### 使用引擎专用的 GPU 内存函数控制 VRAM

现有启动脚本调用 `gpu_utils.sh` 中的引擎专用函数，并把结果传给引擎 CLI 以支持 GPU 并行测试执行。如果不这么做，引擎会抢占全部 VRAM，导致并发测试相互 OOM。

每个引擎都有自己的函数，因为 CLI flag 语义不同：

```bash
# vLLM —— 返回 --kv-cache-memory-bytes N --gpu-memory-utilization 0.01
GPU_MEM_ARGS=$(build_vllm_gpu_mem_args)
python -m dynamo.vllm --model "$MODEL" $GPU_MEM_ARGS &

# SGLang —— 返回 --max-total-tokens N
GPU_MEM_ARGS=$(build_sglang_gpu_mem_args)
python -m dynamo.sglang --model-path "$MODEL" $GPU_MEM_ARGS &

# TensorRT-LLM —— 为 --override-engine-args 返回 JSON（具备合并意识）
JSON=$(build_trtllm_override_args_with_mem)
python -m dynamo.trtllm --model-path "$MODEL" ${JSON:+--override-engine-args "$JSON"} &
```

```bash
# 反例 —— 手动环境变量检查；逻辑重复，容易出错
if [[ -n "${_PROFILE_OVERRIDE_VLLM_KV_CACHE_BYTES:-}" ]]; then
    GPU_MEM_ARGS="--kv-cache-memory-bytes $_PROFILE_OVERRIDE_VLLM_KV_CACHE_BYTES"
fi
```

### 使用 `wait_any_exit` 而不是裸 `wait` 或前台进程

代码库中的启动脚本都把所有进程放到后台，并以 `wait_any_exit` 作为最后一行立即检测故障。任何一个子进程崩溃都会让脚本以那个错误码退出，并由 EXIT trap 清理其余进程。

```bash
# 推荐 —— 全部后台，第一处故障立即被检测
python -m dynamo.frontend &
python -m dynamo.vllm --model "$MODEL" $GPU_MEM_ARGS &
wait_any_exit

# 反例 —— 如果 frontend 崩溃，脚本会阻塞在前台 vllm 进程
python -m dynamo.frontend &
python -m dynamo.vllm --model "$MODEL"

# 反例 —— `wait` 会阻塞直到所有子进程退出；
# 一个的崩溃要等其他全部完成（或永远挂起）才会浮现
python -m dynamo.frontend &
python -m dynamo.vllm --model "$MODEL" &
wait
```

### 让端口可通过环境变量注入

启动脚本应从环境接受 `DYN_HTTP_PORT` 与 `DYN_SYSTEM_PORT`，使测试框架能为并行执行分配唯一端口。代码库中普遍采用此约定，让多个推理栈在同一台机器上并发运行而不发生端口冲突。

```bash
# 推荐 —— 测试框架可覆盖；默认值适合手动使用
HTTP_PORT="${DYN_HTTP_PORT:-8000}"
python -m dynamo.frontend &

DYN_SYSTEM_PORT=${DYN_SYSTEM_PORT:-8081} \
    python -m dynamo.vllm --model "$MODEL" &

# 反例 —— 硬编码端口；两个并发测试会冲突
python -m dynamo.frontend --http-port 8000 &
python -m dynamo.vllm --model "$MODEL" &
```

启动多 worker 的脚本，应使用编号端口变量（`DYN_SYSTEM_PORT1`、`DYN_SYSTEM_PORT2` 等），或在基础端口之上偏移计算。

## 标准脚本结构

一份组织良好的启动脚本按以下顺序：

```bash
#!/bin/bash
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
#
# Brief description of what this script launches.

set -e
trap 'echo Cleaning up...; kill 0' EXIT

SCRIPT_DIR="$(dirname "$(readlink -f "$0")")"
source "$SCRIPT_DIR/../../../common/gpu_utils.sh"
source "$SCRIPT_DIR/../../../common/launch_utils.sh"

# ---- 默认模型 ----
MODEL="Qwen/Qwen3-0.6B"

# ---- 解析 CLI 参数 ----
EXTRA_ARGS=()
while [[ $# -gt 0 ]]; do
    case $1 in
        --model) MODEL="$2"; shift 2 ;;
        *)       EXTRA_ARGS+=("$1"); shift ;;
    esac
done

# ---- 可调参数（可通过环境变量覆盖）----
MAX_MODEL_LEN="${MAX_MODEL_LEN:-4096}"

GPU_MEM_ARGS=$(build_vllm_gpu_mem_args)

HTTP_PORT="${DYN_HTTP_PORT:-8000}"
print_launch_banner "Launching <description>" "$MODEL" "$HTTP_PORT"

# ---- 启动进程 ----
python -m dynamo.frontend &

DYN_SYSTEM_PORT=${DYN_SYSTEM_PORT:-8081} \
    python -m dynamo.vllm --model "$MODEL" \
    --max-model-len "$MAX_MODEL_LEN" \
    $GPU_MEM_ARGS \
    "${EXTRA_ARGS[@]}" &

wait_any_exit
```

### 必备元素

| 元素 | 原因 |
|---------|-----|
| `#!/bin/bash` | 一致的 shebang（不要 `#!/bin/sh` —— 我们需要 bash 特性） |
| SPDX 许可头 | CI 版权检查要求 |
| `set -e` | 第一处错误即退出 |
| `trap 'echo Cleaning up...; kill 0' EXIT` | 退出时撤销所有子进程 |
| `source gpu_utils.sh` | 访问 `build_vllm_gpu_mem_args`、`build_sglang_gpu_mem_args`、`build_trtllm_override_args_with_mem` |
| `source launch_utils.sh` | 访问 `wait_any_exit`、`print_launch_banner` |
| `GPU_MEM_ARGS=$(build_<engine>_gpu_mem_args)` | VRAM 安全的并行执行 |
| `DYN_HTTP_PORT` / `DYN_SYSTEM_PORT` 可注入 | 端口安全的并行执行 |
| `print_launch_banner` | 一致、可调试的启动日志 |
| 所有进程用 `&` 放入后台 | `wait_any_exit` 必需 |
| `wait_any_exit` 作为最后一行 | 即时故障检测 |

### 通过环境变量提供可调参数

启动脚本应将关键参数以环境变量形式暴露，并提供合理默认值：

| 变量 | 用途 | 典型默认值 |
|----------|---------|-----------------|
| `MODEL` 或 `MODEL_PATH` | 要服务的模型 | `Qwen/Qwen3-0.6B` 或类似小模型 |
| `MAX_MODEL_LEN` | 最大序列长度 | `4096` |
| `MAX_CONCURRENT_SEQS` | 最大并发序列数 | `2` |
| `DYN_HTTP_PORT` | 前端 HTTP 端口 | `8000` |
| `DYN_SYSTEM_PORT` | Worker 系统端口 | `8081` |
| `CUDA_VISIBLE_DEVICES` | GPU 分配 | 从环境继承 |

## 不要这么做

**始终标记**启动脚本中的这些反模式：

- 硬编码端口（如 `--http-port 8000` 没有环境变量回退）
- 手动处理 `_PROFILE_OVERRIDE_*` 环境变量而不是 `build_*_gpu_mem_args`
- 把最后一个进程放到前台而不是后台 + `wait_any_exit`
- 使用裸 `wait` 而不是 `wait_any_exit`
- 缺少 `set -e`
- 缺少 EXIT trap 的清理
- 在共享库提供所需功能时不 source `gpu_utils.sh` / `launch_utils.sh`
- 用 `sleep N` 等待服务就绪，而不是合适的健康检查

## 共享工具参考

两个文件均位于 `examples/common/`，启动脚本通过 source（而非执行）使用。

### `gpu_utils.sh`

三个引擎专用函数，每个返回 GPU 内存控制的 CLI flag（或 JSON）。当未设置覆盖环境变量时全部返回空，引擎就使用默认分配。

| 函数 | 环境变量 | 输出 |
|----------|---------|--------|
| `build_vllm_gpu_mem_args` | `_PROFILE_OVERRIDE_VLLM_KV_CACHE_BYTES` | `--kv-cache-memory-bytes N --gpu-memory-utilization 0.01` |
| `build_sglang_gpu_mem_args` | `_PROFILE_OVERRIDE_SGLANG_MAX_TOTAL_TOKENS` | `--max-total-tokens N` |
| `build_trtllm_override_args_with_mem` | `_PROFILE_OVERRIDE_TRTLLM_MAX_TOTAL_TOKENS` 或 `_PROFILE_OVERRIDE_TRTLLM_MAX_GPU_TOTAL_BYTES` | 给 `--override-engine-args` 的 JSON |

vLLM 还会得到 `--gpu-memory-utilization 0.01`，因为当共驻测试使用了 >10% 的 VRAM 时 vLLM 的启动检查会拒绝启动（它在应用字节上限*之前*用该比例核对剩余内存）。把它设为 0.01 可绕过该检查。

TensorRT-LLM 的函数支持 `--merge-with-json`，把 GPU 内存配置与已有 `--override-engine-args` JSON（如性能指标、tracing 端点）合并。

`--kv-cache-memory-bytes` 的值是按进程的：每个 vLLM worker 拿到相同值，即便在多 worker per GPU 的配置中也如此。Profiler 直接找到每 worker 的预算。

自检：`bash examples/common/gpu_utils.sh --self-test` 运行内置断言。

关于使用绝对值上限而非内存比例的理由，参见
[`examples/common/gpu_utils.md`](../examples/common/gpu_utils.md)。

### `launch_utils.sh`

需要 **bash 4.3+**（使用 `wait -n`）。文件在 source 时检查并在 bash 版本太旧时报错退出。

**`wait_any_exit`** —— 等待任意后台子进程退出，并将其退出码向上传播。捕获 TERM/INT 并以 0 退出（干净关停），这样测试 harness 向进程组发送 SIGTERM 时不会产生虚假的非零退出码。如果未发现后台任务（捕获缺失的 `&`）会输出诊断信息。

**`print_launch_banner [flags] <title> <model> <port> [extra_lines...]`** —— 打印带模型信息和示例 curl 命令的启动横幅。当设置了 `MAX_MODEL_LEN`（或 `CONTEXT_LENGTH` / `MAX_SEQ_LEN`）以及 `GPU_MEM_ARGS` 时自动包含在横幅中，让测试日志清晰显示当时使用的配置。

Flag（位于位置参数之前）：
- `--multimodal` —— 使用多模态（image_url）curl 示例（`max_tokens=50`）
- `--max-tokens N` —— 覆盖 curl 示例中的 `max_tokens`（默认：32）
- `--no-curl` —— 仅打印横幅，跳过示例 curl 部分

**`print_curl_footer`** —— 从 stdin 读取自定义 curl 示例并以标准框架包装。与 `print_launch_banner --no-curl` 配合用于非标准端点（图像、视频、embedding）的自定义请求体。

**常量：** `EXAMPLE_PROMPT`（文本 LLM prompt）、`EXAMPLE_PROMPT_VISUAL`（图像/视频生成 prompt）。

## 相关文档

- **[`examples/common/gpu_utils.md`](../examples/common/gpu_utils.md)** —— GPU 内存控制深度解析：为何用绝对值上限而非比例、各引擎语义（vLLM 的字节 vs SGLang 的 token vs TensorRT-LLM 的比例），以及 GPU 内存函数如何融入 profiling 与调度（scheduler）流水线。
- **[`tests/README.md`](../tests/README.md)** —— VRAM profiler（`profile_pytest.py`）、用于 VRAM 预算的 pytest 标记，以及测试框架如何设置 GPU 内存函数读取的 `_PROFILE_OVERRIDE_*` 环境变量。
- **`.ai/bash-launch-guidelines.md`** —— 本文件
  （CodeRabbit 会按本文件审查启动脚本）。
