# 用于 Dynamo 基准测试与剖析的 Kubernetes 工具

本目录提供 Dynamo 基准测试（benchmarking）与性能剖析（profiling）流程所需的工具与清单（manifest）。

## 前置条件

**使用这些工具之前，必须先按照主安装指南完成 Dynamo Kubernetes Platform 的部署：**

👉 **[请先按 Dynamo Kubernetes Platform 安装指南](/docs/kubernetes/installation-guide.md) 安装 Dynamo Kubernetes Platform。**

它包括：
1. 安装 Dynamo CRD
2. 安装 Dynamo Platform（Operator、etcd、NATS）
3. 配置目标命名空间

## 目录内容

- `setup_benchmarking_resources.sh` —— 在已有的 Dynamo 命名空间中配置基准测试与性能剖析资源
- `manifests/`
  - `pvc.yaml` —— PVC `dynamo-pvc`
  - `pvc-access-pod.yaml` —— 短期存活的 Pod，用于从 PVC 复制 profiler 结果
- `kubernetes.py` —— 工具链使用的辅助脚本，用于应用/读取资源（如用于 PVC 访问的 access pod）
- `dynamo_deployment.py` —— 用于操作 DynamoGraphDeployment 资源的工具
- `requirements.txt` —— 基准测试工具的 Python 依赖

## 快速开始

### 基准测试资源准备

完成 Dynamo Kubernetes Platform 的部署后，使用此脚本为命名空间准备基准测试与性能剖析所需的额外资源：

该脚本会创建一个 `dynamo-pvc`，访问模式为 `ReadWriteOnce`（RWO），使用集群默认的存储类（storage class）。这对一次只有一个任务写入的 profiling 工作流已足够。

如果你想使用 `ReadWriteMany`（RWX）以支持并发访问，请在运行脚本前修改 `deploy/utils/manifests/pvc.yaml`：

```yaml
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: <your-rwx-capable-storageclass>  # 例如：基于 NFS 的存储
  resources:
    requests:
      storage: 50Gi
```

> [!TIP]
> **检查集群可用的 storage class**
>
> - 列出存储类与 provisioner：
> ```bash
> kubectl get sc -o wide
> ```

```bash
export NAMESPACE=your-dynamo-namespace
export HF_TOKEN=<HF_TOKEN>  # 可选：用于访问 HuggingFace 模型

deploy/utils/setup_benchmarking_resources.sh
```

该脚本会向你已有的 Dynamo 命名空间应用以下清单：

- `deploy/utils/manifests/pvc.yaml` —— PVC `dynamo-pvc`

如果设置了 `HF_TOKEN`，还会创建用于访问 HuggingFace 模型的 Secret。

运行脚本后，可通过以下命令验证资源：

```bash
kubectl get pvc dynamo-pvc -n $NAMESPACE
```

### 操作 PVC

PVC（Persistent Volume Claim）用于存放配置文件以及基准测试/性能剖析结果。可使用 `kubectl cp` 从 PVC 上传/下载文件。

#### 配置 PVC 访问

首先创建一个临时的 access pod 用于操作 PVC：

```bash
# 创建 access pod
kubectl apply -f deploy/utils/manifests/pvc-access-pod.yaml -n $NAMESPACE

# 等待 Pod 就绪
kubectl wait --for=condition=Ready pod/pvc-access-pod -n $NAMESPACE --timeout=60s
```

#### 上传文件到 PVC

**用于 profiling 的部署配置：**

```bash
# 复制单个文件
kubectl cp ./my-disagg.yaml $NAMESPACE/pvc-access-pod:/data/configs/disagg.yaml

# 复制整个目录
kubectl cp ./configs/ $NAMESPACE/pvc-access-pod:/data/configs/
```

#### 从 PVC 下载文件

**下载基准测试结果：**

```bash
# 下载整个 results 目录
kubectl cp $NAMESPACE/pvc-access-pod:/data/results ./benchmarks/results

# 下载某个子目录
kubectl cp $NAMESPACE/pvc-access-pod:/data/results/benchmark-name ./benchmarks/results/benchmark-name
```

**查看 profiling 结果（可选，本地查阅用）：**

```bash
# 查看 profiling 生成的 DGD 配置
kubectl get configmap dgdr-output-<dgdr-name> -n $NAMESPACE -o yaml

# 查看 planner 的 profiling 数据（JSON 格式）
kubectl get configmap planner-profile-data -n $NAMESPACE -o yaml
```

> **关于 Profiling 结果的说明**：当使用 DGDR（DynamoGraphDeploymentRequest）进行 SLA 驱动的 profiling 时，profiling 数据会自动存入 ConfigMap：
> - `dgdr-output-<dgdr-name>`：包含生成的 DynamoGraphDeployment YAML
> - `planner-profile-data`：包含 JSON 格式的 profiling 性能数据，供 planner 使用
>
> planner 组件直接从挂载的 ConfigMap 读取该数据，因此无需 PVC。

#### 清理 access pod

完成后，删除 access pod：

```bash
kubectl delete pod pvc-access-pod -n $NAMESPACE
```

#### 路径结构

**PVC 中常见的路径模式：**
- `/data/configs/` —— 配置文件（DGD 清单）
- `/data/results/` —— 基准测试结果（基准测试 Job 完成后下载）
- `/data/benchmarking/` —— 基准测试相关产物

#### 后续步骤

完整的基准测试与性能剖析流程：
- **基准测试指南**：参见 [docs/benchmarks/benchmarking.md](../../docs/benchmarks/benchmarking.md)，用于对比 DynamoGraphDeployment 与外部端点
- **部署前性能剖析**：参见 [docs/components/profiler/profiler-guide.md](../../docs/components/profiler/profiler-guide.md)，用于在部署前优化配置

## 备注

- 此目录仅聚焦基准测试与性能剖析所需资源 —— Dynamo 主平台需另行安装。
