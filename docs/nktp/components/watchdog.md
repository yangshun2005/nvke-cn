# Watchdog

Watchdog 通过 OTLP/gRPC 和 mTLS，从 NVKE 遥测流水线（NKTP）采集器接收内部指标、日志和链路数据，地址为 Helm 发布命名空间中的 `nktp-watchdog-collector:4317`。
Target Allocator 通过明文 OTLP/gRPC 在 `nktp-watchdog-collector:4320` 发送自身指标；该接收器仅供指标流水线使用。
Watchdog 默认将收到的遥测数据转发至 LGTM。其自身遥测数据经由 [sidecar](../architecture/self-telemetry.md) 在 `http://127.0.0.1:4319` 转发，sidecar 默认导出至 LGTM。
Watchdog 还通过 `prometheus` 接收器中的 `nktp-operator` 作业抓取 Operator 的指标端点。所有自身遥测来源见 [拓扑](../architecture/topology.md#self-telemetry)。

Watchdog 由内置的 OpenTelemetry Operator 管理。自动扩缩需要 Kubernetes 资源指标 API，通常由 metrics-server 提供。
将 `watchdog.kubernetes.targetAllocator.enabled` 设为 `false` 会导致 Helm 渲染失败。

通过 `watchdog.otelcol.config` 配置 Collector。

**可能破坏兼容性的接收器键迁移：** 默认的 `otlp` 接收器已替换为 `otlp/secure` 和 `otlp/insecure`。`watchdog.otelcol.config.receivers.otlp` 下的现有覆盖不再影响默认流水线。请将其迁移至 `watchdog.otelcol.config.receivers.otlp/secure`（用于 4317 上的 mTLS gRPC，涵盖指标、日志和链路），或迁移至 `watchdog.otelcol.config.receivers.otlp/insecure`（用于 4320 上的明文 gRPC，仅限 allocator 指标）。删除旧覆盖；`otlp` 不是任一替代项的别名。迁移这些覆盖时，请同步更新自定义流水线中的接收器引用。

与信号网关类似，watchdog 的安全 gRPC 接收器在五分钟后（±10% 抖动）发送 GOAWAY，并为进行中的导出留出 30 秒完成时间。这使客户端能够连接到新增副本，同时限制连接抖动。调整 `watchdog.otelcol.config.receivers.otlp/secure.protocols.grpc` 下的 `keepalive.server_parameters`，以匹配所需的扩容缓解和连接抖动。重连可能选中同一副本。参见 [gRPC 连接管理](https://github.com/grpc/proposal/blob/master/A9-server-side-conn-mgt.md)。

Watchdog 及其 sidecar 通过各自的 `transform/cluster` 处理器，从 `global.clusterName` 设置 `k8s.cluster.name`，除非遥测数据中已存在该值。sidecar 需要自己的处理器，因为 watchdog 的自身遥测数据从 sidecar 直接发往后端。当 `global.clusterName` 为空时，该值为 `unknown`。

导出器队列默认位于内存中。如需将其保存在 PersistentVolume 上，参见 [配置持久化导出器队列](../guides/configure-persistent-queue.md)。

使用 `watchdog.kubernetes` 配置 Collector CR 设置，使用 `watchdog.otelcol.image` 覆盖全局 `otelcol_image`。
在任何替换的 `watchdog.kubernetes.env` 列表中包含 `GOMEMLIMIT` 和 `GOGC`，以保留这些设置。
Collector Pod 继承 `priorityClass.name`；覆盖 `watchdog.kubernetes.priorityClassName` 以使用其他类，或设为 `""` 以退出。

## 脱敏

Watchdog 按照 [OWASP 日志指南](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html#data-to-exclude)，将收到的遥测数据中识别出的凭据（包括带引号的 JSON 字段和 Basic/Bearer 凭据）掩码为 `****`。
其他采集器的诊断导出必须经过 watchdog，因为[内部错误日志可能包含被拒绝数据的样本](https://opentelemetry.io/docs/collector/internal-telemetry/#events-observable-with-internal-logs)。

匹配范围涵盖指标数据点、资源和作用域属性。Span 名称、OTLP `Metric.metadata` 和 exemplar `filtered_attributes` 原样通过。

Watchdog 的 sidecar 在导出前对 watchdog 自身的日志、指标和链路数据以及 sidecar 自身的日志和指标进行脱敏。在 `watchdog.sidecar.otelcol.config.processors.redaction/basic_v1` 下配置其规则。Watchdog 保留 `basic` 链路追踪，sidecar 自身的链路追踪保持禁用。