---
# SPDX-FileCopyrightText: Copyright (c) 2025-2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: Apache-2.0
title: Webhooks
---

本文介绍 Dynamo Operator 中的 webhook 功能，包括校验 webhook、证书管理与
故障排查。

## 概述

Dynamo Operator 使用 **Kubernetes admission webhook** 对自定义资源进行实时
校验与变更（mutation）。当前 operator 实现了 **校验型 webhook（validation
webhook）**，可在 API server 层面立即拒绝无效配置，相比基于 controller 的
校验，能更快地把反馈传给用户。

所有 webhook 类型（validating、mutating、conversion 等）共享同一个
**webhook server** 与 **TLS 证书基础设施**，使得证书管理在所有 webhook 操作
之间保持一致。

### 关键特性

- ✅ **始终启用** —— webhook 是 operator 的必需组件
- ✅ **共享证书基础设施** —— 所有 webhook 类型使用同一套 TLS 证书
- ✅ **证书自动生成与轮换** —— 内置 cert-controller，无需手动管理
- ✅ **cert-manager 集成** —— 可选集成，用于自定义 PKI 或符合组织证书策略
- ✅ **不可变字段强制** —— 通过 CEL 校验规则保护关键字段

### 当前的 Webhook 类型

- **Validating Webhook**：在持久化前对 CR 规范进行校验
  - `DynamoComponentDeployment` 校验
  - `DynamoGraphDeployment` 校验
  - `DynamoModel` 校验
  - `DynamoGraphDeploymentRequest` 校验
- **Mutating Webhook**：在创建时对资源应用默认值
  - `DynamoGraphDeployment` defaulting

**注意：** 所有 webhook 类型都使用本文所述的同一套证书基础设施。

---

## 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         API Server                               │
│  1. User submits CR (kubectl apply)                             │
│  2. API server calls MutatingWebhookConfiguration               │
└────────────────────────┬────────────────────────────────────────┘
                         │ HTTPS (TLS required)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Webhook Server (in Operator Pod)                │
│  3. Applies defaults (e.g., operator version annotation)        │
│  4. Returns mutated CR                                          │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                         API Server                               │
│  5. API server calls ValidatingWebhookConfiguration             │
└────────────────────────┬────────────────────────────────────────┘
                         │ HTTPS (TLS required)
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Webhook Server (in Operator Pod)                │
│  6. Validates CR against business rules                         │
│  7. Returns admit/deny decision + warnings                      │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                      API Server                                  │
│  8. If admitted: Persist CR to etcd                             │
│  9. If denied: Return error to user                             │
└─────────────────────────────────────────────────────────────────┘
```

### 准入流程（Admission Flow）

1. **Mutating webhook**：在校验之前应用默认值与变换
2. **Validating webhook**：对（可能已被变更过的）CR 按业务规则进行校验
3. **CEL 校验**：Kubernetes 原生的不可变性检查（始终生效）

---

## 从启用了 `webhook.enabled: false` 的版本升级

`webhook.enabled` 这个 Helm 值已被移除。webhook 现在是 operator 的必需组件并
始终激活。如果你之前以 `webhook.enabled: false` 运行，请在升级前完成以下步骤：

1. **从所有自定义 values 文件中移除 `webhook.enabled`**。Helm 会忽略未知的 key，
   但仍建议清理掉以免混淆。
2. **确保从 Kubernetes API server 到 operator pod 上的 9443 端口可达**。如果你
   配置了 `NetworkPolicy` 规则或防火墙限制流量，请添加一条入向规则，允许 API
   server 通过 9443 访问 webhook server。
3. **确保 webhook 的 TLS 证书可用**。默认情况下，operator 内置的
   cert-controller 会在启动时自动生成并轮换自签名证书 —— 无需任何动作。
   如果你使用 cert-manager 或外部托管的证书，升级前请确认相应配置已就绪。

---

## 配置

### 证书管理选项

operator 支持三种证书管理模式：

| 模式 | 说明 | 适用场景 |
|------|-------------|----------|
| **自动（默认）** | operator 内置的 cert-controller 自动生成并轮换证书 | 所有环境（推荐） |
| **cert-manager** | 集成 cert-manager 进行证书生命周期管理 | 已部署 cert-manager 且具有自定义 PKI 要求的集群 |
| **External** | 自带证书 | 由外部 PKI 托管证书的环境 |

---

### 高级配置

#### 完整配置参考

```yaml
dynamo-operator:
  webhook:
    # Certificate management (optional, to use cert-manager instead of built-in)
    certManager:
      enabled: false
      issuerRef:
        kind: Issuer
        name: selfsigned-issuer

    # Certificate secret configuration
    certificateSecret:
      name: webhook-server-cert
      external: false           # Set to true for externally managed certificates

    # Webhook behavior configuration
    failurePolicy: Fail        # Fail (reject on error) or Ignore (allow on error)
    timeoutSeconds: 10         # Webhook timeout

    # Namespace filtering (advanced)
    namespaceSelector: {}      # Kubernetes label selector for namespaces
