---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Amazon Elastic Kubernetes Service (EKS)
---

# 创建 EKS 集群的步骤

本指南演示如何在 Amazon Elastic Kubernetes Service（EKS）上运行 Dynamo 平台。

## 设置环境变量

整篇指南都会用到这些环境变量。如果想使用其他 region，请修改 `AWS_REGION`。

```bash
export AWS_REGION="us-east-1"
export CLUSTER_NAME="ai-dynamo"
export DYNAMO_NAMESPACE="dynamo-system"
export DYNAMO_RELEASE_VERSION="1.0.0"
```


## 安装 CLI

### 安装 AWS CLI（[AWS CLI 安装指南](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)）

```bash
sudo apt install unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### 安装 Kubernetes CLI（[针对 EKS 的 kubectl 安装指南](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)）

```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.35.2/2026-02-27/bin/darwin/amd64/kubectl
chmod +x ./kubectl
mkdir -p $HOME/bin && cp ./kubectl $HOME/bin/kubectl && export PATH=$HOME/bin:$PATH
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
```

### 安装 Eksctl CLI（[eksctl 安装指南](https://eksctl.io/installation/)）

```bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz
sudo mv /tmp/eksctl /usr/local/bin
```

### 安装 Helm CLI（[针对 EKS 的 Helm 设置](https://docs.aws.amazon.com/eks/latest/userguide/helm.html)）

```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

## 创建 EKS Auto Mode 集群

使用 Eksctl 与 `eksctl.yaml` 创建一个 EKS Auto Mode 集群。这会创建一个安装了 Amazon EFS CSI Driver 插件的 EKS Auto Mode 集群，稍后我们将使用 Amazon EFS 存储 Dynamo 所需的模型权重与编译产物。

```bash
# Use all availability zones in a region, exclude use1-az3 where EKS control plane is not available
export EKS_CP_AZS=$(aws ec2 describe-availability-zones \
      --region ${AWS_REGION} \
      --filters "Name=opt-in-status,Values=opt-in-not-required" \
      --query "AvailabilityZones[?ZoneId!='use1-az3'].[ZoneName]" \
      --output text | sed 's/ /, /g; s/^/  - /')

eksctl create cluster -f <(envsubst < templates/eksctl.yaml)
```
*注意：eksctl 会自动为你配置 kubeconfig 上下文；如果未自动配置，可运行：`aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME`*

### 创建 EKS Auto Mode 的 GPU NodePool

创建一个面向 **g5、g6、g6e、g7e、p5、p5e、p5en** 实例族的 GPU NodePool。

```bash
kubectl apply -f automode-np-gpu.yaml
```

## 创建默认 StorageClass

创建一个默认 StorageClass 以使用 EKS Auto Mode 的存储能力，这会让默认 StorageClass 为 NATS 等需要的有状态工作负载使用 EBS 卷。

```bash
kubectl apply -f - << EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: auto-ebs-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
allowedTopologies:
- matchLabelExpressions:
  - key: eks.amazonaws.com/compute-type
    values:
    - auto
provisioner: ebs.csi.eks.amazonaws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  encrypted: "true"
EOF
```

## 创建 Amazon EFS 共享文件系统

按照 [EFS 设置指南](efs.md) 创建一个 EFS 文件系统，并将其作为 Dynamo 工作负载的共享存储。

## 安装 Dynamo Kubernetes 平台

### 安装 Dynamo 平台
```bash
helm fetch https://helm.ngc.nvidia.com/nvidia/ai-dynamo/charts/dynamo-platform-"$DYNAMO_RELEASE_VERSION".tgz
helm install dynamo-platform dynamo-platform-"$DYNAMO_RELEASE_VERSION".tgz \
  --namespace "$DYNAMO_NAMESPACE" \
  --create-namespace
```

### 配置 HuggingFace TOKEN
```bash
export HF_TOKEN=<HF_TOKEN>
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=${HF_TOKEN} \
  -n ${DYNAMO_NAMESPACE}
```

