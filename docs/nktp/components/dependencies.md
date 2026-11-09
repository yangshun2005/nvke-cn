# 托管依赖

NVKE 遥测流水线（NKTP）将这些上游 chart 作为 Helm 依赖安装。其版本固定于 `nktp/deploy/charts/nktp/Chart.yaml` 和 `nktp/deploy/charts/nktp/Chart.lock`。每项配置通过其 values 键透传，使用上游 chart 自身的 values。

## OpenTelemetry Operator 与 CRD

NKTP 依赖已发布的 [opentelemetry-kube-stack 0.25.1](https://github.com/open-telemetry/opentelemetry-helm-charts/tree/opentelemetry-kube-stack-0.25.1/charts/opentelemetry-kube-stack) chart。它提供 OpenTelemetry Operator chart 0.126.0、operator 镜像 0.161.0 及原生 CRD。NKTP 禁用其默认 Collector 及通用 Collector RBAC。配置通过 `opentelemetry-kube-stack` 透传，使用上游 values。

Helm 会在 chart 资源之前安装缺失的 OpenTelemetry CRD（OpenTelemetryCollector、Instrumentation、TargetAllocator、OpAMPBridge）和 Prometheus CRD（ServiceMonitor、PodMonitor、Probe、ScrapeConfig）。

kube-stack chart 还提供大目标池抓取的 Kubernetes 组件 ServiceMonitor。参见 [Kubernetes 组件指标](large-target-metrics.md#kubernetes-component-metrics)。

`OpenTelemetryCollector` 资源支持的最低 API 版本为 `opentelemetry.io/v1beta1`。

当 Operator 或 CRD 由其他机制管理时，使用以下独立配置。

```yaml
opentelemetry-kube-stack:
  opentelemetry-operator:
    enabled: false
  crds:
    installOtel: false
    installPrometheus: false
```

通过 `opentelemetry-kube-stack.opentelemetry-operator.manager.image.tag` 覆盖 Operator 镜像。

Operator 的准入 Webhook 证书默认来自 cert-manager。替代方案参见 [配置 cert-manager](../guides/configure-cert-manager.md#admission-webhook)。

## Node exporter

NKTP 默认安装 [prometheus-node-exporter](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-node-exporter)，从每个节点采集主机级指标。它以 DaemonSet 运行，使用主机网络，在每个节点绑定端口 9100，并提供 ServiceMonitor，使 exporter 可作为抓取目标被发现。

禁用示例（例如集群已运行 node exporter）：

```yaml
node-exporter:
  enabled: false
```

exporter 的 cpufreq 采集器通过 `--no-collector.cpufreq` 关闭。在 node_exporter 1.12.1 中，该采集器在使用 `cppc_cpufreq` 驱动的 arm64 节点上会挂起，导致对 exporter 的抓取超时（[prometheus/node_exporter#3791](https://github.com/prometheus/node_exporter/issues/3791)）。因此 exporter 不产生 `node_cpu_frequency_*` 和 `node_cpu_scaling_*` 序列。要重新启用该采集器，将 `extraArgs` 设为空列表：

```yaml
node-exporter:
  extraArgs: []
```

## Metrics server

[metrics-server](https://github.com/kubernetes-sigs/metrics-server) 提供 Kubernetes 资源指标 API（`metrics.k8s.io`），供 `kubectl top` 和 Horizontal Pod Autoscaler 读取。它驱动三个信号网关、大目标指标池和 watchdog 的水平自动扩缩：每个带 `autoscaler` 块的工作负载按 CPU 和内存利用率扩缩，而 HPA 控制器只能从此 API 读取这些指标。

NKTP **不**默认安装它，因为许多集群已从托管控制平面或集群插件获得，第二份副本会与集群单例 `v1beta1.metrics.k8s.io` APIService 冲突。若完全没有提供者，这些 HPA 无法计算建议并停止扩缩，使每个工作负载保持当前副本数。当无其他来源提供该 API 时启用：

```yaml
metrics-server:
  enabled: true
```

捆绑 chart 固定为 metrics-server chart 3.13.1，安装 metrics-server 0.8.1。该版本支持 Kubernetes 1.31+。

配置通过 `metrics-server` 透传，使用上游 chart 自身的 values，NKTP 不覆盖其中任何一项。可配置项参见 [chart 的 values 参考](https://github.com/kubernetes-sigs/metrics-server/blob/metrics-server-helm-chart-3.13.1/charts/metrics-server/README.md)，包括 APIService 和 kubelet 证书验证的 TLS 默认值。