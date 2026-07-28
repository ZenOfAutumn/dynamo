# Dynamo 解耦（disaggregated）部署的监控搭建

## 先决条件

k8s 集群（cluster）必须已配置 GPU operator，并启用 DCGM ServiceMonitor 以便 Prometheus 采集指标（metrics）。

```bash
helm upgrade --install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator \
  --create-namespace \
  --set operator.defaultRuntime=containerd \
  --set gdrcopy.enabled=true \
  --set dcgmExporter.serviceMonitor.enabled=true \
  --set dcgmExporter.serviceMonitor.additionalLabels.release=prometheus \
  --wait --timeout=600s
```

监控相关的关键设置为：
- `dcgmExporter.serviceMonitor.enabled=true` - 启用 ServiceMonitor 创建
- `dcgmExporter.serviceMonitor.additionalLabels.release=prometheus` - 添加用于 Prometheus 发现的标签

## 安装

集群正确配置后，运行：

```bash
./setup-monitoring.sh
```

该脚本会：
1. 安装 kube-prometheus-stack（Prometheus + Grafana + Alertmanager）
2. 配置 Prometheus 跨所有命名空间发现 pod monitor
3. 用 Prometheus 端点更新 Dynamo operator
4. 为 NVLink profiling 配置 DCGM 自定义指标
5. 验证 DCGM ServiceMonitor 是否存在（由 GPU operator 创建）
6. 部署 Grafana 解耦仪表盘 ConfigMap（由 Grafana sidecar 自动导入）
7. 提供 Grafana 凭据

### 远程写入（可选）

要将抓取到的指标推送到外部 Prometheus（或任何兼容 remote-write 的端点），在运行脚本前设置以下环境变量：

| 变量                          | 说明                                          | 默认值       |
|-------------------------------|----------------------------------------------|-------------|
| `REMOTE_WRITE_HOST`           | 远程写入端点的主机                              | —（未设置则禁用） |
| `REMOTE_WRITE_PORT`           | 远程写入端点的端口                              | `9091`      |
| `REMOTE_WRITE_METRIC_REGEX`   | 用于匹配要发送的指标名的 Prometheus 正则        | `^agent_.*` |

只有在设置了 `REMOTE_WRITE_HOST` 时才会发送指标。脚本会为 Prometheus 配置一个 `remote_write` 目标：`http://<host>:<port>/api/v1/write`。只有名称匹配 `REMOTE_WRITE_METRIC_REGEX` 的指标才会被发送，其它都会被过滤掉。你可以使用任意合法的 Prometheus 正则（例如 `^agent_.*`、`^nixl_.*`、`^(agent_|nixl_).*`）。

示例：

```bash
# 默认过滤器：agent_.*
REMOTE_WRITE_HOST=prom.example.com REMOTE_WRITE_PORT=9091 ./setup-monitoring.sh

# 自定义过滤器：仅 nixl_ 指标
REMOTE_WRITE_HOST=prom.example.com REMOTE_WRITE_METRIC_REGEX='^nixl_.*' ./setup-monitoring.sh

# 多个模式：agent_ 或 nixl_
REMOTE_WRITE_HOST=prom.example.com REMOTE_WRITE_METRIC_REGEX='^(agent_|nixl_).*' ./setup-monitoring.sh
```

## 验证

检查 GPU 指标是否正常上报：

```bash
# 验证 DCGM ServiceMonitor 存在且归属于 ClusterPolicy
kubectl get servicemonitor -n gpu-operator nvidia-dcgm-exporter -o yaml | grep -A 5 ownerReferences

# 在 Prometheus 中查询 GPU 指标
kubectl exec -n monitoring prometheus-prometheus-kube-prometheus-prometheus-0 -- \
  wget -q -O- 'http://localhost:9090/api/v1/query?query=DCGM_FI_DEV_GPU_UTIL'
```

预期：ServiceMonitor 应显示 `ownerReferences` 指向 ClusterPolicy，查询应返回 8 条以上 GPU 时序。