```

#### 失败策略（Failure Policy）

```yaml
# Fail: Reject resources if webhook is unavailable (recommended for production)
webhook:
  failurePolicy: Fail

# Ignore: Allow resources if webhook is unavailable (use with caution)
webhook:
  failurePolicy: Ignore
```

**建议：** 生产环境使用 `Fail`，以确保校验始终生效。仅在你需要高可用并能容忍
偶发的非法资源时才使用 `Ignore`。

#### 命名空间过滤

控制哪些命名空间会被校验（仅适用于**集群范围 operator**）：

```yaml
# Only validate resources in namespaces with specific labels
webhook:
  namespaceSelector:
    matchLabels:
      dynamo-validation: enabled

# Or exclude specific namespaces
webhook:
  namespaceSelector:
    matchExpressions:
    - key: dynamo-validation
      operator: NotIn
      values: ["disabled"]
```

**注意：** 对于**命名空间受限模式（namespace-restricted）的 operator**
（已废弃），namespace selector 会被自动设置为只校验该 operator 自己的命名空间。
此模式下该配置项会被忽略。

---

## 证书管理

### 自动证书（默认）

**零配置！** operator 内置的 cert-controller 会在启动时自动生成并轮换证书。

#### 工作原理

1. **operator 启动**：`CertManager` 检查现有的证书 Secret（由
   `webhook.certificateSecret.name` 配置，默认 `webhook-server-cert`）。如果
   缺失或无效，则生成自签名 Root CA 与服务器证书并写入该 Secret。

2. **CA bundle 注入**：`CABundleInjector` 从 Secret 中读取 `ca.crt`，并把
   base64 编码后的 CA bundle 写入 `ValidatingWebhookConfiguration` 与
   `MutatingWebhookConfiguration`。

3. **证书轮换**：cert-controller 监控证书有效期，并在过期前重新生成证书。

4. **webhook server 启动**：webhook server 仅在确认证书就绪后才开始服务，
   避免启动竞态。

#### 证书有效期

- **Root CA**：10 年
- **服务器证书**：10 年（与 Root CA 相同）
- **自动轮换**：cert-controller 监控有效期，并在到期前重新生成

#### 智能证书管理

cert-controller 在证书生命周期上比较智能：
- ✅ 启动时**先检查现有证书** 再决定是否生成新证书
- ✅ 如果 Secret 中已有有效证书，则**跳过生成**
- ✅ 仅在必要时**才重新生成**（缺失、即将过期或 SAN 不正确）

这样带来的好处：
- operator 启动更快（无谓的证书生成被跳过）
- 不依赖 Helm hook 或外部 Job
- 证书可在 pod 重启间持久保留（保存在 Secret 中）

#### 手动证书轮换

如果你需要手动轮换证书：

```bash
# 删除证书 secret —— operator 会在重启时重新生成
kubectl delete secret <release>-webhook-server-cert -n <namespace>

# 重启 operator pod 触发重新生成
kubectl rollout restart deployment/<release>-dynamo-operator -n <namespace>
```

---

### cert-manager 集成

对于已经安装了 cert-manager 的集群，可以启用自动化的证书生命周期管理。

#### 前置条件

1. **已安装 cert-manager**（v1.0+）
2. **已配置 CA issuer**（例如 `selfsigned-issuer`）

#### 配置

```yaml
dynamo-operator:
  webhook:
    certManager:
      enabled: true
      issuerRef:
        kind: Issuer              # Or ClusterIssuer
        name: selfsigned-issuer   # Your issuer name
