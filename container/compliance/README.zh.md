# 容器合规性工具（Container Compliance Tooling）

用于从已构建的容器镜像生成归属（attribution）CSV 的脚本，列出所有已安装的 dpkg 与 Python 软件包及其已知的 SPDX 许可证标识。

## 输出格式

每次运行最多产生两个 CSV 文件：

| 列 | 描述 |
|--------|-------------|
| `package_name` | dpkg 或 pip 报告的包名 |
| `version` | 已安装版本 |
| `type` | `dpkg` 或 `python` |
| `spdx_license` | SPDX 标识（如 `MIT`、`Apache-2.0`）或 `UNKNOWN` |

文件按 `(type, package_name)` 排序以保证 diff 稳定。

当提供基础镜像时，会额外生成第二个 `_diff.csv` 文件，仅包含相对基础镜像新增或版本变化的软件包 —— 也就是 Dynamo 在上游镜像之上构建层所新增的内容。

## 本地使用

### 前置条件

- 支持 [BuildKit](https://docs.docker.com/build/buildkit/) 的 Docker（Docker 23+）
- Python 3.11+

### 第 1 步 —— 创建本地 BuildKit builder（一次性）

```bash
docker buildx create --use --name compliance-builder
```

### 第 2 步 —— 从镜像中提取软件包

```bash
docker buildx build \
  --builder compliance-builder \
  --platform linux/amd64 \
  --build-arg TARGET_IMAGE=<image:tag> \
  --output type=local,dest=./output \
  --pull \
  --no-cache-filter extractor \
  --progress=plain \
  -f container/compliance/Dockerfile.extract \
  container/compliance/
```

会生成 `./output/dpkg.tsv` 与 `./output/python.tsv` —— 制表符分隔的文件，每行格式为 `package_name\tversion\tspdx_license`。

> **为什么使用 `--no-cache-filter extractor`？** 当源是 stage 名称（与直接的镜像引用相比）时，BuildKit 对 `RUN --mount=type=bind,from=<stage>` 的缓存键并不可靠地包含挂载 stage 的内容摘要（content digest）。如果不加该标记，即使 `--pull` 解析到新的 digest，缓存命中仍可能返回上一次针对不同镜像生成的 TSV。
> `--no-cache-filter extractor` 仅强制重跑提取阶段；`python:3.12-slim` 基础层与辅助脚本的 COPY 仍会被缓存。

### 第 3 步 —— 转换为 CSV

```bash
python container/compliance/process_results.py \
  --target-dir ./output \
  --output attribution.csv
```

### 含基础镜像 diff 的完整示例

使用 `resolve_base_image.py` 从 `container/context.yaml` 中查找正确的基础镜像，避免硬编码 URI：

```bash
# 从 context.yaml 解析基础镜像（需要：pip install pyyaml）
BASE_IMAGE=$(python container/compliance/resolve_base_image.py \
  --framework vllm \
  --cuda-version 12.9)

# 提取目标镜像
docker buildx build \
  --builder compliance-builder \
  --platform linux/amd64 \
  --build-arg TARGET_IMAGE=<image:tag> \
  --output type=local,dest=./output \
  --pull \
  --no-cache-filter extractor \
  -f container/compliance/Dockerfile.extract \
  container/compliance/

# 提取基础镜像
docker buildx build \
  --builder compliance-builder \
  --platform linux/amd64 \
  --build-arg TARGET_IMAGE="${BASE_IMAGE}" \
  --output type=local,dest=./base-output \
  --pull \
  --no-cache-filter extractor \
  -f container/compliance/Dockerfile.extract \
  container/compliance/

# 生成包含 diff 的 CSV
python container/compliance/process_results.py \
  --target-dir ./output \
  --base-dir ./base-output \
  --output attribution.csv
# 产出：attribution.csv（完整）与 attribution_diff.csv（相对基础的增量）
```

### resolve_base_image.py 参数

| 参数 | 默认值 | 描述 |
|------|---------|-------------|
| `--framework` | *（必填）* | `vllm`、`sglang`、`trtllm` 或 `dynamo` |
| `--target` | `runtime` | `runtime` 或 `frontend` |
| `--cuda-version` | — | runtime 目标必填（如 `12.9`、`13.0`、`13.1`）|
| `--context-yaml` | `container/context.yaml` | context.yaml 路径 |

### process_results.py 参数

| 参数 | 默认值 | 描述 |
|------|---------|-------------|
| `--target-dir` | *（必填）* | 包含目标提取产物 `dpkg.tsv` 与 `python.tsv` 的目录 |
| `--base-dir` | — | 包含基础镜像提取产物的 TSV 目录（启用 `_diff.csv` 输出）|
| `--output`, `-o` | stdout | 输出 CSV 路径 |

## 基础镜像参考

| 框架 | CUDA | 基础镜像 |
|-----------|------|------------|
| `vllm` | 12.9 | `nvcr.io/nvidia/cuda:12.9.1-runtime-ubuntu24.04` |
| `vllm` | 13.0 | `nvcr.io/nvidia/cuda:13.0.2-runtime-ubuntu24.04` |
| `sglang` | 12.9 | `lmsysorg/sglang:v0.5.11-cu129-runtime` |
| `sglang` | 13.0 | `lmsysorg/sglang:v0.5.11-cu130-runtime` |
| `trtllm` | 13.1 | `nvcr.io/nvidia/cuda-dl-base:25.12-cuda13.1-runtime-ubuntu24.04` |
| `dynamo` frontend | — | `nvcr.io/nvidia/base/ubuntu:noble-20250619` |

上述值来自 `container/context.yaml`；表格反映当前默认值。

## 工作原理

提取过程使用 BuildKit 的 bind-mount 机制 —— 将目标镜像的文件系统以只读方式挂载到 Python 3.12 builder 容器内的 `/target`，两个辅助脚本直接从磁盘读取软件包元数据，无需启动目标容器：

- **`helpers/dpkg_helper.py`** —— 解析 `/target/var/lib/dpkg/status` 列出已安装包，并读取 `/target/usr/share/doc/<pkg>/copyright`（DEP-5 格式）以获取许可证信息。
- **`helpers/python_helper.py`** —— 通过 `importlib.metadata` 枚举 `/target` 下的 site-packages 目录。许可证依次从 `License-Expression`（PEP 639）、`License` 元数据、trove classifier 中读取。

两个辅助脚本均自包含（仅依赖标准库），运行在 `python:3.12-slim` 提取阶段中，而非目标镜像内部。

## 许可证识别

识别策略刻意保守：仅在能够明确匹配时才赋予 SPDX 标识。`UNKNOWN` 项是预期的；可通过对原始 copyright 文件做进一步分析来解决。

## CI 集成

每次镜像构建成功后，CI 都会自动生成归属 CSV。产物可在 GitHub Actions 工作流运行结果中获取：

- `compliance-{framework}-cuda{major}-{platform}` —— runtime 镜像
- `compliance-frontend-{arch}` —— frontend 镜像

该扫描作为独立 Job 与测试并行运行，因此不会延长流水线总耗时。
