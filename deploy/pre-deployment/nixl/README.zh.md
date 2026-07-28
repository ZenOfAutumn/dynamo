# NIXL Benchmark 文档

本指南介绍如何在 Kubernetes（K8s）集群上使用提供的脚本构建并部署 NIXL benchmark。

> **注意**：NIXL benchmark 是 Dynamo 平台的一部分。在继续之前，请运行上级目录中的预部署检查脚本（`../pre-deployment-check.sh`），确保你的集群满足 Dynamo 的基本要求。

---

## 前置条件

### 集群要求
在部署 NIXL benchmark 之前，运行预部署检查以确认集群满足 Dynamo 平台要求：

```bash
# Run from the parent directory
../pre-deployment-check.sh
```

该脚本会校验：
- `kubectl` 连通性与集群访问
- GPU 节点可用性（带 `nvidia.com/gpu.present=true` 标签）
- GPU Operator 已安装且状态正常

### NIXL 专属要求
除上述集群要求之外，NIXL benchmark 还需要：
- 本地机器上已安装并配置 **Docker**（用于构建镜像）
- 可访问的 **Docker registry**，用于推送构建好的 nixlbench 镜像
- **ETCD 服务**已部署，可通过 `etcd:2379` 访问
- **构建工具**：用于下载 NIXL 源码的 `wget` 与 `unzip`

### 验证步骤
1. **运行预部署检查**（推荐）：
   ```bash
   ../pre-deployment-check.sh
   ```
   确保所有检查通过后再继续。

2. **确认 ETCD 可用**（NIXL 专属）：
   ```bash
   kubectl get svc etcd
   ```

3. **确认 Docker registry 可访问**：
   ```bash
   docker login your-registry.com  # if using private registry
   ```

---

## 快速开始

最简单的方式是使用交互式构建与部署脚本：

```bash
./build_and_deploy.sh
```

该脚本提供灵活的工作流，你可以：
1. **选择架构**：在 x86_64（Intel/AMD 64 位）与 aarch64（ARM64）之间选择
2. **选择要执行的步骤**：以下任意组合：
   - 构建 nixlbench Docker 镜像
   - 更新部署 YAML 文件
   - 部署到 Kubernetes
3. **提供 Docker registry**（仅在构建或更新部署时需要）

---

## 交互式脚本特性

### 架构选择
脚本支持两种架构：
- **选项 1**：x86_64（Intel/AMD 64 位）
- **选项 2**：aarch64（ARM64）

可以通过输入以下值选择：
- `1` 或 `x86_64` 选 x86_64 架构
- `2` 或 `aarch64` 选 aarch64 架构

### 步骤选择
通过输入以逗号分隔的数字来选择要执行的步骤：

- **全部步骤**：`1,2,3`
- **仅构建并更新**：`1,2`（跳过 K8s 部署）
- **仅部署**：`3`（用于镜像已构建且部署文件已存在的情况）
- **仅构建**：`1`（仅创建 Docker 镜像）
- **仅更新部署**：`2`（用于以新 registry/版本更新部署文件）

### 智能 registry 询问
脚本只在需要时询问 Docker registry 信息：
- **步骤 1 或 2**：构建镜像或更新部署时需要 registry
- **仅步骤 3**：不询问 registry（使用现有部署文件）

---

## 各步骤的作用

### 步骤 1：构建 nixlbench Docker 镜像
- 从 GitHub 下载 NIXL 源码（版本 0.10.1）
- 解压并切换到构建目录
- 在构建前等待用户确认
- 用指定的 registry 与架构构建 Docker 镜像
- 镜像 tag 为：`{registry}/nixlbench:0.10.1-{arch}`

### 步骤 2：更新部署 YAML 文件
- 复制基础部署模板（`nixlbench-deployment.yaml`）
- 创建架构相关的部署文件（`nixlbench-deployment-{arch}.yaml`）
- 根据你的 registry 与架构更新镜像引用
- 保留所有其他部署配置

### 步骤 3：部署到 Kubernetes
- 校验部署文件存在
- 把部署应用到 Kubernetes 集群
- 提供用于检查状态的监控命令

---

## 部署配置

部署会创建：
- nixlbench pod 的 **2 个副本**
- **资源 requests/limits**：
  - CPU：10 核
  - 内存：5Gi
  - GPU：每个 pod 1 张 NVIDIA GPU
