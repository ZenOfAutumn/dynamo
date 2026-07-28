# [实验性] Kimi-K2.5 nvidia/Kimi-K2.5-NVFP4 —— Kubernetes 上的 TokenSpeed 聚合部署

> **仅文本：** Kimi-K2.5 的 NVFP4 导出版本不包含视觉塔（vision-tower）权重。
> TokenSpeed 加载器会为缺失的 `vision_tower.*` 参数记录告警，并继续以
> 仅文本的前向路径运行。图像输入会失败。

本配方在 Dynamo 的 KV 感知聚合前端下，在 **TokenSpeed** 引擎上运行
`nvidia/Kimi-K2.5-NVFP4`。它对齐了上游
[TokenSpeed 模型配方](https://lightseek.org/tokenspeed/recipes/models)
中针对 Kimi K2.5 / K2.6 的参数集合。

| 部署 | Manifest | 说明 | 硬件 |
|-----------|----------|-------------|----------|
| **聚合（TokenSpeed）** | [`deploy.yaml`](deploy.yaml) | 使用 TokenSpeed 引擎的 KV 感知聚合服务 | 1× 4×B200（TP=4, EP=4） |

> **注意：使用原生 Kubernetes 资源，而非 `DynamoGraphDeployment`。** Dynamo
> Operator 的 CRD 当前只允许 `backendFramework` 取值为 `vllm`、`sglang`、
> `trtllm`（详见 `deploy/operator/api/v1beta1/common.go`）。在 `tokenspeed` 被
> 加入该枚举之前，本配方将 operator 否则会生成的四个进程（etcd 发现 + NATS
> event-plane + 前端 HTTP router + worker 引擎）以普通 `Deployment` 与 `Service`
> 的形式部署。前端 Service 命名为 `kimi-k25-tokenspeed-agg-frontend`，与
> operator 生成的命名一致，使得在迁移到 `DynamoGraphDeployment` 前后，
> port-forward 指令保持稳定。`deploy.yaml` 中带有内联 TODO 标注切换点。
>
> **为什么使用 etcd 而非 `--discovery-backend kubernetes`**：K8s 原生发现客户端
> 需要 operator 注入的脚手架（downward-API env、`dynamoworkermetadatas.nvidia.com`
> CRD、对 `DynamoWorkerMetadata` + `EndpointSlices` 的 RBAC、带标签的 worker
> `Service`/`EndpointSlice`）。手动复制这些会破坏本配方"无需 operator"的定位，
> 因此我们将 etcd 保留为自包含的 sidecar；最终的 DGD 迁移会把这些都委托给
> operator。

## 镜像 —— 需要本地构建

写作本配方时，**没有公开的 `nvcr.io/nvidia/ai-dynamo/tokenspeed-runtime` 镜像**；
TokenSpeed 在 Dynamo 中的集成仍在进行中。本目录提供了一个 [`Dockerfile`](Dockerfile)，
基于 `docker.io/lightseekorg/tokenspeed-runner:cu130-torch-2.11.0`（上游
TokenSpeed 运行时基础镜像，已通过 tag 固定以保证可复现性）构建一个 Dynamo+TokenSpeed
镜像。

lightseek runner 镜像 **仅是运行时基础**——提供 CUDA 13、PyTorch 2.11 与 mooncake，
但不包含 TokenSpeed 引擎本身（该引擎尚未在 PyPI 发布；只存在用于占位的命名预留）。
本 Dockerfile 按官方
[lightseek.org/tokenspeed/guides/getting-started](https://lightseek.org/tokenspeed/guides/getting-started)
指南从上游源码安装 TokenSpeed，然后在其上叠加 Dynamo。

执行构建，将结果推送到集群可拉取的镜像仓库，然后更新
[`deploy.yaml`](deploy.yaml) 中的 `image:` 字段。

### 1. 构建 Dynamo+TokenSpeed 镜像

构建上下文必须是 **Dynamo 仓库根目录**（Dockerfile 通过 `COPY` 复制源码树，
再用 `maturin` 构建 Dynamo Python wheel）。

```bash
# From the repo root.
docker build \
  -f recipes/kimi-k2.5/tokenspeed/agg/nvidia/Dockerfile \
  --target dev \
  -t <your-registry>/dynamo-tokenspeed:dev \
  .
```

Dockerfile 默认 `BASE_IMAGE` 指向已固定的公共 TokenSpeed 运行时 tag：

```
docker.io/lightseekorg/tokenspeed-runner:cu130-torch-2.11.0
```

该 tag 基于 CUDA 13（匹配 B200 的 CUDA 13.x 驱动），并包含 PyTorch 2.11.0。
同仓库的同级 tag 包括：`cu130`、`cu129-torch-2.11.0`、`cu129`、`latest`。如需
针对其他 runtime，覆盖构建参数：

```bash
--build-arg BASE_IMAGE=docker.io/lightseekorg/tokenspeed-runner:cu129-torch-2.11.0
```

Dockerfile 有两个可达 target：

- `--target runtime`：精简镜像，无编译器。适合面向生产的部署。
- `--target dev`：保留 `cargo`、`rustc`、`maturin`、build-essential、git 等。在
  pod 内迭代 Rust 绑定时（内联 `maturin develop`）很有用。

对于 arm64 基础镜像，加上 `--build-arg ARCH=arm64`。

### 2. 推送到集群可访问的镜像仓库

```bash
docker push <your-registry>/dynamo-tokenspeed:dev
```

### 3. 更新 `deploy.yaml` 中两处 `image:` 字段

[`deploy.yaml`](deploy.yaml) 中的 `Frontend` 与 `TokenSpeedWorker` 服务都引用了
占位符 `<your-registry>/dynamo-tokenspeed:dev`。请用刚刚推送的 tag 替换它们。

## 前置条件

- 一个 Kubernetes 集群（本配方 **不** 需要 Dynamo Operator——见上文说明）
- 4× B200 GPU（如果将 `--tensor-parallel-size` 提升到 8，则需要 8× B200）
- 一个包含你 Hugging Face token 的 `hf-token-secret` Secret
- 命名空间内的 `nvcrimagepullsecret` Secret，或将 [`deploy.yaml`](deploy.yaml) 中
  的 `imagePullSecrets` 修改为你集群的 pull-secret 命名
- 已有的 `model-cache` PVC，其中已下载 `nvidia/Kimi-K2.5-NVFP4`（参见
  [`model-cache/nvidia/`](../../../model-cache/nvidia/) 同级目录的 job）
- 集群节点可拉取的镜像仓库，并已推送 `dynamo-tokenspeed` 镜像（见上面的构建步骤）

## 部署

```bash
export NAMESPACE=dynamo-demo

# Download model weights into the model-cache PVC
kubectl apply -f ../../../model-cache/model-cache.yaml -n ${NAMESPACE}
kubectl apply -f ../../../model-cache/nvidia/model-download.yaml -n ${NAMESPACE}
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE} --timeout=6000s

# After updating both image: fields in deploy.yaml, apply.
kubectl apply -f deploy.yaml -n ${NAMESPACE}
```

这会创建 4 个 Deployment + 3 个 Service：

| 资源 | 用途 | 镜像 |
|---|---|---|
| `kimi-k25-tokenspeed-etcd`（Deployment + Service） | 发现后端（discovery backend） | `gcr.io/etcd-development/etcd:v3.6.7` |
| `kimi-k25-tokenspeed-nats`（Deployment + Service） | 事件平面（JetStream） | `nats:2.12.4` |
| `kimi-k25-tokenspeed-frontend`（Deployment） | 8000 端口上以 KV-router 模式运行的 `dynamo.frontend` | 你本地构建的 `dynamo-tokenspeed` |
| `kimi-k25-tokenspeed-agg-frontend`（Service） | 用于 port-forward 的稳定名称；选中 frontend Deployment | — |
| `kimi-k25-tokenspeed-worker`（Deployment） | `dynamo.tokenspeed`，运行 `nvidia/Kimi-K2.5-NVFP4`，TP=4 + EP=4，NVFP4 权重，FP8 KV 缓存，通过 `trtllm_mla` 实现 MLA attention，通过 `flashinfer_trtllm` 实现 MoE，并使用 `kimi_k25` reasoning + `kimi_k2` tool-call 解析器 | 你本地构建的 `dynamo-tokenspeed` |

发现使用 etcd Service（`--discovery-backend etcd`，`ETCD_ENDPOINTS=http://kimi-k25-tokenspeed-etcd:2379`）
在前端与 worker 上同时启用；事件平面使用 nats Service
（`NATS_SERVER=nats://kimi-k25-tokenspeed-nats:4222`）。

## 测试部署

```bash
kubectl port-forward svc/kimi-k25-tokenspeed-agg-frontend 8000:8000 -n ${NAMESPACE}
```

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "kimi-k2.5",
    "messages": [{"role": "user", "content": "Hello!"}],
    "max_tokens": 300,
    "skip_special_tokens": false,
    "chat_template_kwargs": {"thinking": true}
  }'
```

Kimi K2.5 会输出 `<think>...</think>` 推理前缀；`kimi_k25` reasoning 解析器会
将它拆分到响应的 `reasoning_content` 字段中，最终答案放在 `content` 中。
请发送 `max_tokens >= 200`，为两个阶段都留出空间。

两个额外请求字段都是必需的：

- `chat_template_kwargs: {thinking: true}` 告知 chat template 输出 `<think>` 起始
  标记，引导模型在答案之前生成 reasoning 块。没有它，模板会跳过 think 前缀，
  模型解码出的第一个 token 会改变。（`thinking` 不是 OpenAI 的顶层字段，因此
  必须放进 `chat_template_kwargs` 里——Dynamo 的前端会以
  `400 Unsupported parameter(s): thinking` 拒绝未知顶层字段。）
- `skip_special_tokens: false` 在流式输出中保留结构性 token（`<|im_end|>`、
  `</think>`），使 `kimi_k25` reasoning 解析器能找到 reasoning/content 边界。
  默认 `true` 时，解析器看到的是被剥离的输出，无法切分段落，会返回
  `content: null` 与少量结尾停止 token。

这两个开关共同恢复了引擎侧的契约——上游 `tokenspeed serve` 单体应用因为同时
拥有 chat template 与解析器而隐式具备这一契约；Dynamo 的拆分（前端渲染
template、worker 流式输出、解析器在后运行）造成了一个需要用户显式打通的接缝。

## 参数选择说明

- `--tensor-parallel-size 4 --enable-expert-parallel`：与上游 TokenSpeed 配方一致；
  attention 以 TP=4 切分，MoE 专家以 EP=4 在同样的 4 个 rank 上切分。如果你有
  8×B200 节点并希望以更高的单 GPU 内存压力换取更低延迟，可提升到 8。
- `--quantization nvfp4 --attention-backend trtllm_mla --moe-backend flashinfer_trtllm`：
  Blackwell 原生快路径。这三者一起是本配方面向 B200 的原因。
- `--kv-cache-dtype fp8`：与 `--quantization nvfp4` 配合，相对 BF16 KV 减半 KV 缓存
  占用，对 chat 工作负载没有可测量的精度损失。
- `--gpu-memory-utilization 0.80`：低于 TokenSpeed 的默认值 0.85。Dynamo worker
  层（etcd watcher、NATS event-plane 订阅者以及 TCP request-plane listener）在
  引擎之外还会占用额外内存，默认余量对 K2.5 的 MoE 权重初始化过紧。0.75 又
  过低——引擎计算出的 KV 缓存池为负值。
- `--dyn-reasoning-parser kimi_k25 --dyn-tool-call-parser kimi_k2`：上游 TokenSpeed
  配方使用 `--reasoning-parser kimi_k2 --tool-call-parser kimi_k2`，它们应用在
  引擎层。Dynamo 在 worker 层用 `--dyn-*` 变体进行拦截；`kimi_k25` 是面向 K2.5/K2.6
  的解析器。
- `--max-model-len 262144`：K2.5 支持 256K 上下文。该长度下的 KV 缓存预算很大；
  如果你的 prompt 较短，可减小该值以释放 GPU 内存用于批次。

## 与 TRT-LLM 同级配方的差异

[`../../trtllm/agg/nvidia/`](../../trtllm/agg/nvidia/) 部署使用 TP=8 与基于
ConfigMap 的引擎 YAML，并以单一 `DynamoGraphDeployment` 形式提供，让 Dynamo
Operator 生成底层的 `Deployment`/`Service`。本配方在三个方面有所不同：

1. **引擎参数为内联 CLI 参数，而非 ConfigMap。** TokenSpeed 直接接受在 worker
   命令行上传入的引擎参数，与 vLLM 类似。因此布局更接近
   [`recipes/llama-3-70b/vllm/agg/`](../../../../llama-3-70b/vllm/agg/) 而非 TRT-LLM
   的 ConfigMap 模式。
2. **使用原生 `Deployment` + `Service`，而非 `DynamoGraphDeployment`。** operator 的
   `backendFramework` 枚举当前仅校验 `vllm`、`sglang`、`trtllm`——`tokenspeed` 会
   被准入校验拒绝。在 operator 加入 `tokenspeed` 支持之前，本配方以普通
   `Deployment` 形式列出 operator 否则会生成的四个进程。详见 `deploy.yaml` 内
   联 TODO——一旦 operator 后端落地，deploy.yaml 即可塌缩为与 TRT-LLM 同级配方
   形态一致的单一 `DynamoGraphDeployment`。
3. **本地镜像构建。** TRT-LLM 配方基于公共
   `nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime` 镜像；在 NVIDIA 发布对应镜像之前，
   本配方需要你自行构建并推送 Dynamo+TokenSpeed 镜像（见上面的构建步骤）。
