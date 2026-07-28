# 在 AWS ECS 上部署 Dynamo + vLLM 示例
## 1. EC2 集群搭建（用于 vLLM 工作负载）
1. 进入 AWS ECS 控制台 **Clusters** 标签页，点击 **Create cluster**，名称设为 `dynamo-GPU`
2. 输入集群名称，并选择 **AWS EC2 instances** 作为基础设施。该选项会创建一个使用 EC2 实例来部署容器的集群（cluster）。
3. 选择 ECS 优化的 GPU AMI `Amazon Linux 2 (GPU)`（Amazon ECS–optimized），它默认包含 NVIDIA 驱动和 Docker GPU runtime。
4. 选择 `g6e.2xlarge` 作为 **EC2 instance type**，并添加一个 `SSH Key pair` 以便登录实例进行调试。如果要测试解耦式（disaggregated）服务，至少需要 2 块 GPU，因此可选择带 4 GPU 的 `g6e.12xlarge`
5. 将 **Root EBS volume size** 设为 `200`
6. 网络部分使用默认设置。请确保 **security group** 包含
- 一条入站规则，允许该安全组内的 “All traffic”
- 端口 22 与 8000 的入站规则，便于 ssh 登录实例进行调试
7. 将 **Auto-assign public IP** 选项设为 `Turn on`。
8. 点击 **Create**，集群将通过 CloudFormation 进行部署。

## 2. Fargate 集群搭建（用于 ETCD/NATS 服务）
1. 进入 AWS ECS 控制台 **Clusters** 标签页，点击 **Create cluster**
2. 集群名称填为 `dynamo-fargate`
3. 选择 **AWS Fargate (serverless)** 作为基础设施
4. 网络部分，使用与 EC2 集群相同的 VPC 与子网，确保服务之间互通
5. 安全组使用与 EC2 集群相同的安全组。这会自动允许各服务之间的通信。
6. 确保出站规则允许所有流量（默认设置即可），以便 Fargate 任务能下载容器镜像并对外通信
7. 点击 **Create**，部署 Fargate 集群

## 3. ETCD/NATS 任务定义搭建
为 ETCD 与 NATS 服务在 Fargate 上添加任务。下面附有任务定义 JSON 示例。

### 3.1 创建 ecsTaskExecutionRole（必需）
在创建任务定义之前，需要先创建 `ecsTaskExecutionRole` IAM 角色。该角色用于让 ECS 代表你从仓库拉取容器镜像、向 CloudWatch 写日志。

> [!IMPORTANT]
> 如果你通过 AWS 控制台的分步向导创建任务定义，该角色会自动创建。然而，如果你按本文档建议从 JSON 导入任务定义，则必须手动创建该角色。