### 验证安装

校验 Dynamo 平台 pod 是否处于运行状态，应当看到与下方类似的输出。

```bash
kubectl get pods -n ${DYNAMO_NAMESPACE}
NAME                                                              READY   STATUS    RESTARTS   AGE
dynamo-platform-dynamo-operator-controller-manager-ff54b5dstgcq   1/1     Running   0          106s
dynamo-platform-nats-0                                            2/2     Running   0          106s
```

校验 Dynamo CRD 已被安装：

```bash
kubectl get crds | grep dynamo
dynamocheckpoints.nvidia.com                      2026-03-17T13:18:05Z
dynamocomponentdeployments.nvidia.com             2026-03-17T13:18:06Z
dynamographdeploymentrequests.nvidia.com          2026-03-17T13:18:08Z
dynamographdeployments.nvidia.com                 2026-03-17T13:18:09Z
dynamographdeploymentscalingadapters.nvidia.com   2026-03-17T13:18:10Z
dynamomodels.nvidia.com                           2026-03-17T13:18:10Z
dynamoworkermetadatas.nvidia.com                  2026-03-17T13:18:11Z
```

## 部署一个 DynamoGraphDeployment（DGD）

| 清单 | 描述 |
|----------|-------------|
| `manifests/vllm/disagg.yaml` | 解耦的 prefill/decode DGD，使用 NIXL 与 LIBFABRIC backend over EFA。面向支持 GPUDirect RDMA 的 `g7e.12xlarge` 实例，用于在 prefill 与 decode worker 之间高吞吐传输 KV 缓存。 |
| `manifests/vllm/disagg-p5.yaml` | 解耦的 prefill/decode DGD，使用 NIXL 与 LIBFABRIC backend over EFA。面向带 8 个 EFA 设备（p5.48xlarge 每 1 GPU 4 个 EFA）的 `p5.48xlarge` 预留实例，TP-2 用于 Qwen3-32B。在预留容量（`karpenter.sh/capacity-type: reserved`）上使用 2 个 decode 与 6 个 prefill 副本。 |
| `manifests/vllm/disagg-tcp.yaml` | 解耦的 prefill/decode 推理图的另一种方案，使用 TCP 而非 EFA。面向不支持 EFA 的 `g6e.2xlarge` 实例。 |
| `manifests/vllm/agg.yaml` | 聚合（单 worker）推理图，由单个 vLLM worker 同时处理 prefill 与 decode。部署更简单，没有 KV 缓存传输开销。 |


### 在 EFS 上缓存模型

部署推理图之前，先把模型权重下载到共享 EFS 文件系统。每个 Dynamo recipe 都包含一个 `model-cache/model-download.yaml` Job 清单，可从 HuggingFace 下载模型。

把 recipe 的下载清单复制到本地 kustomize 目录后应用：

```bash
# Example: cache the Qwen3-32B model which we will be using later
cp ../../../recipes/qwen3-32b/model-cache/model-download.yaml manifests/model-download/model-download.yaml
kubectl kustomize manifests/model-download | kubectl -n ${DYNAMO_NAMESPACE} apply -f -
rm -f manifests/model-download/model-download.yaml
```

recipe 清单未为下载容器设置任何内存资源。如果没有内存请求，下载期间 Job pod 可能因 OOMKilled 被杀 —— 大模型尤甚。`manifests/model-download/` 中的 `kustomization.yaml` 通过 patch 加上内存请求来避免该问题。默认值为 `4Gi`。

对于更大的模型（例如 DeepSeek-R1、Nemotron-3-Super-120B），在应用前请在 `manifests/model-download/kustomization.yaml` 中提高该值：

```yaml
patches:
  - target:
      kind: Job
      name: model-download
    patch: |
      apiVersion: batch/v1
      kind: Job
      metadata:
        name: model-download
      spec:
        template:
          spec:
            containers:
              - name: model-download
                resources:
                  requests:
                    memory: "16Gi"   # increase for larger models
```

