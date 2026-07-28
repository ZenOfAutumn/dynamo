---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Amazon Elastic Container Service (ECS)
---

# 在 AWS ECS 上部署 Dynamo vLLM 示例
## 1. EC2 集群搭建（用于 vLLM 工作负载）
1. 进入 AWS ECS 控制台 **Clusters** 标签页，点击 **Create cluster**，名称填 `dynamo-GPU`。
2. 输入集群名，**基础设施**选择 **AWS EC2 instances**。该选项会创建一个使用 EC2 实例来部署容器的集群。
3. 选择 ECS 优化的 GPU AMI `Amazon Linux 2 (GPU)`（Amazon ECS–optimized），其开箱即用地包含 NVIDIA 驱动和 Docker GPU 运行时。
4. **EC2 实例类型**选 `g6e.2xlarge` 并添加一个 `SSH Key pair`，方便登录实例做调试。要测试解耦服务，至少需要 2 张 GPU，因此可以选用带 4 张 GPU 的 `g6e.12xlarge`。
5. 把 **Root EBS volume size** 设置为 `200`。
6. 网络部分使用默认设置。请确保 **security group** 配置了：
- 一条 inbound 规则允许来自该 security group 的 "All traffic"。
- 22 与 8000 端口的 inbound 规则，便于 ssh 登录调试。
7. 把 **Auto-assign public IP** 设为 `Turn on`。
8. 点击 **Create**，集群将通过 CloudFormation 部署。

## 2. Fargate 集群搭建（用于 ETCD/NATS 服务）
1. 进入 AWS ECS 控制台 **Clusters** 标签页，点击 **Create cluster**。
2. 集群名填 `dynamo-fargate`。
3. **基础设施**选择 **AWS Fargate (serverless)**。
4. 网络方面使用与 EC2 集群相同的 VPC 和 subnet，以确保各服务间可连通。
5. security group 与 EC2 集群保持一致。这样所有服务之间就自动互通了。
6. 确保出站规则允许所有流量（默认即可），让 Fargate 任务能拉取镜像并对外通信。
7. 点击 **Create** 部署 Fargate 集群。

## 3. ETCD/NATS Task Definition 配置
为运行在 Fargate 上的 ETCD 与 NATS 服务添加任务。附带一份示例任务定义 JSON。

### 3.1 创建 ecsTaskExecutionRole（必需）
在创建任务定义之前，需要先创建 `ecsTaskExecutionRole` IAM 角色。该角色允许 ECS 代你从镜像仓库拉镜像并把日志写入 CloudWatch。

> [!IMPORTANT]
> 如果通过 AWS Console 的分步向导创建任务定义，该角色会自动创建。但当你像本指南推荐的那样从 JSON 导入任务定义时，必须手动创建。

