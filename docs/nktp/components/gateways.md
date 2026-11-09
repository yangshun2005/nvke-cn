# 网关

NVKE 遥测流水线（NKTP）为每种信号运行一个网关。每个网关在 Helm 发布命名空间的 Service 上通过带 mTLS 的 gRPC 接收 OTLP，默认将收到的数据转发到 LGTM，并通过其[边车](../architecture/self-telemetry.md)将自身的指标、日志和追踪发送到 watchdog。

| 网关 | Values 键 | gRPC |
| --- | --- | --- |
| 日志 | `gateway_logs` | `nktp-gateway-logs-collector:4317` |
| 指标 | `gateway_metrics` | `nktp-gateway-metrics-collector:4317` |
| 追踪 | `gateway_traces` | `nktp-gateway-traces-collector:4317` |

**HTTP 客户端的破坏性变更：** 端口 4318 上的默认网关 HTTP 接收器已移除。现有客户端必须改用端口 4317 上的 OTLP/gRPC，或在 `<gateway>.otelcol.config.receivers.otlp.protocols.http` 下显式配置带 TLS 的 HTTP 接收器。仅更改客户端端口不会改变其协议。DaemonSet HTTP 入口和边车回环 HTTP 接收器保持不变。

## Collector 连接

Node Collector 通过带双向 TLS 的 OTLP/gRPC 将日志、指标和追踪发送到各自的网关。大目标池通过 gRPC 发送指标，集群事件读取器通过 gRPC 发送日志。三个网关均在连接年龄达到一分钟时发送 GOAWAY，抖动 ±10%，使客户端通过 Service 重连并到达新增副本。进行中的导出在强制关闭前另有 30 秒完成。参见 [gRPC 连接管理](https://github.com/grpc/proposal/blob/master/A9-server-side-conn-mgt.md)。

根据网关余量和扩容后可接受的延迟，调整 `<gateway>.otelcol.config.receivers.otlp.protocols.grpc.keepalive.server_parameters` 下的 `max_connection_age`。调低可更快缓解；调高可减少 TCP 建立和 TLS 握手。重连可能选中同一网关，因此重新分配可能需要多轮轮换。若进行中的导出需要更长时间完成，请增大 `max_connection_age_grace`。显式配置的 HTTP 接收器不使用这些 gRPC 年龄设置。

导出器覆盖使用 `daemonset.otelcol.config.exporters.otlp/logs`、`otlp/metrics` 和 `otlp/traces`，以及 `large_target_metrics.otelcol.config.exporters.otlp` 和 `cluster_events.otelcol.config.exporters.otlp`。

Collector 配置使用 `<gateway>.otelcol.config`，Kubernetes 工作负载配置使用 `<gateway>.kubernetes`。自动扩缩需要 Kubernetes 资源指标 API。替换 `<gateway>.kubernetes.env` 时请保留 `GOMEMLIMIT` 和 `GOGC`。

每个网关的发送队列约容纳 0.48 GiB 序列化遥测数据，占其默认 4 GiB 容器限制的 12%。处理和传入请求也会消耗堆内存；请用有代表性的遥测数据验证余量。更改网关内存限制时，请在网关配置中显式设置 `exporters.otlp.sending_queue.queue_size`。参见 OpenTelemetry 的[队列大小指导](https://opentelemetry.io/docs/collector/resiliency/#sending-queue-in-memory-buffering)和[发送队列配置](https://github.com/open-telemetry/opentelemetry-collector/blob/v0.162.0/exporter/exporterhelper/README.md#sending-queue)。

网关（包括其边车）继承 `priorityClass.name`（默认 `nktp-telemetry-critical`，值为 `1000000000`，低于 system-critical Pod）；设置 `priorityClass.preemptionPolicy: Never` 可禁用抢占。使用现有类别时设 `priorityClass.create: false`，或按网关覆盖 `<gateway>.kubernetes.priorityClassName`（`""` 表示退出）。

每个网关通过其 `transform/cluster` 处理器从 `global.clusterName` 设置 `k8s.cluster.name`，除非遥测数据已有该值。当 `global.clusterName` 为空时，值为 `unknown`。Chart 通过 `K8S_CLUSTER_NAME` 环境变量传递名称，处理器在 OTTL 字符串中读取它，因此 `42` 或 `true` 等名称保持为字符串。名称不得包含双引号或反斜杠。

导出器队列默认在内存中。要将其保存在 PersistentVolume 上，请参见[配置持久化导出器队列](../guides/configure-persistent-queue.md)。

要导出到生产后端，请参见[设置后端](../getting-started.md#set-a-backend)和[保护网关导出](../guides/secure-gateway-export.md)。