```

#### 工作原理

1. **Helm 创建 Certificate 资源**：向 cert-manager 申请 TLS 证书
2. **cert-manager 生成证书**：基于配置的 issuer
3. **cert-manager 写入 Secret**：`<release>-webhook-server-cert`
4. **cert-manager 的 ca-injector**：自动把 CA bundle 注入到
   `ValidatingWebhookConfiguration`
5. **operator pod**：挂载证书 secret 并提供 webhook 服务

#### 何时使用 cert-manager

- ✅ **自定义有效期**：根据组织策略配置证书生命周期
- ✅ **与现有 PKI 集成**：使用组织自有的证书基础设施
- ✅ **集中证书管理**：通过 cert-manager 统一管理集群所有证书

#### 证书轮换

使用 cert-manager 时，证书轮换是**完全自动化的**：

1. **叶子证书轮换**（默认：每年）
   - cert-manager 在到期前自动续签
   - controller-runtime 自动重载新证书
   - **不需要 pod 重启**
   - **不需要更新 caBundle**（Root CA 相同）

2. **Root CA 轮换**（每 10 年）
   - cert-manager 轮换 Root CA
   - ca-injector 自动更新 `ValidatingWebhookConfiguration` 中的 caBundle
   - **无需人工干预**

#### 示例：自签名 Issuer

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: selfsigned-issuer
  namespace: dynamo-system
spec:
  selfSigned: {}
---
# Enable in platform values.yaml
dynamo-operator:
  webhook:
    certManager:
      enabled: true
      issuerRef:
        kind: Issuer
        name: selfsigned-issuer
```

---

### 外部证书

为自定义 PKI 需求自带证书。

#### 步骤

1. **手动创建证书 secret**：

```bash
kubectl create secret tls <release>-webhook-server-cert \
  --cert=tls.crt \
  --key=tls.key \
  -n <namespace>

# Also add ca.crt to the secret
kubectl patch secret <release>-webhook-server-cert -n <namespace> \
  --type='json' \
  -p='[{"op": "add", "path": "/data/ca.crt", "value": "'$(base64 -w0 < ca.crt)'"}]'
```

2. **配置 operator 使用外部 secret**：

```yaml
dynamo-operator:
  webhook:
    certificateSecret:
      external: true
    caBundle: <base64-encoded-ca-cert>  # Must manually specify
```

3. **部署 operator**：

```bash
helm install dynamo-platform . -n <namespace> -f values.yaml
```

#### 证书要求

- **Secret 名**：必须与 `webhook.certificateSecret.name` 一致（默认：
  `webhook-server-cert`）
- **Secret keys**：`tls.crt`、`tls.key`、`ca.crt`
- **证书 SAN**：必须包含 `<service-name>.<namespace>.svc`
  - 示例：`dynamo-platform-dynamo-operator-webhook-service.dynamo-system.svc`

---

## 多 Operator 部署（已废弃）

> **已废弃：** namespace-restricted 模式与多 operator 部署已废弃，将在未来版本
> 中移除。请改用单个集群范围 operator。

operator 通过基于 Lease 的协调机制，支持同时运行**集群范围**与**命名空间受限**
两种实例。

### 场景

```
Cluster:
├─ Operator A (cluster-wide, namespace: platform-system)
│  └─ Validates all namespaces EXCEPT team-a
└─ Operator B (namespace-restricted, namespace: team-a)
   └─ Validates only team-a namespace
```

### 工作原理

1. **命名空间受限的 operator** 在自己的命名空间中创建一个 Lease
2. **集群范围的 operator** 监听名为 `dynamo-operator-ns-lock` 的 Lease
3. **集群范围的 operator** 跳过对存在活跃 Lease 的命名空间的校验
4. **命名空间受限的 operator** 校验自己命名空间中的资源

### Lease 配置

Lease 机制会**根据部署模式自动配置**：

