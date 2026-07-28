# CUDA 故障注入 - 测试库

**用途**：在测试中安全模拟 GPU 故障（XID 错误），且不会损坏真实硬件。

> **⚠️ 注意**：本目录仅包含 **C 语言库源代码**。该库会在 Kubernetes 测试中**于 Pod 内部编译**以保证 Linux 兼容性。除非你要进行独立的本地测试，否则**不需要**在本地构建。

## 功能概述

通过 LD_PRELOAD 拦截 CUDA 调用以模拟 GPU 故障。借助 hostPath 卷，故障在 Pod 重启后仍可保持，从而支持真实的硬件故障测试。

```
Pod calls cudaMalloc() → LD_PRELOAD intercepts → Checks /host-fault/cuda_fault_enabled → Returns error → Pod crashes
```

**关键特性**：
- **持久化故障**：hostPath 卷（`/var/lib/cuda-fault-test`）在同一节点上的 Pod 重启后依然存在
- **运行时切换**：通过 `/host-fault/cuda_fault_enabled` 启用/禁用故障，无需重启 Pod
- **节点定向**：故障仅作用于目标节点，健康节点不受影响

## 适用范围

本库模拟当 GPU 硬件不可访问或不可用时出现的**软件/编排层级故障**：
- ✅ **范围内**：因 GPU 不可用导致的 CUDA API 失败（XID 错误、设备未找到、ECC 错误）
- ✅ **使用场景**：测试 Kubernetes Pod 重新调度、推理（inference）故障转移、恢复编排
- ❌ **范围外**：比特级静默数据损坏（SDC）、计算错误、错误结果
- ❌ **不建模**：计算/内存层面上的通用 GPU 故障现象

**注意**：由于我们在 CUDA API 层而非硬件/计算层进行拦截，因此该方法不会触发 SDC 检测机制。

## 支持的 XID 错误

| XID | 说明 | CUDA Error | 使用场景 |
|-----|-------------|------------|----------|
| **79** | GPU 从总线脱落（GPU fell off bus） | `CUDA_ERROR_NO_DEVICE` | 最常见的节点级故障 |
| **48** | 双比特 ECC 错误 | `CUDA_ERROR_ECC_UNCORRECTABLE` | 内存损坏 |
| **94** | 已隔离的 ECC 错误 | `CUDA_ERROR_ECC_UNCORRECTABLE` | 可恢复的内存错误 |
| **95** | 未隔离错误 | `CUDA_ERROR_UNKNOWN` | 致命 GPU 错误 |
| **43** | GPU 停止响应 | `CUDA_ERROR_LAUNCH_TIMEOUT` | 内核（kernel）挂起 |
| **74** | NVLink 错误 | `CUDA_ERROR_PEER_ACCESS_UNSUPPORTED` | 多 GPU 通信失败 |

## 工作原理

1. **Deployment 补丁**：添加 hostPath 卷与用于编译库的 init 容器
2. **LD_PRELOAD 注入**：通过环境变量在 CUDA 之前加载该库
3. **运行时控制**：通过开关文件（`/host-fault/cuda_fault_enabled`）控制故障状态
4. **节点持久化**：hostPath 确保同节点上 Pod 重启后故障仍然存在

## 本目录文件

| 文件 | 用途 |
|------|---------|
| `cuda_intercept.c` | 用于拦截 CUDA 调用并检查故障标记的 C 库 |
| `inject_into_pods.py` | Kubernetes 部署补丁工具（添加 hostPath 卷与库） |
| `Makefile` | 本地构建（可选，用于测试） |

## 前置条件

- **gcc 编译器**（用于构建库）
- **kubectl** 且具备集群访问权限
- Python 包：`kubernetes`、`requests`
- 不需要本地编译（在 Pod 内部编译）

### 独立本地测试（可选）
- **gcc**（推荐 7.5+，任意现代 gcc 均可）
- **CUDA 开发头文件**（可选，仅使用 runtime API）

## 编写自定义测试

### 引入辅助函数

```python
import sys
from pathlib import Path

# Add cuda_fault_injection to path
cuda_injection_dir = Path(__file__).parent.parent / "cuda_fault_injection"
sys.path.insert(0, str(cuda_injection_dir))

from inject_into_pods import (
    create_cuda_fault_configmap,      # Step 1: Create ConfigMap with library source
    patch_deployment_env,             # Step 2: Patch deployment to use it
    delete_cuda_fault_configmap       # Cleanup: Remove ConfigMap
)
```
