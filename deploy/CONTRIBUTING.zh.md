# 为 Dynamo Deploy 贡献

欢迎来到 Dynamo Deploy 项目！本指南将帮助你开始为 Dynamo 分布式推理（inference）平台的部署基础设施和工具贡献代码。

## 起步

### 前置条件


### 快速搭建

### 项目结构

deploy 目录包含若干关键组件：

```
├── discovery # 如何使用 Dynamo Kubernetes 发现后端（discovery backend）
├── helm
│   └── charts
│       ├── crds # Dynamo CRD helm chart
│       ├── platform # Dynamo platform helm chart
├── inference-gateway # Dynamo 与 inference gateway 的集成
├── observability # 用于 Dynamo k8s 的可观测性（observability）工具
├── operator # Dynamo operator 源代码
├── pre-deployment # 用于检查 k8s 集群是否满足部署 Dynamo 要求的预部署脚本
└── utils # 用于 Dynamo 基准测试与性能分析工作流的工具与清单
```

## 开发环境

### 配置环境


### IDE 配置

**VS Code：**

- 安装 Go 扩展
- 安装 Python 扩展
- 配置 Go 格式化与 lint 设置
- 添加 workspace 设置以保持一致的格式化

### 贡献流程注意事项

- 我们采用签名提交（signed commits）

```bash
commit -S
```

- 每次修改 `deploy/helm/charts/crds/templates/*.yaml` 时，请提升以下位置中 CRD helm chart 的版本：
    1. deploy/helm/charts/platform/components/operator/Chart.yaml
    2. deploy/helm/charts/platform/Chart.yaml
然后

```bash
deploy/helm/charts/platform
helm dependency update
```

#### 提交信息规范

遵循约定式提交（conventional commit）格式：

- `feat:` 新特性
- `fix:` 错误修复
- `docs:` 文档变更
- `test:` 添加或更新测试
- `refactor:` 代码重构
- `perf:` 性能改进
- `ci:` CI/CD 变更

示例：

```
feat(operator): add support for custom resource limits
fix(sdk): resolve service discovery timeout issue
docs(helm): update deployment guide with new examples
test(e2e): add integration tests for disaggregated serving
```

## 风格指南

### Go 代码风格（Operator）

遵循 Go 标准约定。


### Python 代码风格（SDK）

遵循 PEP 8，并采用现代 Python 实践：


### YAML/Helm 模板

```yaml
# Use consistent indentation (2 spaces)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "dynamo.fullname" . }}
  labels:
    {{- include "dynamo.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "dynamo.selectorLabels" . | nindent 6 }}
```

## 测试

提交 MR 并通过标准检查后，可通过添加 “/ok to test <COMMIT-ID>” 评论来触发集成测试。


### 单元测试

**Go 测试（Operator）：**

```bash
cd deploy/operator
go test ./... -v
go test -race ./...
```

### 集成测试

**端到端部署测试：**

```bash
# 运行完整部署测试套件
pytest tests/serve/test_dynamo_serve.py -v

# 测试特定部署场景
pytest tests/serve/test_dynamo_serve.py::test_serve_deployment[agg] -v
```

**Operator 集成测试：**

```bash
cd deploy/operator
make test-e2e
```

### 编写测试

**单元测试示例：**

**集成测试示例：**


### 示例测试

确保文档示例可正常运行。


感谢为 Dynamo Deploy 做出贡献！🚀