```yaml
# Cluster-wide operator (default)
namespaceRestriction:
  enabled: false
# → Watches for leases in all namespaces
# → Skips validation for namespaces with active leases

# Namespace-restricted operator
namespaceRestriction:
  enabled: true
  namespace: team-a
# → Creates lease in team-a namespace
# → Does NOT check for leases (no cluster permissions)
```

### 部署示例

```bash
# 1. Deploy cluster-wide operator
helm install platform-operator dynamo-platform \
  -n platform-system \
  --set namespaceRestriction.enabled=false

# 2. Deploy namespace-restricted operator for team-a
helm install team-a-operator dynamo-platform \
  -n team-a \
  --set namespaceRestriction.enabled=true \
  --set namespaceRestriction.namespace=team-a
```

### ValidatingWebhookConfiguration 命名

webhook 配置的名称会反映部署模式：

- **集群范围**：`<release>-validating`
- **命名空间受限**：`<release>-validating-<namespace>`

示例：

```bash
# Cluster-wide
platform-operator-validating

# Namespace-restricted (team-a)
team-a-operator-validating-team-a
```

这样可以让多个 webhook 配置共存而不冲突。

### Lease 健康度

如果命名空间受限的 operator 被删除或变得不健康：
- Lease 会在 `leaseDuration + gracePeriod` 后过期（默认约 30 秒）
- 集群范围的 operator 会自动恢复对该命名空间的校验

---

## 故障排查

### Webhook 未被调用

**现象：**
- 无效资源被接受
- 日志中没有校验错误

**检查：**

1. **确认 webhook 配置存在**：
```bash
kubectl get validatingwebhookconfiguration | grep dynamo
```

2. **检查 webhook 配置**：
```bash
kubectl get validatingwebhookconfiguration <name> -o yaml
# Verify:
# - caBundle is present and non-empty
# - clientConfig.service points to correct service
# - webhooks[].namespaceSelector matches your namespace
```

3. **确认 webhook service 存在**：
```bash
kubectl get service -n <namespace> | grep webhook
```

4. **查看 operator 启动日志中关于 webhook 的部分**：
```bash
kubectl logs -n <namespace> deployment/<release>-dynamo-operator | grep webhook
# Should see: "Registering validation webhooks"
# Should see: "Starting webhook server"
```

---

### 连接被拒（Connection Refused）

**现象：**
```
Error from server (InternalError): Internal error occurred: failed calling webhook:
Post "https://...webhook-service...:443/validate-...": dial tcp ...:443: connect: connection refused
```

**检查：**

1. **确认 operator pod 处于运行状态**：
```bash
kubectl get pods -n <namespace> -l app.kubernetes.io/name=dynamo-operator
```

2. **检查 webhook server 是否在监听**：
```bash
# Port-forward to pod
kubectl port-forward -n <namespace> pod/<operator-pod> 9443:9443

# In another terminal, test connection
curl -k https://localhost:9443/validate-nvidia-com-v1alpha1-dynamocomponentdeployment
# Should NOT get "connection refused"
```

3. **确认 deployment 中暴露了 webhook 端口**：
```bash
kubectl get deployment -n <namespace> <release>-dynamo-operator -o yaml | grep -A5 "containerPort: 9443"
```

4. **检查 webhook 初始化错误**：
```bash
kubectl logs -n <namespace> deployment/<release>-dynamo-operator | grep -i error
```

---

### 证书错误

**现象：**
```
Error from server (InternalError): Internal error occurred: failed calling webhook:
x509: certificate signed by unknown authority
```

**检查：**

1. **确认 caBundle 存在**：
```bash
kubectl get validatingwebhookconfiguration <name> -o jsonpath='{.webhooks[0].clientConfig.caBundle}' | base64 -d
# Should output a valid PEM certificate
```

2. **确认证书 secret 存在**：
```bash
kubectl get secret -n <namespace> <release>-webhook-server-cert
```

3. **检查证书有效性**：
```bash
kubectl get secret -n <namespace> <release>-webhook-server-cert -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text
# Check:
# - Not expired
# - SAN includes: <service-name>.<namespace>.svc
```

4. **检查 operator 中关于 CA 注入的日志**：
```bash
kubectl logs -n <namespace> deployment/<release>-dynamo-operator | grep -i "cert\|ca.*bundle\|inject"
```

---

### Cert Controller 错误

