---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: 在 AKS 上的 RDMA / InfiniBand
---

# 在 AKS 上的 RDMA / InfiniBand

本指南介绍如何在 AKS 上为 Dynamo 的高性能解耦推理搭建基于 InfiniBand 的 RDMA。RDMA 实现跨节点 GPU 之间的直接内存访问，绕过 CPU 与内核开销 —— 这对于 prefill 与 decode worker 之间低延迟的 KV 缓存（KV cache）传输至关重要。

如果没有 RDMA，解耦推理会回退到 TCP，性能会严重下降（TTFT 约 98 秒 vs 使用 RDMA 时 200-500 毫秒）。关于传输选项与性能预期的详情，参见 [Disaggregated Communication Guide](../../disagg-communication-guide.md)。

> [!NOTE]
> 本指南中的 Network Operator 与 NicClusterPolicy 步骤基于 [Azure AKS RDMA InfiniBand](https://github.com/Azure/aks-rdma-infiniband) 仓库。该项目是开源的且不在 Microsoft Azure 支持范围内 —— 请在该 GitHub 仓库中提交 issue。

## 前置条件

**带 RDMA 能力节点的 AKS 集群：**

- 至少 **2 个 GPU 节点** 以启用跨节点 RDMA 通信
- 带 Mellanox ConnectX InfiniBand 网卡的 **ND 系列虚拟机**（如 `Standard_ND96asr_v4`、`Standard_ND96isr_H100_v5`）
- 节点池上使用 **Ubuntu OS**（NVIDIA 驱动兼容性需要）
- 节点池**跳过** GPU 驱动安装（`--skip-gpu-driver-install`）—— 详见 [GPU Node Pool Setup](aks.md#step-2-add-a-gpu-node-pool)

**注册 AKS InfiniBand 特性**以确保节点落在同一物理 InfiniBand 网络上：

```bash
az feature register --namespace Microsoft.ContainerService --name AKSInfinibandSupport
az feature show --namespace Microsoft.ContainerService --name AKSInfinibandSupport --query "properties.state"
# 等待出现 "Registered"

az provider register --namespace Microsoft.ContainerService
```

## 概览

RDMA 搭建涉及五个组件，按以下顺序安装：

1. **Network Operator** —— 部署 Mellanox OFED 驱动与 Node Feature Discovery
2. **NicClusterPolicy** —— 在支持 InfiniBand 的节点上配置 OFED 驱动
3. **IB 节点配置** —— 加载 InfiniBand 内核模块并设置 memlock 上限
4. **RDMA Shared Device Plugin** —— 将 InfiniBand 网卡作为 Kubernetes 资源暴露给 pod
5. **GPU Operator** —— 以 RDMA 特定设置安装（关闭 NFD、启用 GPUDirect RDMA、使用宿主 MOFED）

## 步骤 1：安装 NVIDIA Network Operator

[NVIDIA Network Operator](https://docs.nvidia.com/networking/display/cokan10/network+operator) 自动部署网络组件，包括用于 InfiniBand 支持的 Mellanox OFED 驱动。

创建命名空间并标注以允许特权工作负载：

```bash
kubectl create ns network-operator
kubectl label --overwrite ns network-operator pod-security.kubernetes.io/enforce=privileged
```

添加 NVIDIA Helm 仓库（如尚未添加）：

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
```

创建 `network-operator-values.yaml`：

```yaml
nfd:
  deployNodeFeatureRules: false
```

安装 Network Operator：

```bash
helm install network-operator nvidia/network-operator \
  --namespace network-operator \
  -f network-operator-values.yaml \
  --version v26.1.0
```

确认 Network Operator pod 已运行：

```bash
kubectl get pods -n network-operator
```

## 步骤 2：应用 NicClusterPolicy

NicClusterPolicy 在所有支持 InfiniBand 的节点上以 DaemonSet 形式配置 OFED 驱动（Mellanox OFED / DOCA driver）。

使用 kustomize 应用基础 NicClusterPolicy：

```bash
kubectl apply -k https://github.com/Azure/aks-rdma-infiniband/configs/nicclusterpolicy/base
```

它针对带有 Mellanox 网卡（`feature.node.kubernetes.io/pci-15b3.present`）的节点，并以 DaemonSet 形式安装 DOCA/OFED 驱动。

等待 MOFED 驱动 DaemonSet 在所有节点上完成安装（可能需要数分钟）：

```bash
kubectl get pods -n network-operator -l app=mofed-ubuntu22.04-ds -w
# 等到所有 pod 显示 Running
```

## 步骤 3：部署 IB 节点配置 DaemonSet

该 DaemonSet 在 GPU 节点上加载 InfiniBand 内核模块并设置无限制 memlock 上限。这是 RDMA 正常工作的前提 —— 否则 InfiniBand 设备文件可能不存在，且 RDMA 传输的内存固定（pinning）会失败。

> [!IMPORTANT]
> 该步骤未在 Azure RDMA 仓库中涵盖，但对完整搭建是必需的。该 DaemonSet 加载 `ib_umad` 与 `rdma_ucm` 内核模块，为 containerd 与 kubelet 设置无限制 memlock，并重启两个服务以使变更生效。

创建 `ib-node-config.yaml`：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ib-node-config
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: ib-node-config
  template:
    metadata:
      labels:
        app: ib-node-config
    spec:
      hostPID: true
      nodeSelector:
        kubernetes.azure.com/agentpool: <GPU_NODE_POOL_NAME>
      tolerations:
      - operator: Exists
      initContainers:
      - name: ib-setup
        image: busybox:1.36
        securityContext:
          privileged: true
        command:
        - sh
        - -c
        - |
          echo "=== IB Node Configuration ==="

          nsenter -t 1 -m -u -i -n -- modprobe ib_umad
          nsenter -t 1 -m -u -i -n -- modprobe rdma_ucm 2>/dev/null || true
          nsenter -t 1 -m -u -i -n -- modprobe ib_ucm 2>/dev/null || true
          nsenter -t 1 -m -u -i -n -- lsmod | grep ib_umad && echo "OK: ib_umad" || echo "FAIL: ib_umad"
          nsenter -t 1 -m -u -i -n -- ls /dev/infiniband/rdma_cm && echo "OK: rdma_cm device" || echo "WARN: no rdma_cm device"

          nsenter -t 1 -m -u -i -n -- sh -c 'printf "ib_umad\nrdma_ucm\n" > /etc/modules-load.d/ib-umad.conf'
          nsenter -t 1 -m -u -i -n -- sh -c 'printf "* - memlock unlimited\nroot - memlock unlimited\n" > /etc/security/limits.d/99-ib-memlock.conf'

          nsenter -t 1 -m -u -i -n -- sh -c 'mkdir -p /etc/systemd/system/containerd.service.d && printf "[Service]\nLimitMEMLOCK=infinity\n" > /etc/systemd/system/containerd.service.d/memlock.conf'
          nsenter -t 1 -m -u -i -n -- sh -c 'mkdir -p /etc/systemd/system/kubelet.service.d && printf "[Service]\nLimitMEMLOCK=infinity\n" > /etc/systemd/system/kubelet.service.d/memlock.conf'

          nsenter -t 1 -m -u -i -n -- systemctl daemon-reload
          nsenter -t 1 -m -u -i -n -- systemctl restart containerd
          nsenter -t 1 -m -u -i -n -- systemctl restart kubelet

          sleep 10

          nsenter -t 1 -m -u -i -n -- systemctl is-active containerd && echo "OK: containerd active" || echo "FAIL: containerd"
          nsenter -t 1 -m -u -i -n -- systemctl is-active kubelet && echo "OK: kubelet active" || echo "FAIL: kubelet"

          echo "=== Setup Complete ==="
      containers:
      - name: keepalive
        image: busybox:1.36
        command: ["sh", "-c", "echo IB node config active; sleep infinity"]
```

> [!NOTE]
> 将 `<GPU_NODE_POOL_NAME>` 替换为你的 GPU 节点池名（如 `ndh100pool`）。

```bash
kubectl apply -f ib-node-config.yaml
```

等待所有 pod 完成初始化：

```bash
kubectl get pods -n kube-system -l app=ib-node-config -w
```

**它做了什么：**
- **`ib_umad`** —— InfiniBand 用户态管理数据报模块，RDMA 设备访问所需
- **`rdma_ucm`** —— RDMA 用户态连接管理器
- **memlock 上限** —— RDMA 需要固定内存页；没有无限制 memlock，大型传输会失败
- **服务重启** —— containerd 与 kubelet 必须重启以应用新的 memlock 限制

## 步骤 4：部署 RDMA Shared Device Plugin

RDMA Shared Device Plugin 将 InfiniBand 网卡暴露为 Kubernetes 扩展资源，让 pod 能请求 RDMA 访问。

创建带设备插件配置的 ConfigMap：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rdma-devices
  namespace: kube-system
data:
  config.json: |
    {
        "periodicUpdateInterval": 300,
        "configList": [{
             "resourceName": "hca_shared_devices_a",
             "rdmaHcaMax": 1000,
             "selectors": {
               "vendors": ["15b3"],
               "drivers": ["mlx5_core"]
             }
           }
        ]
    }
```

创建 DaemonSet：

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: rdma-shared-dp-ds
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: rdma-shared-dp-ds
  template:
    metadata:
      labels:
        name: rdma-shared-dp-ds
    spec:
      hostNetwork: true
      nodeSelector:
        kubernetes.azure.com/agentpool: <GPU_NODE_POOL_NAME>
      tolerations:
      - operator: Exists
      containers:
      - name: k8s-rdma-shared-dp-ds
        image: ghcr.io/mellanox/k8s-rdma-shared-dev-plugin:v1.5.3
        securityContext:
          privileged: true
        volumeMounts:
        - name: device-plugin
          mountPath: /var/lib/kubelet/device-plugins
        - name: plugins-registry
          mountPath: /var/lib/kubelet/plugins_registry
        - name: config
          mountPath: /k8s-rdma-shared-dev-plugin
        - name: devs
          mountPath: /dev/
      volumes:
      - name: device-plugin
        hostPath:
          path: /var/lib/kubelet/device-plugins
      - name: plugins-registry
        hostPath:
          path: /var/lib/kubelet/plugins_registry
      - name: config
        configMap:
          name: rdma-devices
      - name: devs
        hostPath:
          path: /dev/
```

> [!NOTE]
> 将 `<GPU_NODE_POOL_NAME>` 替换为你的 GPU 节点池名（如 `ndh100pool`）。

```bash
kubectl apply -f rdma-configmap.yaml
kubectl apply -f rdma-shared-dp-ds.yaml
```

等待设备插件 pod 启动：

```bash
kubectl get pods -n kube-system -l name=rdma-shared-dp-ds -w
```

## 步骤 5：安装 GPU Operator（启用 RDMA）

使用 RDMA 特定的 values 安装 GPU Operator：

```bash
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator --create-namespace \
  --set nfd.enabled=false \
  --set driver.rdma.enabled=true \
  --set driver.rdma.useHostMofed=true
```

与标准 GPU Operator 安装的关键区别：

- `nfd.enabled=false` —— Network Operator 已部署 Node Feature Discovery；运行两个 NFD 实例会冲突
- `driver.rdma.enabled=true` —— 启用 GPUDirect RDMA；让驱动 daemonset 构建并加载 `nvidia_peermem`
- `driver.rdma.useHostMofed=true` —— 让 GPU Operator 使用步骤 1 中 Network Operator 安装的 MOFED 驱动，而不是它自己的；当 Network Operator 管理 OFED 时必需

等待 GPU Operator pod 进入 `Running` 状态：

```bash
kubectl get pods -n gpu-operator -w
```

## 验证

**1. 检查 MOFED 驱动 pod 在所有 InfiniBand 节点上运行：**

```bash
kubectl get pods -n network-operator -l app=mofed-ubuntu22.04-ds
```

**2. 检查 IB 节点配置 pod 已完成初始化：**

```bash
kubectl get pods -n kube-system -l app=ib-node-config
```

**3. 检查 RDMA Shared Device Plugin 已运行：**

```bash
kubectl get pods -n kube-system -l name=rdma-shared-dp-ds
```

**4. 验证 GPU 节点上 RDMA 资源可用：**

```bash
kubectl get nodes -o json | jq '.items[] | select(.status.allocatable["rdma/hca_shared_devices_a"] != null) | {name: .metadata.name, rdma: .status.allocatable["rdma/hca_shared_devices_a"], gpu: .status.allocatable["nvidia.com/gpu"]}'
```

每个支持 InfiniBand 的节点应报告 `rdma/hca_shared_devices_a` 资源（基于 `rdmaHcaMax: 1000` 通常为 `1k`）。

**5. 检查 GPU Operator pod 健康：**

```bash
kubectl get pods -n gpu-operator
```

## Pod 资源请求

需要 RDMA 访问的 Dynamo pod 应请求 `rdma/hca_shared_devices_a` 资源。当对带 RDMA 能力的集群使用 Dynamo operator 与 DGDR 时，对解耦部署该步骤会自动处理。

对手写 DGD 规格，向你的容器添加资源请求：

```yaml
resources:
  limits:
    nvidia.com/gpu: 8
    rdma/hca_shared_devices_a: 1
```

> [!NOTE]
> 当遵循本搭建时，**`IPC_LOCK` 能力不是必须的**。`IPC_LOCK` 在 RDMA 上历史上是必需的，因为 `ibv_reg_mr` 调用 `mlock()` 来固定内存页 —— 但只有当 memlock rlimit 会阻止时 `mlock()` 才需要该能力。步骤 3 的 `ib-node-config` DaemonSet 在 kubelet 与 containerd 的 systemd 单元上设置了 `LimitMEMLOCK=infinity`，因此 GPU 节点上的所有 pod 都继承无限制 memlock，RDMA 内存固定无需在 pod spec 中加任何能力即可工作。
>
> 如果你看到来自 `ibv_reg_mr` 的 `ENOMEM` 错误，且 `ib-node-config` 在运行，请确认在限制应用后 containerd 与 kubelet 已重启（查看 init container 日志）。如果未部署 `ib-node-config`，请在 pod 的 `securityContext.capabilities.add` 中加入 `IPC_LOCK`。

## 故障排查

**MOFED pod 卡在 `Init` 或 `CrashLoopBackOff`：**
- 确认节点为 Ubuntu OS：`kubectl get nodes -o custom-columns="NAME:.metadata.name,OS:.status.nodeInfo.osImage"`
- 查看 MOFED pod 日志：`kubectl logs -n network-operator <mofed-pod> -c mofed-container`

**节点上未出现 `rdma/hca_shared_devices_a`：**
- 检查 RDMA 设备插件 pod 是否运行：`kubectl get pods -n kube-system -l name=rdma-shared-dp-ds`
- 查看设备插件日志：`kubectl logs -n kube-system <rdma-shared-dp-pod>`
- 确认 `rdma-devices` ConfigMap 存在：`kubectl get configmap rdma-devices -n kube-system`

**IB 内核模块未加载：**
- 查看 ib-node-config init container 日志：`kubectl logs -n kube-system <ib-node-config-pod> -c ib-setup`
- 确认 MOFED 驱动已先安装（步骤 2 必须早于步骤 3 完成）

**RDMA 传输期间出现 memlock 错误（来自 `ibv_reg_mr` 的 `ENOMEM`）：**
- 确认 ib-node-config DaemonSet 已在所有 GPU 节点运行且 init container 已完成
- 检查 containerd 与 kubelet 已重启：`kubectl logs -n kube-system <ib-node-config-pod> -c ib-setup`
- 确认限制对 kubelet 进程生效：
  ```bash
  # 在 GPU 节点上（通过 kubectl debug 或 ssh）
  cat /proc/$(pgrep -x kubelet)/limits | grep -i memlock
  # 应显示：Max locked memory  unlimited  unlimited
  ```
- 如果限制不是 unlimited，需要重新应用 ib-node-config DaemonSet 并重启服务

**GPUDirect RDMA 不工作 —— 缺少 `nvidia_peermem` 模块：**

ND 系列节点（包括 ND H100 v5）的宿主 OS **不**自带 `nvidia_peermem`。该模块对 InfiniBand 适配器直接读写 GPU 显存是必需的 —— 没有它 RDMA 传输会回退到经过宿主内存的 staging。

确认模块是否已加载：

```bash
# 通过特权 pod 或节点 shell 在 GPU 节点上检查
lsmod | grep nvidia_peermem
# 如果为空，模块未加载

modinfo nvidia_peermem
# 如果显示 "Module not found"，宿主 /lib/modules 中也没有
```

当 GPU Operator 管理驱动（`driver.rdma.enabled=true`）时，`nvidia_peermem` 由 `nvidia-driver-daemonset` 构建并加载 —— 它存在于驱动 pod 的 `/lib/modules` 中，而不是宿主原生内核模块中。验证驱动 daemonset 在加载它：

```bash
kubectl exec -n gpu-operator $(kubectl get pod -n gpu-operator -l app=nvidia-driver-daemonset -o jsonpath='{.items[0].metadata.name}') -- lsmod | grep nvidia_peermem
```

如果返回为空，确保 GPU Operator Helm values 中设置了 `driver.rdma.enabled=true` 与 `driver.rdma.useHostMofed=true`（参见上方 [步骤 5](#步骤-5安装-gpu-operator启用-rdma)），然后重启驱动 daemonset：

```bash
kubectl rollout restart daemonset/nvidia-driver-daemonset -n gpu-operator
```

> [!NOTE]
> Azure RDMA 仓库中的 [nvidia-peermem-reloader](https://github.com/Azure/aks-rdma-infiniband/tree/main/configs/nvidia-peermem-reloader) DaemonSet 是为使用 **AKS 托管 GPU 驱动** 的集群（不带 GPU Operator）设计的。它仅运行 `modprobe nvidia-peermem` —— 在 ND H100 v5 节点上会失败，因为宿主 OS 不包含该模块。当使用 GPU Operator（推荐）时，operator 会通过 `driver.rdma.enabled=true` 自动处理 `nvidia_peermem`。

## 参见

- [Azure AKS RDMA InfiniBand — GitHub](https://github.com/Azure/aks-rdma-infiniband)
- [Set up InfiniBand on Azure HPC VMs — Microsoft Learn](https://learn.microsoft.com/en-us/azure/virtual-machines/setup-infiniband)
- [Enable InfiniBand VM extension — Microsoft Learn](https://learn.microsoft.com/en-us/azure/virtual-machines/extensions/enable-infiniband)
- [NVIDIA Network Operator 文档](https://docs.nvidia.com/networking/display/cokan10/network+operator)
- [Disaggregated Communication Guide](../../disagg-communication-guide.md) —— 传输选项、UCX 配置、性能预期