请按 [AWS 创建任务执行 IAM 角色文档](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html#create-task-execution-role) 创建一个名为 `ecsTaskExecutionRole`、附带 `AmazonECSTaskExecutionRolePolicy` 策略的角色。

根据任务定义不同，你可能还需要给 `ecsTaskExecutionRole` 添加 Amazon CloudWatch 与 AWS Secrets Manager 的权限。详见 [Amazon CloudWatch Logs 权限参考](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/permissions-reference-cwl.html) 和 [AWS Secrets Manager 鉴权与访问控制指南](https://docs.aws.amazon.com/secretsmanager/latest/userguide/auth-and-access.html#auth-and-access_secrets)

> [!NOTE]
> 该角色的 ARN 形如 `arn:aws:iam::<your-account-id>:role/ecsTaskExecutionRole`。请务必把任务定义 JSON 中的 `<your-account-id>` 替换为你实际的 AWS account ID。

### 3.2 任务定义配置
1. ETCD 容器
- 容器名设为 `etcd`
- 镜像 URL 为 `bitnamilegacy/etcd`，Essential container 选 **Yes**
- 容器端口

|Container port|Protocol|Port name| App protocol|
|-|-|-|-|
|2379|TCP|2379|HTTP|
|2380|TCP|2380|HTTP|
- 环境变量 key 为 `ALLOW_NONE_AUTHENTICATION`，value 为 `YES`
2. NATS 容器
- 容器名设为 `nats`
- 镜像 URL 为 `nats`，Essential container 选 **Yes**
- 容器端口

|Container port|Protocol|Port name| App protocol|
|-|-|-|-|
|4222|TCP|4222|HTTP|
|6222|TCP|6222|HTTP|
|8222|TCP|8222|HTTP|
- Docker 配置：在 **Command** 中添加 `-js, --trace`

## 4. vLLM 任务定义搭建
> [!Note]
> 在创建以下任务定义之前，请按 3.1 节先创建好 `ecsTaskExecutionRole`。

1. Dynamo vLLM 前端（Frontend）与 Decoding Worker 任务
该任务会创建 vLLM 前端、processor、router 以及一个 decoding worker。
按下列步骤创建该任务：
- 容器名设为 `dynamo-frontend`，使用预构建的 [Dynamo container](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/vllm-runtime)。
- **Launch type** 选择 `Amazon EC2 instances`，**Task size** 设为 `2 vCPU` 与 `40 GB` 内存
- Network mode 选择 `host`。
- 容器名设为 `dynamo-vLLM-frontend`
- 添加镜像 URL（可使用预构建的 [Dynamo container](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/ai-dynamo/containers/vllm-runtime)），Essential container 选 **Yes**。可以是 AWS ECR URL 或 Nvidia NGC URL。如果使用 NGC URL，请同时勾选 **Private registry authentication** 并填入你的 Secret Manager ARN 或名称。
- 容器端口

|Container port|Protocol|Port name| App protocol|
|-|-|-|-|
|8000|TCP|8000|HTTP|
- **Resource allocation limits** 中使用 `1` 块 GPU
- 环境变量按下表设置。稍后会覆盖 `IP_ADDRESS`。

|Key|Value type|Value|
|-|-|-|
|ETCD_ENDPOINTS|Value|http://IP_ADDRESS:2379|
|NATS_SERVER|Value|nats://IP_ADDRESS:4222|
- Docker 配置
**Entry point** 中填 `sh,-c`，**Command** 中填 `cd examples/backends/vllm && python -m dynamo.frontend --router-mode kv & python3 -m dynamo.vllm --model Qwen/Qwen3-0.6B --enforce-eager`

2. Dynamo vLLM PrefillWorker 任务
PrefillWorker 任务的创建方式与前端任务相同，但有以下差异：
- 容器名设为 `dynamo-prefill`
- 不映射容器端口
- Docker 配置中的命令为 `cd examples/backends/vllm && python3 -m dynamo.vllm --model Qwen/Qwen3-0.6B --enforce-eager --disaggregation-mode prefill`

## 5. 任务部署
你可以创建 service，也可以直接基于任务定义运行任务
1. ETCD/NATS 任务
- **Existing cluster** 选择第 2 步创建的 Fargate 集群（`dynamo-fargate`）
- **Launch type** 选 `FARGATE`
- 在 **Networking** 中选择与 EC2 集群相同的 VPC 与子网
- **Security group** 选择与 EC2 集群相同的安全组
- 确认出站规则允许所有流量，以便下载镜像并进行对外通信
- 等待该部署完成，并获取该任务的 **Private IP**。
2. Dynamo Frontend 与 Decoding Worker 任务
- **Existing cluster** 选择第 1 步创建的 EC2 集群（`dynamo-GPU`）
- 在 **Container Overrides** 中，把 `ETCD_ENDPOINTS` 和 `NATS_SERVER` 的值设为 ETCD/NATS 任务的 IP。
- 部署完成后，会建立一个聚合式（aggregated）服务端点，可以用第 6 步中的脚本测试。
3. Dynamo PrefillWorker 任务
- 对于解耦式服务，可以在另一块 GPU 上单独部署一个 prefill worker。**Existing cluster** 选择第 1 步创建的、至少包含 2 块 GPU 的 EC2 集群（例如 `g6e.12xlarge`）
- 在 **Container Overrides** 中，把 `ETCD_ENDPOINTS` 和 `NATS_SERVER` 的值设为 ETCD/NATS 任务的 IP。

## 6. 测试
在任务页面找到 dynamo frontend 任务的 public IP，运行以下命令测试该端点：
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
你应该可以从托管的端点看到响应。