**现象：**
- operator 日志显示 cert-controller 错误
- 证书 Secret 未被创建
- CA bundle 未被注入到 webhook 配置

**检查：**

1. **检查 cert-controller 日志**：
```bash
kubectl logs -n <namespace> deployment/<release>-dynamo-operator | grep -i "cert-manager\|cert-rotation\|cert-controller"
```

2. **确认 RBAC 权限**：
```bash
# operator 需要管理 Secrets、ValidatingWebhookConfigurations、
# MutatingWebhookConfigurations 与 CustomResourceDefinitions 的权限
kubectl auth can-i create secrets -n <namespace> --as=system:serviceaccount:<namespace>:<release>-dynamo-operator
kubectl auth can-i patch validatingwebhookconfigurations --as=system:serviceaccount:<namespace>:<release>-dynamo-operator
```

3. **检查证书 Secret 是否被创建**：
```bash
kubectl get secret -n <namespace> <release>-webhook-server-cert
```

4. **强制重新生成证书**：
```bash
# Delete the certificate secret and restart the operator
kubectl delete secret <release>-webhook-server-cert -n <namespace>
kubectl rollout restart deployment/<release>-dynamo-operator -n <namespace>
```

---

### 校验错误信息不清晰

**现象：**
- webhook 拒绝资源，但报错信息不清晰

**解决方案：**

查看 operator 日志中的详细校验错误：

```bash
kubectl logs -n <namespace> deployment/<release>-dynamo-operator | grep "validate create\|validate update"
```

webhook 日志会包含：
- 资源名与命名空间
- 带上下文的校验错误
- 不可变字段被修改时的告警

---

### 资源卡在删除中

**现象：**
- 资源卡在 "Terminating" 状态
- webhook 阻塞了 finalizer 的移除

**解决方案：**

webhook 会自动跳过对正在删除的资源的校验。如果仍然卡住：

1. **检查是否被 webhook 阻塞**：
```bash
kubectl describe <resource-type> <name> -n <namespace>
# Look for events mentioning webhook errors
```

2. **临时绕过 webhook**：
```bash
# Option 1: Set failurePolicy to Ignore
kubectl patch validatingwebhookconfiguration <name> \
  --type='json' \
  -p='[{"op": "replace", "path": "/webhooks/0/failurePolicy", "value": "Ignore"}]'

# Option 2 (last resort): Delete ValidatingWebhookConfiguration
kubectl delete validatingwebhookconfiguration <name>
```

3. **再次删除资源**：
```bash
kubectl delete <resource-type> <name> -n <namespace>
```

4. **恢复 webhook 配置**：
```bash
helm upgrade <release> dynamo-platform -n <namespace>
```

---

## 最佳实践

### 生产部署

1. ✅ **使用 `failurePolicy: Fail`**（默认），确保校验始终生效
2. ✅ **监控 webhook 延迟** —— 校验通常会给每次资源操作增加约 10-50ms
3. ✅ **自动证书在生产环境下表现良好** —— 内置的 cert-controller 处理生成与
   轮换；只有在需要与组织 PKI 集成时才使用 cert-manager
4. ✅ 在上生产前，先在 staging 环境中**测试 webhook 配置**

### 开发部署

1. ✅ 如果开发期间 webhook 可用性不稳定，可使用 `failurePolicy: Ignore`
2. ✅ **保留自动证书**（零配置，operator 内置）

### 多租户部署

1. ✅ **部署一个集群范围 operator**，做平台级校验
2. ~~为租户专属命名空间部署命名空间受限的 operator~~（**已废弃** —— 改用
   集群范围模式）

---

## 其他资源

- [Kubernetes Admission Webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
- [cert-manager 文档](https://cert-manager.io/docs/)
- [Kubebuilder Webhook Tutorial](https://book.kubebuilder.io/cronjob-tutorial/webhook-implementation.html)
- [CEL 校验规则](https://kubernetes.io/docs/reference/using-api/cel/)

---

## 支持

如果遇到问题或有疑问：
- 查看 [故障排查](#troubleshooting) 一节
- 查看 operator 日志：`kubectl logs -n <namespace> deployment/<release>-dynamo-operator`
- 在 GitHub 上提 issue
