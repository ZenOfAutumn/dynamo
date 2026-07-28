<!--
SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->
# 创建 EKS 集群的步骤

本指南演示如何在 Amazon Elastic Kubernetes Service（EKS）上部署 Dynamo 平台。

## 设置环境变量

下面这些环境变量会贯穿整篇指南。
如需使用其他区域，请修改 `AWS_REGION` 变量。

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

### 安装 Helm CLI（[针对 EKS 的 Helm 安装](https://docs.aws.amazon.com/eks/latest/userguide/helm.html)）

```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

## 创建 EKS Auto Mode 集群

使用 Eksctl 配合 `eksctl.yaml` 创建一个 EKS Auto Mode 集群（cluster）。
此过程会创建一个 EKS Auto Mode 集群，并把 Amazon EFS CSI Driver 作为 addon 一并安装好；之后我们会用 Amazon EFS 来保存 Dynamo 使用的模型权重与编译产物。

```bash
# Use all availability zones in a region, exclude use1-az3 where EKS control plane is not available
export EKS_CP_AZS=$(aws ec2 describe-availability-zones \
      --region ${AWS_REGION} \
      --filters "Name=opt-in-status,Values=opt-in-not-required" \
      --query "AvailabilityZones[?ZoneId!='use1-az3'].[ZoneName]" \
      --output text | sed 's/ /, /g; s/^/  - /')

eksctl create cluster -f <(envsubst < templates/eksctl.yaml)
```
*注：eksctl 会自动配置 kubeconfig 上下文，如未自动配置可运行：`aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME`*

### 创建 EKS Auto Mode GPU NodePool

创建一个面向 **g5、g6、g6e、g7e、p5、p5e、p5en** 实例族的 GPU NodePool。

```bash
kubectl apply -f automode-np-gpu.yaml
```

## 创建默认的 StorageClass

创建一个默认 StorageClass 来使用 EKS Auto Mode 的存储能力。这样默认 StorageClass 就会使用 EBS 卷，从而满足 Dynamo 所用 NATS 的有状态工作负载需求。

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

按 [EFS 安装指南](EFS.md) 创建一个 EFS 文件系统，并将其作为 Dynamo 工作负载的共享存储。

## 安装 Dynamo Kubernetes 平台

### 安装 Dynamo Platform
```bash
helm fetch https://helm.ngc.nvidia.com/nvidia/ai-dynamo/charts/dynamo-platform-"${DYNAMO_RELEASE_VERSION}".tgz
helm install dynamo-platform dynamo-platform-"${DYNAMO_RELEASE_VERSION}".tgz --namespace "${DYNAMO_NAMESPACE}" --create-namespace
```

### 配置 HuggingFace TOKEN
```bash
export HF_TOKEN=<HF_TOKEN>
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN=${HF_TOKEN} \
  -n ${DYNAMO_NAMESPACE}
