# 启用 eBPF 插桩

[OpenTelemetry eBPF Instrumentation](https://opentelemetry.io/docs/zero-code/obi/)
（OBI）观测节点上服务收发的请求（如 HTTP 和 gRPC）。它为这些请求生成追踪数据和请求速率、错误、时长（RED）指标，适用于任何语言的服务，无需改动其代码或镜像。

OBI 以特权 DaemonSet 运行，每个节点都需要支持 BTF 的 Linux 内核。按 [Deploy OBI in Kubernetes with Helm](https://opentelemetry.io/docs/zero-code/obi/setup/kubernetes-helm/) 所述，用其 Helm chart 安装。将其 OTLP exporter 指向节点 collector `http://nktp-daemonset-collector.nktp:4318`，并[选择要观测的工作负载](https://opentelemetry.io/docs/zero-code/obi/configure/service-discovery/)。端点中只需把点号后的命名空间（`.nktp`）替换为发布版本所在命名空间，保留 `nktp-daemonset-collector` 前缀。

若服务同时使用[自动插桩](enable-auto-instrumentation.md)，两者都会上报其请求。请在仪表板中区分这两个来源，避免重复计数请求。