然后应用：

```bash
kubectl kustomize manifests/model-download | kubectl -n ${DYNAMO_NAMESPACE} apply -f -
```

监控下载 Job：

```bash
kubectl -n ${DYNAMO_NAMESPACE} get jobs model-download
kubectl -n ${DYNAMO_NAMESPACE} logs -f job/model-download
```

要重新下载（例如更换模型或修复 OOM 之后），先删除原 Job：

```bash
kubectl -n ${DYNAMO_NAMESPACE} delete job model-download
```

然后复制新的 recipe 清单并再次应用。

### 解耦服务

该示例部署一个解耦的 prefill/decode Dynamo 推理图，使用 NIXL 与 LIBFABRIC backend，并通过 [Elastic Fabric Adapter（EFA）](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html) 在 worker 间高吞吐传输 KV 缓存。

它面向支持 GPUDirect RDMA 的 `g7e.12xlarge` 实例，并使用预装了 [EFA Installer](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa-changelog.html) 的 Dynamo EFA-enabled vLLM 容器 `nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.0.0-efa-amd64`。

*注：支持 EFA 的实例类型完整列表见 [AWS EC2 文档](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html#efa-instance-types)。*

```yaml
        nodeSelector:
          node.kubernetes.io/instance-type: g7e.12xlarge
```

worker 间的 KV 缓存传输使用 [NIXL](https://github.com/ai-dynamo/nixl) 与 LIBFABRIC backend。通过向 vLLM 传入以下参数启用：

`--kv-transfer-config '{"kv_connector":"NixlConnector","kv_role":"kv_both","kv_connector_extra_config": {"backends": ["LIBFABRIC"]}}'`

*注：在不支持 EFA 的实例类型上，NIXL 的 libfabric backend 会自动回退到 TCP。然而 vLLM 的 `NixlConnector` 默认 buffer device 为 `cuda`，因此在没有 EFA 时，必须在 `kv-transfer-config` 中加上 `"kv_buffer_device":"cpu"` 以使解耦服务工作。*

为每个 worker pod 通过 `vpc.amazonaws.com/efa` 扩展资源申请一个 EFA 设备：
```yaml
      resources:
        requests:
          gpu: "1"
          custom:
            vpc.amazonaws.com/efa: "1"
        limits:
          gpu: "1"
          custom:
            vpc.amazonaws.com/efa: "1"
```
*注：EKS Auto Mode 已包含 EFA 设备插件，使 `vpc.amazonaws.com/efa` 扩展资源可用。*

所有 worker（prefill 与 decode）必须位于同一可用区，因为 EFA 流量不跨 AZ。使用 pod affinity 规则强制：

```yaml
        affinity:
          podAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              - topologyKey: "topology.kubernetes.io/zone"
                labelSelector:
                  matchLabels:
                    nvidia.com/dynamo-graph-deployment-name: "vllm-disagg"
```

```bash
kubectl -n ${DYNAMO_NAMESPACE} apply -f manifests/vllm/disagg.yaml
```
*注：`manifests/vllm/disagg-tcp.yaml` 提供使用 TCP 而非 EFA 的备选示例，面向 `g6e.2xlarge` 实例。*

确认所有 pod 进入 `Running` 状态：

```bash
kubectl -n ${DYNAMO_NAMESPACE} get pods
NAME                                                              READY   STATUS    RESTARTS   AGE
dynamo-platform-dynamo-operator-controller-manager-ff54b5dstgcq   1/1     Running   0          39m
dynamo-platform-nats-0                                            2/2     Running   0          39m
vllm-disagg-frontend-85f8476887-wwtwk                             1/1     Running   0          2m13s
vllm-disagg-vllmdecodeworker-510a1741-7666987b-tp58w              1/1     Running   0          2m13s
vllm-disagg-vllmprefillworker-510a1741-54f76d7954-tjgn8           1/1     Running   0          2m13s
```

```bash
kubectl -n ${DYNAMO_NAMESPACE} port-forward svc/vllm-disagg-frontend 8000:8000

curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-32B",
    "messages": [
    {
        "role": "user",
        "content": "In the heart of Eldoria, an ancient land of boundless magic and mysterious creatures, lies the long-forgotten city of Aeloria. Once a beacon of knowledge and power, Aeloria was buried beneath the shifting sands of time, lost to the world for centuries. You are an intrepid explorer, known for your unparalleled curiosity and courage, who has stumbled upon an ancient map hinting at ests that Aeloria holds a secret so profound that it has the potential to reshape the very fabric of reality. Your journey will take you through treacherous deserts, enchanted forests, and across perilous mountain ranges. Your Task: Character Background: Develop a detailed background for your character. Describe their motivations for seeking out Aeloria, their skills and weaknesses, and any personal connections to the ancient city or its legends. Are they driven by a quest for knowledge, a search for lost familt clue is hidden."
    }
    ],
    "stream": false,
    "max_tokens": 30
  }'
```

应当看到与下方类似的输出：

```bash
{"id":"chatcmpl-23a7c94b-99cb-42ca-ae56-2397aa5a560f","choices":[{"index":0,"message":{"content":"<think>\nOkay, so I need to develop a character background for someone who's an intrepid explorer in Eldoria, specifically focusing on their motivations,","role":"assistant","reasoning_content":null},"finish_reason":"length"}],"created":1773336002,"model":"Qwen/Qwen3-0.6B","object":"chat.completion","usage":{"prompt_tokens":196,"completion_tokens":30,"total_tokens":226,"prompt_tokens_details":{"audio_tokens":null,"cached_tokens":192}},"nvext":{"worker_id":{"prefill_worker_id":4265733549773195,"prefill_dp_rank":0,"decode_worker_id":7535192362430132,"decode_dp_rank":0},"timing":{"request_received_ms":1773336002136,"prefill_wait_time_ms":0.852483,"prefill_time_ms":12.90597,"ttft_ms":13.758453000000001,"total_time_ms":110.89621500000001,"kv_hit_rate":0.0}}}
```

*注：每个 worker 的首次请求会有较高时延，这是 NIXL backend 握手与初始化的开销，仅第一次传输会发生。*

查看日志：
```bash
kubectl logs -n ${DYNAMO_NAMESPACE} -l nvidia.com/dynamo-graph-deployment-name=vllm-disagg --all-containers=true --max-log-requests=20 --prefix=true --timestamps -f
```

清理：

```bash
kubectl -n ${DYNAMO_NAMESPACE} delete -f manifests/vllm/disagg.yaml
```

### 聚合服务

```bash
kubectl -n ${DYNAMO_NAMESPACE} apply -f manifests/vllm/agg.yaml
```

你的 pod 应当处于如下输出所示的运行状态，确保它们都是 "Running"。

```bash
kubectl -n ${DYNAMO_NAMESPACE} get pods
NAME                                                              READY   STATUS    RESTARTS   AGE
dynamo-platform-dynamo-operator-controller-manager-ff54b5dstgcq   1/1     Running   0          12m
dynamo-platform-nats-0                                            2/2     Running   0          12m
vllm-agg-frontend-ff8457bcf-tq9jh                                 1/1     Running   0          4m46s
vllm-agg-vllmdecodeworker-d0a70291-759df94478-8lc74               1/1     Running   0          4m46s
```

查看日志：
```bash
kubectl logs -n ${DYNAMO_NAMESPACE} -l nvidia.com/dynamo-graph-deployment-name=vllm-agg --all-containers=true --max-log-requests=20 --prefix=true --timestamps -f
```

```bash
kubectl -n ${DYNAMO_NAMESPACE} port-forward svc/vllm-agg-frontend 8000:8000

curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [
    {
        "role": "user",
        "content": "In the heart of Eldoria, an ancient land of boundless magic and mysterious creatures, lies the long-forgotten city of Aeloria. Once a beacon of knowledge and power, Aeloria was buried beneath the shifting sands of time, lost to the world for centuries. You are an intrepid explorer, known for your unparalleled curiosity and courage, who has stumbled upon an ancient map hinting at ests that Aeloria holds a secret so profound that it has the potential to reshape the very fabric of reality. Your journey will take you through treacherous deserts, enchanted forests, and across perilous mountain ranges. Your Task: Character Background: Develop a detailed background for your character. Describe their motivations for seeking out Aeloria, their skills and weaknesses, and any personal connections to the ancient city or its legends. Are they driven by a quest for knowledge, a search for lost familt clue is hidden."
    }
    ],
    "stream": false,
    "max_tokens": 30
  }'
```

应当看到与下方类似的输出：

```bash
{"id":"chatcmpl-093fac0e-f75e-43b5-90dc-96c8c77a2e7c","choices":[{"index":0,"message":{"content":"<think>\nOkay, I need to develop a character background for the explorer in Eldoria. Let me start by understanding the user's query. They mentioned","role":"assistant","reasoning_content":null},"finish_reason":"length"}],"created":1773443560,"model":"Qwen/Qwen3-0.6B","object":"chat.completion","usage":{"prompt_tokens":196,"completion_tokens":30,"total_tokens":226},"nvext":{"timing":{"request_received_ms":1773443560878,"total_time_ms":99.89782}}}%
```

清理：

```bash
kubectl -n ${DYNAMO_NAMESPACE} delete -f manifests/vllm/agg.yaml
```

## 在 ML 中使用 On-Demand Capacity Reservation（ODCR）与 Capacity Block（CB）

GPU 实例的按需获取通常较难。AWS 提供两种预留机制以保证 ML 工作负载的容量：

- [On-Demand Capacity Reservations（ODCR）](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-reservations.html) 在指定 AZ 中按任意时长预留容量。无论是否使用，都会按预留容量付费。
- [Capacity Blocks for ML](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-blocks.html) 在固定时间窗口内（数小时到数天）预留 GPU 实例。实例放置于 EC2 UltraClusters 中以获得低时延网络。Capacity Blocks 有明确结束时间，EC2 会在 block 到期前终止实例。

EKS Auto Mode 底层使用 Karpenter，它将预留容量建模为 `karpenter.sh/capacity-type: reserved`，并将其优先于按需与竞价。

> [!NOTE]
> 默认情况下，EKS Auto Mode 可以自动启动到开放的 ODCR，但不会优先它们。Capacity Blocks 永远不会被自动使用。两者都需要在 NodeClass 上显式配置 `capacityReservationSelectorTerms` 才会被优先并被标记为 `reserved`。

### 创建带容量预留的 NodeClass

创建一个引用 ODCR 或 Capacity Block 预留的 NodeClass。可按预留 ID 或标签选择。

首先，从 EKS Auto Mode 已经创建的 `default` NodeClass 中提取子网、安全组与 role 配置：

```bash
export NC_SUBNETS=$(kubectl get nodeclass default -o json | jq -c '.spec.subnetSelectorTerms')
export NC_SG=$(kubectl get nodeclass default -o json | jq -c '.spec.securityGroupSelectorTerms')
export NC_ROLE=$(kubectl get nodeclass default -o json | jq -r '.spec.role')
```

把 `<CR ID>` 替换为 EC2 控制台中的实际预留 ID：
```bash
export CR_ID=<CR ID>
kubectl apply -f - << EOF
apiVersion: eks.amazonaws.com/v1
kind: NodeClass
metadata:
  name: gpu-reserved
spec:
  role: ${NC_ROLE}
  subnetSelectorTerms: ${NC_SUBNETS}
  securityGroupSelectorTerms: ${NC_SG}
  capacityReservationSelectorTerms:
    # Select by reservation ID (ODCR or Capacity Block)
    - id: "${CR_ID}"
    # Or select by tags (can be combined)
    # - tags:
    #     team: "dynamo"
EOF
```

等待 capacityReservation 的状态变为 `active`：

```bash
kubectl get nodeclass gpu-reserved -o json | jq '.status.capacityReservations'
[
  {
    "availabilityZone": "us-east-2c",
    "endTime": "2026-03-18T11:30:00Z",
    "id": "cr-xxxxxxxxxxxxxx",
    "instanceMatchCriteria": "targeted",
    "instanceType": "p5.48xlarge",
    "ownerID": "xxxxxxxxxxx",
    "reservationType": "capacity-block",
    "state": "active"
  }
]
```

### 创建针对预留容量的 NodePool

创建一个引用 `gpu-reserved` NodeClass 并使用 `reserved` 容量类型的 NodePool。可选地包含 `on-demand` 与 `spot` 作为预留耗尽时的回退。

```bash
kubectl apply -f - << EOF
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: gpu-reserved
spec:
  disruption:
    budgets:
      - nodes: 10%
    consolidateAfter: 300s
    consolidationPolicy: WhenEmptyOrUnderutilized
  template:
    spec:
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: gpu-reserved
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values:
            - reserved
            # Uncomment to fallback to on-demand or spot when reservation is exhausted
            # - on-demand
            # - spot
        - key: eks.amazonaws.com/instance-family
          operator: In
          values:
            - g6e
            - g7e
            - p5
            - p5e
            - p5en
      taints:
        - effect: NoSchedule
          key: nvidia.com/gpu
          value: Exists
EOF
```

确认 `gpu-reserved` NodePool 就绪：

```bash
kubectl get nodepool gpu-reserved
NAME           NODECLASS      NODES   READY   AGE
gpu-reserved   gpu-reserved   0       True    8s
```

> [!NOTE]
> 当集群中任何 NodeClass 配置了 `capacityReservationSelectorTerms` 后，EKS Auto Mode 会停止对所有 NodeClass 自动使用开放 ODCR。请确保所有应使用 ODCR 的 NodeClass 都配置了显式的 selector terms。

### 让工作负载选中预留节点

pod 会通过现有 NodePool 的 requirements 与 taints 被调度到预留节点上。如果你希望某个工作负载只跑在预留容量上，添加 node selector：

```yaml
      nodeSelector:
        karpenter.sh/capacity-type: reserved
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
```

### Capacity Blocks 注意事项

Capacity Blocks 有固定结束时间。EC2 会在 block 过期前 30 分钟开始终止实例（UltraServer 类型为 60 分钟）。Karpenter 会在 EC2 终止前 10 分钟开始驱逐节点，给工作负载留出优雅关闭的时间。

请相应规划你的推理工作负载，并考虑在 NodePool 中加入 `on-demand` 作为回退容量类型，以在 Capacity Block 窗口之外保持连续性。

## 清理

删除全部 DynamoGraphDeployment：

```bash
kubectl -n ${DYNAMO_NAMESPACE} get dgd

# If you have any, delete them
kubectl -n ${DYNAMO_NAMESPACE} delete dgd <name>
```

卸载 Dynamo 平台：
```bash
helm uninstall -n ${DYNAMO_NAMESPACE} dynamo-platform
```

清理 NATS 残留 PVC：
```bash
kubectl -n ${DYNAMO_NAMESPACE} get pvc
NAME                                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
dynamo-platform-nats-js-dynamo-platform-nats-0   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   10Gi       RWO            auto-ebs-sc    <unset>                 75m

kubectl -n ${DYNAMO_NAMESPACE} delete pvc dynamo-platform-nats-js-dynamo-platform-nats-0
```

删除 AutoMode GPU NodePool：

```bash
kubectl delete nodepool gpu
```

按 [EFS 设置指南](efs.md#cleanup) 的清理章节清理 EFS 相关资源。

使用 Eksctl 删除 EKS Auto Mode 集群：

```bash
eksctl delete cluster -f <(envsubst < templates/eksctl.yaml)
```