```

### 验证安装

确认 Dynamo 平台的 pod 正在运行，输出应类似如下：

```bash
kubectl get pods -n ${DYNAMO_NAMESPACE}
NAME                                                              READY   STATUS    RESTARTS   AGE
dynamo-platform-dynamo-operator-controller-manager-ff54b5dstgcq   1/1     Running   0          106s
dynamo-platform-nats-0                                            2/2     Running   0          106s
```

确认 Dynamo CRD 已安装：

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

## 部署 Dynamo DynamoGraphDeployment（DGD）

| Manifest | 描述 |
|----------|-------------|
| `manifests/vllm/disagg.yaml` | 解耦（disaggregated）的 prefill/decode DGD，使用基于 EFA 的 NIXL LIBFABRIC 后端。面向支持 GPUDirect RDMA 的 `g7e.12xlarge` 实例，用于在 prefill 与 decode worker 之间进行高吞吐 KV 缓存（KV cache）传输。 |
| `manifests/vllm/disagg-p5.yaml` | 解耦的 prefill/decode DGD，使用基于 EFA 的 NIXL LIBFABRIC 后端。面向 `p5.48xlarge` 预留实例，配备 8 个 EFA 设备（p5.48xlarge 上每 GPU 4 个 EFA）以及 Qwen3-32B 的 TP-2。在预留容量（`karpenter.sh/capacity-type: reserved`）上使用 2 decode + 6 prefill 副本。 |
| `manifests/vllm/disagg-tcp.yaml` | 替代版的解耦 prefill/decode 推理图（inference graph），使用 TCP 而非 EFA。面向 `g6e.2xlarge` 实例，适用于不支持 EFA 的实例类型。 |
| `manifests/vllm/agg.yaml` | 聚合（agg，单 worker）推理图，单个 vLLM worker 同时处理 prefill 与 decode。部署更简单，没有 KV 缓存传输的开销。 |


### 在 EFS 上缓存模型

在部署推理图之前，先把模型权重下载到共享 EFS 文件系统。每个 Dynamo recipe 都附带一个 `model-cache/model-download.yaml` Job manifest，可从 HuggingFace 下载模型。

把 recipe 中的下载 manifest 复制到本地 kustomize 目录中并 apply：

```bash
# Example: cache the Qwen3-32B model which we will be using later
cp ../../../recipes/qwen3-32b/model-cache/model-download.yaml manifests/model-download/model-download.yaml
kubectl kustomize manifests/model-download | kubectl -n ${DYNAMO_NAMESPACE} apply -f -
rm -f manifests/model-download/model-download.yaml
```

recipe manifest 没有为下载容器设置任何 memory 资源。如果不设置 memory request，Job pod 在下载过程中可能被 OOMKilled —— 大模型尤其如此。`manifests/model-download/` 中的 `kustomization.yaml` 会通过 patch 注入 memory request 来避免该问题，默认值为 `4Gi`。

对于更大的模型（如 DeepSeek-R1、Nemotron-3-Super-120B），请在 apply 之前调高 `manifests/model-download/kustomization.yaml` 中的该值：

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

随后 apply：

```bash
kubectl kustomize manifests/model-download | kubectl -n ${DYNAMO_NAMESPACE} apply -f -
```

监视下载 Job：

```bash
kubectl -n ${DYNAMO_NAMESPACE} get jobs model-download
kubectl -n ${DYNAMO_NAMESPACE} logs -f job/model-download
```

如需重新下载（例如更换模型或修复 OOM），先删除上一次的 Job：

```bash
kubectl -n ${DYNAMO_NAMESPACE} delete job model-download
```

然后复制新的 recipe manifest 重新 apply。

### 解耦式服务（Disaggregated Serving）

该示例部署一个解耦的 prefill/decode Dynamo 推理图，使用基于 [Elastic Fabric Adapter (EFA)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html) 的 NIXL LIBFABRIC 后端，在 worker 之间进行高吞吐 KV 缓存传输。

它面向支持 GPUDirect RDMA 的 `g7e.12xlarge` 实例，并使用 Dynamo 启用 EFA 的 vLLM 容器 `nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.1.1-efa-amd64`，该镜像已预装 [EFA Installer](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa-changelog.html)。

*注：完整的 EFA 支持实例类型清单见 [AWS EC2 文档](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html#efa-instance-types)。*

```yaml
        nodeSelector:
          node.kubernetes.io/instance-type: g7e.12xlarge
