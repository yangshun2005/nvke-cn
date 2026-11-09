# NKTP Grafana 仪表板

随 Chart 一起发布的 JSON Grafana 仪表板。`default/` 中预置的仪表板使用 Classic JSON 模型。`examples/` 中仅供导入的仪表板使用 V1 Resource 模型，仪表板位于 `spec` 下。导入示例时请使用 Grafana 12.2 或更高版本。

**`default/`** 在 `otel-lgtm.enabled` 为 true（Chart 默认值）时预置到内置的 LGTM Grafana。Grafana 在 Pod 启动时读取它们。修改默认仪表板会通过 ConfigMap 校验和滚动重启 LGTM Pod。禁用 LGTM 会移除该预置。

**`examples/`** 不会预置。请在你自己的 Grafana 中使用。

## 导入（你自己的 Grafana）

1. 打开 Grafana → **Dashboards** → **Import**。
2. 上传 JSON 文件（或粘贴其内容）。
3. 选择一个 Prometheus 数据源。示例仪表板声明了一个名为 `DS_PROMETHEUS` 的输入。
4. 确认所导入仪表板要求的标签。大多数示例查询使用 `k8s_cluster_name`（来自 OpenTelemetry 资源属性 `k8s.cluster.name`）。

## 默认（自动加载到 LGTM）

每个 Collector 都有自己的仪表板，标签为 `nktp-collector`。它们共享同一布局：一个 **Pipeline** 行，显示 Collector 处理的每种信号的接收、导出、失败和拒绝条目总数，以及最繁忙的 20 个副本；该 Collector 专属的行；一个 **Queues and resources** 行，显示最大的 20 个副本；以及一个 **Self-telemetry sidecar** 行，显示全部三种信号。顶栏的 **Collectors** 下拉菜单可在它们之间切换。查询选择主 Collector（`job="nktp-<name>"`）或其 sidecar（`job="nktp-<name>-sidecar"`），绝不两者同时选择。

| 文件 | 标题 | 描述 |
| --- | --- | --- |
| `default/nktp-health.json` | NKTP Health | 所有 Collector 的工作负载就绪状态、流水线故障、队列压力、诊断日志和内部追踪，并为 sidecar 单独设一行。 |
| `default/nktp-daemonset.json` | NKTP DaemonSet | 每个节点的节点 Collector 流水线（全部三种信号）、其 Target Allocator、资源和 sidecar。 |
| `default/nktp-watchdog.json` | NKTP Watchdog | Watchdog 流水线、其 Target Allocator 和 Operator 抓取、资源以及 sidecar。 |
| `default/nktp-gateway-logs.json` | NKTP Logs Gateway | 日志网关流水线、资源和 sidecar。 |
| `default/nktp-gateway-metrics.json` | NKTP Metrics Gateway | 指标网关流水线、资源和 sidecar。 |
| `default/nktp-gateway-traces.json` | NKTP Traces Gateway | 追踪网关流水线、资源和 sidecar。 |
| `default/nktp-cluster-events.json` | NKTP Cluster Events | Kubernetes Event 读取器流水线、领导者选举、资源和 sidecar。 |
| `default/nktp-large-target-metrics.json` | NKTP Large-target Metrics | 大目标池流水线、其 Target Allocator、最大的目标、kube-state-metrics 抓取健康状态、资源和 sidecar。 |
| `default/nktp-operator.json` | NKTP Operator | OpenTelemetry Operator 协调、Webhook HTTP 流量和进程健康状态。需要在所选 Prometheus 数据源中抓取 Operator 指标。 |
| `default/otel-lgtm-health.json` | OTel LGTM Health | 内置 LGTM Collector 运行状态、接收/导出速率、内存和导出器队列。 |

## 示例（仅导入）

| 文件 | 标题 | 描述 |
| --- | --- | --- |
| `examples/availability-per-cluster-reference.json` | Reference - Cluster Availability | 各集群容量可用性（GPU）和服务可用性（Kubernetes API）。需要 `k8s_cluster_name`。额外安装：带 DCGM Exporter 的 NVIDIA GPU Operator；NVSentinel Janitor。 |
| `examples/etcd-reference.json` | Reference - etcd | etcd 健康状态、RPC、存储、内存和网络流量。需要一个抓取 etcd 的 Prometheus 数据源；可用时集群过滤器使用 `k8s_cluster_name`。 |
| `examples/kubernetes-health-reference.json` | Reference - Kubernetes Health | 集群、控制平面、节点、Pod 和工作负载健康状态。需要 `k8s_cluster_name`。 |
| `examples/nvsentinel-latency-errors-reference.json` | Reference - NVSentinel Latency and Errors | NVSentinel 延迟、错误、部署健康状态、GPU 健康状态、节点排空、MongoDB 和故障隔离。需要 `k8s_cluster_name`。参见 https://github.com/NVIDIA/NVSentinel。 |