按照 [AWS 关于创建 task execution IAM role 的文档](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html#create-task-execution-role) 创建一个名为 `ecsTaskExecutionRole`、附带 `AmazonECSTaskExecutionRolePolicy` 策略的角色。

根据任务定义的需要，可能还要给 `ecsTaskExecutionRole` 加上 Amazon CloudWatch 权限和 AWS Secrets Manager 权限。详情见 [Amazon CloudWatch Logs permissions reference](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/permissions-reference-cwl.html) 与 [AWS Secrets Manager authentication and access control guide](https://docs.aws.amazon.com/secretsmanager/latest/userguide/auth-and-access.html#auth-and-access_secrets)。

> [!NOTE]
> 该角色的 ARN 形如 `arn:aws:iam::<your-account-id>:role/ecsTaskExecutionRole`。请把任意任务定义 JSON 中的 `<your-account-id>` 替换为你的实际 AWS 账户 ID。

### 3.2 任务定义配置
1. ETCD 容器
- 容器名使用 `etcd`
- 镜像 URL 是 `bitnamilegacy/etcd`，**Essential container** 选 **Yes**
- 容器端口

|Container port|Protocol|Port name| App protocol|
|-|-|-|-|
|2379|TCP|2379|HTTP|
|2380|TCP|2380|HTTP|

- 环境变量 key 为 `ALLOW_NONE_AUTHENTICATION`，值为 `YES`
2. NATS 容器
- 容器名使用 `nats`
- 镜像 URL 是 `nats`，**Essential container** 选 **Yes**
- 容器端口

|Container port|Protocol|Port name| App protocol|
|-|-|-|-|
|4222|TCP|4222|HTTP|
|6222|TCP|6222|HTTP|
|8222|TCP|8222|HTTP|

- Docker 配置：在 **Command** 中加入 `-js, --trace`

## 4. vLLM 任务定义配置
> [!NOTE]
> 创建以下任务定义之前，请先按 3.1 节创建好 `ecsTaskExecutionRole`。

1. Dynamo vLLM 前端 + 解码 worker 任务
该任务会创建 vLLM 前端、processor、router 以及一个解码 worker。
按以下步骤创建：
- 容器名设为 `dynamo-frontend`，使用预构建的 [Dynamo 容器](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/vllm-runtime)。
- **Launch type** 选 `Amazon EC2 instances`，**Task size** 设置为 `2 vCPU` 与 `40 GB` 内存。
- **Network mode** 选 `host`。
- 容器名使用 `dynamo-vLLM-frontend`。
- 填写 Image URL（可使用预构建的 [Dynamo 容器](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/vllm-runtime)），**Essential container** 选 **Yes**。可以是 AWS ECR URL 或 Nvidia NGC URL。如果使用 NGC URL，请同时勾选 **Private registry authentication** 并填入你的 Secret Manager ARN 或名称。
- 容器端口

|Container port|Protocol|Port name| App protocol|
|-|-|-|-|
|8000|TCP|8000|HTTP|
- **Resource allocation limits** 使用 `1` GPU
- 环境变量配置如下，稍后会覆盖 `IP_ADDRESS`：

|Key|Value type|Value|
|-|-|-|
|ETCD_ENDPOINTS|Value|http://IP_ADDRESS:2379|
|NATS_SERVER|Value|nats://IP_ADDRESS:4222|
- Docker 配置
**Entry point** 填 `sh,-c`，**Command** 填 `cd examples/backends/vllm && python -m dynamo.frontend --router-mode kv & python3 -m dynamo.vllm --model Qwen/Qwen3-0.6B --enforce-eager`

2. Dynamo vLLM PrefillWorker 任务
按与前端 worker 相同的步骤创建 PrefillWorker 任务，但需做以下改动：
- 容器名设为 `dynamo-prefill`
- 不做容器端口映射
- Docker 配置中的 command 改为 `cd examples/backends/vllm && python3 -m dynamo.vllm --model Qwen/Qwen3-0.6B --enforce-eager --disaggregation-mode prefill`

## 5. 任务部署
你可以创建一个 service，或直接从任务定义运行任务：
1. ETCD/NATS 任务
- **Existing cluster** 选第 2 步创建的 Fargate 集群（`dynamo-fargate`）。
- **Launch type** 选 `FARGATE`。
- **Networking** 选与 EC2 集群相同的 VPC 与 subnet。
- **Security group** 选与 EC2 集群相同的 security group。
- 确认出站规则允许所有流量，便于拉取镜像和对外通信。
- 等待部署完成，记录该任务的 **Private IP**。
2. Dynamo 前端 + 解码 worker 任务
- **Existing cluster** 选第 1 步创建的 EC2 集群（`dynamo-GPU`）。
- 在 **Container Overrides** 中，把 ETCD/NATS 任务的 IP 填入 `ETCD_ENDPOINTS` 和 `NATS_SERVER`。
- 部署完成后，会创建一个聚合服务的端点，可用第 6 节的脚本测试。
3. Dynamo PrefillWorker 任务
- 对于解耦服务，可以在另一张 GPU 上单独部署 prefill worker。**Existing cluster** 选第 1 步创建的 EC2 集群（`dynamo-GPU`），并要求至少 2 张 GPU（例如 `g6e.12xlarge`）。
- 在 **Container Overrides** 中，把 ETCD/NATS 任务的 IP 填入 `ETCD_ENDPOINTS` 和 `NATS_SERVER`。

## 6. 测试
在任务页面找到 dynamo 前端任务的公网 IP。运行以下命令访问端点。
```sh
export DYNAMO_IP_ADDRESS=TASK_PUBLIC_IP_ADDRESS
curl http://$DYNAMO_IP_ADDRESS:8000/v1/models
curl http://$DYNAMO_IP_ADDRESS:8000/v1/chat/completions   -H "Content-Type: application/json"   -d '{
    "model": "Qwen/Qwen3-0.6B",
    "messages": [
    {
        "role": "user",
        "content": "In the heart of Eldoria, an ancient land of boundless magic and mysterious creatures, lies the long-forgotten city of Aeloria. Once a beacon of knowledge and power, Aeloria was buried beneath the shifting sands of time, lost to the world for centuries. You are an intrepid explorer, known for your unparalleled curiosity and courage, who has stumbled upon an ancient map hinting at ests that Aeloria holds a secret so profound that it has the potential to reshape the very fabric of reality. Your journey will take you through treacherous deserts, enchanted forests, and across perilous mountain ranges. Your Task: Character Background: Develop a detailed background for your character. Describe their motivations for seeking out Aeloria, their skills and weaknesses, and any personal connections to the ancient city or its legends. Are they driven by a quest for knowledge, a search for lost familt clue is hidden."
    }
    ],
    "stream":false,
    "max_tokens": 30
  }'
```
你应该可以看到来自托管端点的响应。
