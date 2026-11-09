# 大目标指标

大目标指标池（`large_target_metrics`）选择带有元数据标签 `nktp.nvidia.io/large-target: "true"` 的 ServiceMonitor、PodMonitor、Probe 和 ScrapeConfig 资源。将此标签应用于抓取响应较大的端点的监控资源，例如 kube-state-metrics。

Target Allocator 将抓取任务分配到池的各个副本。采集器每 10 秒刷新一次分配的目标。通过 `large_target_metrics.kubernetes.targetAllocator.collectorTargetReloadInterval` 配置。

Target Allocator 将 `prometheusCR.denyFSAccessThroughSMs` 默认为 `false`，以兼容引用本地 bearer-token 或 TLS 文件的现有 ServiceMonitor 和 PodMonitor。被引用的文件必须存在于每个抓取采集器容器中。编写监控器时优先使用 Secret 或 CA ConfigMap 引用。启用 `denyFSAccessThroughSMs` 之前，确保分配器所选监控器均未引用采集器容器中的文件。参见[限制监控器文件访问](../guides/restrict-monitor-file-access.md)。

将 `large_target_metrics.kubernetes.targetAllocator.enabled` 设为 `false` 会导致 Helm 渲染失败。Target Allocator 遥测数据发送至 watchdog。分配器在 `large_target_metrics.kubernetes.targetAllocator` 下拥有独立的安全上下文和资源预算。根据发现的 Kubernetes 对象和目标为其资源设置大小。

采集的指标通过 OTLP/gRPC 发送至 `nktp-gateway-metrics-collector:4317`。每个采集器通过其[边车](../architecture/self-telemetry.md)将自身遥测数据发送至 watchdog。使用 `large_target_metrics.otelcol.config` 配置采集器内部配置，使用 `large_target_metrics.kubernetes` 配置工作负载。采集器 Pod 继承 `priorityClass.name`；覆盖 `large_target_metrics.kubernetes.priorityClassName` 以使用其他类，或设为 `""` 退出，不影响独立 Target Allocator 的优先级。

## 大抓取任务的大小设置

批处理器在队列准入前拆分抓取任务。内存限制器在 Go 堆超过 7.8 GiB 时开始拒绝，低于 12 GiB 容器限制。自动扩缩需要 Kubernetes 资源指标 API，并按完整目标分配。单个端点仍须能放入一个采集器。抓取默认将每个响应限制为 1,000,000 个样本和 1 GiB。尽可能将超过这些限制的端点拆分为更小的抓取目标。若提高抓取限制，需同时调整导出器队列、内存请求和限制、`GOMEMLIMIT` 及内存限制器阈值。响应字节数不能直接衡量解析和导出抓取任务所需的内存。替换环境变量列表时保留 `GOMEMLIMIT` 和 `GOGC`。对于每 60 秒抓取一次的大目标，考虑在 ServiceMonitor 或 PodMonitor 端点上设置 `scrapeTimeout: 55s`。保持超时低于抓取间隔。

## Kubernetes 组件指标

NVKE 遥测流水线（NKTP）为 kube-apiserver、kube-controller-manager、kube-scheduler、etcd、CoreDNS 和 kube-proxy 启用 kube-stack ServiceMonitor。它们被标记为 `nktp.nvidia.io/large-target: "true"`，由大目标指标 Target Allocator 选择。组件可用性、标签、端口和网络暴露因 Kubernetes 发行版而异，因此渲染出的 ServiceMonitor 本身并不保证目标可达。大目标采集器的服务账户对 `/metrics` 非资源 URL 具有只读授权。

来自 opentelemetry-kube-stack 0.25.1 的 ServiceMonitor 引用采集器容器中的 bearer-token 文件，对于安全端点还引用 CA 文件。因此大目标 Target Allocator 设置 `denyFSAccessThroughSMs: false`。将其设为 `true` 会导致这些 ServiceMonitor 端点被丢弃，如 [opentelemetry-helm-charts issue #2408](https://github.com/open-telemetry/opentelemetry-helm-charts/issues/2408) 所述。