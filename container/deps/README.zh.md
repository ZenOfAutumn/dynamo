# 容器 Python 依赖

按组件拆分的 Dynamo 容器镜像 Python 依赖文件，使每个镜像只安装其所需依赖。

## 文件

| 文件 | 用途 |
|------|---------|
| `requirements.common.txt` | 所有容器共享的核心依赖 |
| `requirements.planner.txt` | Planner、profiler 和 global_planner 依赖 |
| `requirements.frontend.txt` | 前端（frontend）依赖 |
| `requirements.vllm.txt` | vLLM 专用依赖 |
| `requirements.benchmark.txt` | 基准测试与性能分析工具 |
| `requirements.test.txt` | 仅测试用依赖 |
| `requirements.dev.txt` | 仅开发用工具 |

## 版本固定策略

- 对纯 Python 且经过良好测试的包使用 `==`。
- 对可能存在平台特定构建（CUDA、系统包）的包使用 `<=` 或 `<`，因为 x86_64 / aarch64 与不同 CUDA 版本之间可获取的最大版本可能不同。
- **永远不要使用 `>=`**，因为这会允许未经测试的未来版本，可能引入破坏性变更、造成不可复现的构建并引发依赖冲突。每一个安装版本都应被显式测试过。