- **环境变量**：
  - `ETCD_ENDPOINTS`：指向 `etcd:2379`
- **Command**：以 VRAM segments 运行 nixlbench 并保持容器存活

---

## 使用示例

### 示例 1：完整流程
```bash
./build_and_deploy.sh
# Select: 1 (x86_64)
# Steps: 1,2,3
# Registry: docker.io/myusername
# Confirm: y
```

### 示例 2：仅构建镜像
```bash
./build_and_deploy.sh
# Select: 2 (aarch64)
# Steps: 1
# Registry: my-private-registry.com
# Confirm: y
```

### 示例 3：部署已有镜像
```bash
./build_and_deploy.sh
# Select: 1 (x86_64)
# Steps: 3
# Confirm: y
```

### 示例 4：仅更新部署文件
```bash
./build_and_deploy.sh
# Select: 2 (aarch64)
# Steps: 2
# Registry: new-registry.com
# Confirm: y
```

---

## 生成的文件

脚本会生成与架构相关的部署文件：
- `nixlbench-deployment-x86_64.yaml` —— 用于 x86_64 构建
- `nixlbench-deployment-aarch64.yaml` —— 用于 aarch64 构建

它们是基础模板的定制版本，包含你指定的：
- Docker registry
- 镜像 tag
- 架构

---

## 监控部署

部署完成后，监控你的 NIXL benchmark：

```bash
# Check pod status
kubectl get pods -l app=nixl-benchmark

# View logs
kubectl logs -l app=nixl-benchmark -f

# Check resource usage
kubectl top pods -l app=nixl-benchmark

# Get detailed pod information
kubectl describe pods -l app=nixl-benchmark
```

如果部署到指定命名空间：
```bash
kubectl get pods -l app=nixl-benchmark -n your-namespace
kubectl logs -l app=nixl-benchmark -f -n your-namespace
```

---


## 故障排查

### 集群层面问题
对于集群相关问题，先运行预部署检查以定位问题：

```bash
../pre-deployment-check.sh
```

它能帮助诊断：
- kubectl 连通性问题
- 缺失默认 StorageClass
- GPU 节点可用性问题
- GPU Operator 状态问题

### NIXL 专属问题

1. **ETCD 连接**：
   - 确认 etcd 服务在运行：`kubectl get svc dynamo-platform-etcd`
   - 验证 pod 能访问 etcd 端点
   - 检查 etcd 是否在正确的命名空间下

2. **镜像拉取问题**：
   - 验证 registry 凭证已配置
   - 确认镜像存在：`docker pull {registry}/nixlbench:0.10.1-{arch}`
   - 确认构建后镜像已成功推送

3. **构建失败**：
   - 确认 Docker daemon 正在运行
   - 检查 `/tmp` 可用磁盘空间
   - 验证到 GitHub 的网络连通性
   - 确认构建工具已安装：`which wget unzip`

4. **找不到部署文件**：
   - 在执行步骤 3 之前先用步骤 2 创建部署文件
   - 检查脚本目录的文件权限
   - 确认脚本目录路径正确

### 调试命令
```bash
# Check script-generated files
ls -la nixlbench-deployment-*.yaml

# Verify deployment status
kubectl get deployment nixl-benchmark -o yaml

# Check events for issues
kubectl get events --sort-by=.metadata.creationTimestamp
```

### 清理

移除部署：
```bash
kubectl delete deployment nixl-benchmark
```

或在指定命名空间：
```bash
kubectl delete deployment nixl-benchmark -n your-namespace
```

清理生成的文件：
```bash
rm -f nixlbench-deployment-*.yaml
```

---

## 脚本参考

### build_and_deploy.sh
交互式脚本，提供灵活的构建与部署工作流：
- **架构选择**：x86_64 或 aarch64
- **步骤选择**：构建、更新、部署的任意组合
- **校验**：在部署前检查部署文件存在

### nixlbench-deployment.yaml
基础 Kubernetes 部署模板，由脚本进行定制：
- **模板镜像**：`my-registry/nixlbench:version-arch`
- **资源分配**：每个 pod 10 CPU、5Gi 内存、1 张 GPU
- **ETCD 集成**：预配置的环境变量
- **Benchmark 命令**：以 VRAM segment 配置运行