```

worker 间的 KV 缓存传输使用 [NIXL](https://github.com/ai-dynamo/nixl) 的 LIBFABRIC 后端。通过传入以下参数给 vLLM 启用：

`--kv-transfer-config '{"kv_connector":"NixlConnector","kv_role":"kv_both","kv_connector_extra_config": {"backends": ["LIBFABRIC"]}}'`

*注：在不支持 EFA 的实例类型上，NIXL 的 libfabric 后端会自动回退到 TCP。但 vLLM 的 `NixlConnector` 默认把 buffer device 设为 `cuda`，因此在没有 EFA 的解耦服务下你必须在 `kv-transfer-config` 中加上 `"kv_buffer_device":"cpu"` 才能正常工作。*

通过 `vpc.amazonaws.com/efa` 扩展资源为每个 worker pod 申请一个 EFA 设备：
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
*注：EKS Auto Mode 内置了 EFA 设备插件，使 `vpc.amazonaws.com/efa` 扩展资源可用。*

所有 worker（prefill 和 decode）必须位于同一个可用区，因为 EFA 流量不跨 AZ。可通过 pod affinity 强制满足该约束：

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
*注：`manifests/vllm/disagg-tcp.yaml` 提供使用 TCP（而非 EFA）的另一示例，面向 `g6e.2xlarge` 实例。*

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

应当看到类似如下输出：

```bash
{"id":"chatcmpl-23a7c94b-99cb-42ca-ae56-2397aa5a560f","choices":[{"index":0,"message":{"content":"<think>\nOkay, so I need to develop a character background for someone who's an intrepid explorer in Eldoria, specifically focusing on their motivations,","role":"assistant","reasoning_content":null},"finish_reason":"length"}],"created":1773336002,"model":"Qwen/Qwen3-0.6B","object":"chat.completion","usage":{"prompt_tokens":196,"completion_tokens":30,"total_tokens":226,"prompt_tokens_details":{"audio_tokens":null,"cached_tokens":192}},"nvext":{"worker_id":{"prefill_worker_id":4265733549773195,"prefill_dp_rank":0,"decode_worker_id":7535192362430132,"decode_dp_rank":0},"timing":{"request_received_ms":1773336002136,"prefill_wait_time_ms":0.852483,"prefill_time_ms":12.90597,"ttft_ms":13.758453000000001,"total_time_ms":110.89621500000001,"kv_hit_rate":0.0}}}
```

*注：每个 worker 的首个请求会有更高的延迟，因为存在 NIXL 后端握手与初始化开销，仅在第一次传输时发生。*

查看日志：
```bash
kubectl logs -n ${DYNAMO_NAMESPACE} -l nvidia.com/dynamo-graph-deployment-name=vllm-disagg --all-containers=true --max-log-requests=20 --prefix=true --timestamps -f
```

清理：

```bash
kubectl -n ${DYNAMO_NAMESPACE} delete -f manifests/vllm/disagg.yaml
```

### 聚合式服务（Aggregated Serving）

```bash
kubectl -n ${DYNAMO_NAMESPACE} apply -f manifests/vllm/agg.yaml
```

你的 pod 应当如下所示，状态为 "Running"：

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

应当看到类似如下输出：

```bash
{"id":"chatcmpl-093fac0e-f75e-43b5-90dc-96c8c77a2e7c","choices":[{"index":0,"message":{"content":"<think>\nOkay, I need to develop a character background for the explorer in Eldoria. Let me start by understanding the user's query. They mentioned","role":"assistant","reasoning_content":null},"finish_reason":"length"}],"created":1773443560,"model":"Qwen/Qwen3-0.6B","object":"chat.completion","usage":{"prompt_tokens":196,"completion_tokens":30,"total_tokens":226},"nvext":{"timing":{"request_received_ms":1773443560878,"total_time_ms":99.89782}}}%
```

清理：

```bash
kubectl -n ${DYNAMO_NAMESPACE} delete -f manifests/vllm/agg.yaml
```

## 在 ML 工作负载中使用 On-Demand Capacity Reservations（ODCR）与 Capacity Blocks（CB）

按需获取 GPU 实例可能比较困难。AWS 提供了两种容量预留机制来为 ML 工作负载保证可用资源：

- [On-Demand Capacity Reservations (ODCRs)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-reservations.html)：在指定可用区为任意时长预留容量。无论是否使用，都需要为预留容量付费。
- [Capacity Blocks for ML](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-capacity-blocks.html)：在固定时间窗口（数小时到数天）内预留 GPU 实例。实例位于 EC2 UltraClusters 中，具备低延迟网络。Capacity Block 有明确的结束时间，EC2 会在 block 到期前终止实例。

EKS Auto Mode 底层使用 Karpenter，它把预留容量建模为 `karpenter.sh/capacity-type: reserved`，并比 on-demand、spot 优先级更高。

> **注：** 默认情况下，EKS Auto Mode 可以自动启动 open ODCR，但不会优先使用它们；Capacity Blocks 永远不会被自动使用。要让两者被优先使用并被标记为 `reserved`，必须在 NodeClass 上显式配置 `capacityReservationSelectorTerms`。

### 创建带容量预留的 NodeClass

创建一个 NodeClass，引用你的 ODCR 或 Capacity Block 预留。可按预留 ID 选择，也可按 tag 选择。

首先，从 EKS Auto Mode 已经创建好的 `default` NodeClass 中提取子网、安全组与 role 配置：

```bash
export NC_SUBNETS=$(kubectl get nodeclass default -o json | jq -c '.spec.subnetSelectorTerms')
export NC_SG=$(kubectl get nodeclass default -o json | jq -c '.spec.securityGroupSelectorTerms')
export NC_ROLE=$(kubectl get nodeclass default -o json | jq -r '.spec.role')
```

把 `<CR ID>` 替换为你在 EC2 控制台中的实际预留 ID。
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

等到 capacityReservation 状态为 `active`：

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

### 为预留容量创建 NodePool

创建一个引用 `gpu-reserved` NodeClass、capacity type 使用 `reserved` 的 NodePool。可选地把 `on-demand` 与 `spot` 作为预留耗尽时的回退。

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

确认 `gpu-reserved` NodePool 处于就绪状态：

```bash
kubectl get nodepool gpu-reserved
NAME           NODECLASS      NODES   READY   AGE
gpu-reserved   gpu-reserved   0       True    8s
```

> **注：** 一旦在集群中任意 NodeClass 上配置了 `capacityReservationSelectorTerms`，EKS Auto Mode 就会停止为所有 NodeClass 自动使用 open ODCR。请确保所有应当使用 ODCR 的 NodeClass 都显式配置了 selector terms。

### 在工作负载中指向预留节点

通过 NodePool 现有的 requirements 与 taints，pod 会被调度到预留节点。如果你希望某个工作负载只跑在预留容量上，可加上 node selector：

```yaml
      nodeSelector:
        karpenter.sh/capacity-type: reserved
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
```

### Capacity Blocks 注意事项

Capacity Block 有固定的结束时间。EC2 会在 block 到期前 30 分钟开始终止实例（UltraServer 类型为 60 分钟）。Karpenter 会比 EC2 提前 10 分钟开始 drain 节点，以便你的工作负载有时间优雅退出。

请据此规划推理工作负载；如果你需要在 Capacity Block 时间窗外仍然连续运行，可考虑在 NodePool 中把 `on-demand` 作为回退 capacity type。

## 清理

删除所有 DynamoGraphDeployment：

```bash
kubectl -n ${DYNAMO_NAMESPACE} get dgd

# If you have any, delete them
kubectl -n ${DYNAMO_NAMESPACE} delete dgd <name>
```

卸载 Dynamo 平台：
```bash
helm uninstall -n ${DYNAMO_NAMESPACE} dynamo-platform
```

清理与 NATS 相关的遗留 PVC：
```bash
kubectl -n ${DYNAMO_NAMESPACE} get pvc
NAME                                             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
dynamo-platform-nats-js-dynamo-platform-nats-0   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   10Gi       RWO            auto-ebs-sc    <unset>                 75m

kubectl -n ${DYNAMO_NAMESPACE} delete pvc dynamo-platform-nats-js-dynamo-platform-nats-0
```

删除 AutoMode GPU nodepool：

```bash
kubectl delete nodepool gpu
```

清理 EFS 相关资源，请按 [EFS 安装指南的清理章节](EFS.md#cleanup) 操作。

使用 Eksctl 删除 EKS Auto Mode 集群：

```bash
eksctl delete cluster -f <(envsubst < templates/eksctl.yaml)
```
