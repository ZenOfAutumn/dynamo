# Grafana 仪表盘示例

本目录包含用于 Dynamo 可观测性（observability）的 Grafana 仪表盘示例。这些是起步文件，可作为构建自定义仪表盘的参考。

- `dynamo.json` - 通用 Dynamo 仪表盘，展示软件与硬件指标
- `sglang.json` - SGLang 引擎指标（请求延迟、吞吐、缓存）以及 HiCache KV 缓存（KV cache）指标（GPU/CPU 层级使用情况、驱逐/回载、PIN 计数）
- `disagg-dashboard.json` - 解耦（disaggregated）服务仪表盘 - 详见 [DASHBOARD_METRICS.md](DASHBOARD_METRICS.md) 了解所有指标与面板的详细文档
- `dcgm-metrics.json` - 使用 DCGM exporter 数据的 GPU 指标仪表盘
- `kvbm.json` - KV Block Manager 指标仪表盘
- `temp-loki.json` - 用于 Loki 集成的日志仪表盘
- `dashboard-providers.yml` - 仪表盘自动配置文件

关于安装与使用说明，请参阅 [可观测性文档](../../../docs/observability/)。

关于 Kubernetes 部署设置，请参阅 [../k8s/MONITORING_SETUP.md](../k8s/MONITORING_SETUP.md)。
