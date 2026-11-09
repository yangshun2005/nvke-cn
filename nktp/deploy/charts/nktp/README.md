# NKTP Helm Chart

NVKE 遥测流水线（NKTP）在 Kubernetes 集群上运行 OpenTelemetry Collector。

该 Chart 部署一条流水线，采集集群的指标、日志和链路，并通过 OTLP 导出：每个节点一个 Collector、一个用于大型指标端点的抓取池、一个 Kubernetes Event 读取器、每种信号一个网关，以及一个监控流水线自身遥测的 watchdog。内置的 OpenTelemetry Operator 负责管理这些 Collector。

本 README 涵盖 Chart 的安装、配置和开发。NKTP 文档（链接见[本 README 末尾](#more-information)）深入介绍每个组件。

## 前置条件

安装 NKTP 前需准备以下条件。

- **Kubernetes 1.31+。** 内置的 metrics-server 0.8.1 支持 1.31+。每个 Collector Pod 以原生边车方式运行其自身遥测边车，即一个带 `restartPolicy: Always` 的 init 容器，Kubernetes 从 1.29 起默认启用该特性。内置的 OpenTelemetry Operator 0.161.0 本身支持 1.25+。
- **Helm 4。** 仓库在 `make/tool-versions.mk` 中固定了版本。
- **cert-manager。** 使用默认值时必需；需单独安装。NKTP 不内置也不安装它。它负责签发内部 Collector mTLS、Operator 的准入 Webhook 以及 Collector 到 Target Allocator 的 mTLS 所需证书。不安装它则需替换全部三项集成。安装与配置文档见[更多信息](#more-information)。
- **镜像仓库访问权限**，需能访问 `docker.io`、`ghcr.io`、`quay.io` 和 `registry.k8s.io` 或其镜像。卸载 Hook 还会从 `docker.io` 拉取 `rancher/kubectl`。参见[镜像仓库](#image-registry)。
- **组件间的网络访问。** Kubernetes API Server 必须能访问节点上 Operator 准入 Webhook 的 TCP 9443 端口；托管集群可能通过防火墙规则或安全组阻断该访问。节点 Collector 抓取本节点 kubelet 的 TCP 10250 端口和 node-exporter 的 TCP 9100 端口。Collector 跨节点向网关和 watchdog 发送 OTLP。控制平面组件的指标取决于发行版，这决定了这些组件是否暴露可访问的指标端点。
- **Kubernetes 资源指标 API**（`metrics.k8s.io`）。网关、大型目标和 watchdog 的自动扩缩容需要它。NKTP 默认不安装 metrics-server；若集群中没有其他组件提供该 API，请设置 `metrics-server.enabled: true`。
- **发布命名空间需为 Privileged Pod Security 级别。** 节点 Collector 挂载主机路径以读取 Pod 日志、journald 和 `/proc`。node-exporter 使用主机网络和 PID 命名空间、主机路径以及主机端口 9100。`baseline` 和 `restricted` Pod Security 级别会拒绝这些 Pod。
- **容量。** 默认值按大型集群规模设定。请检查每个 Collector、边车、Target Allocator 和内置依赖的资源请求与限制，并为自动扩缩容预留容量。

## 安装

```bash
kubectl create namespace nktp
kubectl label namespace nktp pod-security.kubernetes.io/enforce=privileged
helm install nktp oci://ghcr.io/dsx-ai-factory/nvke/charts/nktp \
  --version <version> --namespace nktp --values my-values.yaml --wait
```

已发布版本即仓库的 `nktp/vX.Y.Z` 标签；将 `X.Y.Z` 作为 `--version` 传入。`--create-namespace` 会创建命名空间但不加 Pod Security 标签。`my-values.yaml` 存放你的覆盖值；省略则使用默认值安装。Operator 在 `helm install --wait` 返回后才创建 Collector，因此也需检查它们的 Pod。

### 升级

Helm 从不升级从 Chart 的 `crds/` 目录安装的 CRD。OpenTelemetry 和 Prometheus 的 CRD 来自该目录，因此升级 Operator 的 Chart 版本会保留旧 CRD，只有新版本才认识的字段会被静默丢弃。请在 `helm upgrade` 前应用新的 CRD。

## 组件

| 组件 | Values 键 | 运行方式 | 用途 |
| --- | --- | --- | --- |
| 节点 Collector | `daemonset` | DaemonSet | 接收本节点工作负载的 OTLP，并采集 Pod 日志、journal、主机和审计日志、kubelet 和 cAdvisor 指标，以及本节点上的抓取目标。 |
| 日志、指标和链路网关 | `gateway_logs`、`gateway_metrics`、`gateway_traces` | 各一个 StatefulSet | 将每种信号导出到后端。 |
| 大型目标指标池 | `large_target_metrics` | StatefulSet | 抓取带 `nktp.nvidia.io/large-target: "true"` 标签的 Monitor，包括 Kubernetes 控制平面。 |
| 集群事件读取器 | `cluster_events` | StatefulSet | 读取 Kubernetes Event，同一时间只有一个选出的副本处于活动状态。 |
| Watchdog | `watchdog` | StatefulSet | 接收流水线自身的遥测并导出到后端。 |
| kube-state-metrics | `kube-state-metrics` | DaemonSet 和 Deployment | 将 Kubernetes 对象状态暴露为指标。 |
| OpenTelemetry Operator 和 CRD | `opentelemetry-kube-stack` | Deployment | 创建 Collector 及其 Target Allocator。 |
| node-exporter | `node-exporter` | DaemonSet | 暴露每个节点的主机指标。 |
| metrics-server | `metrics-server` | Deployment，默认关闭 | 提供自动扩缩容读取的资源指标 API。 |
| LGTM | `otel-lgtm` | Deployment | 含 Grafana、Prometheus、Loki 和 Tempo 的开发后端。仅供评估。 |

节点 Collector、大型目标池和事件读取器通过带双向 TLS 的 OTLP/gRPC 将每种信号发送到对应网关，网关再导出到后端。
每个 Collector Pod 运行一个边车 Collector，缓冲 Collector 自身的遥测并对其链路采样。除 watchdog 的边车外，所有边车都通过端口 4317 上带 mTLS 的 OTLP/gRPC 发送到 watchdog。DaemonSet 还将采集到的 Operator 和 Target Allocator 日志直接发送到该端点。Target Allocator 通过端口 4320 上明文 OTLP/gRPC 将自身指标推送到 watchdog。Watchdog 将收到的遥测直接导出到后端，不经过网关；其自身的边车将 watchdog 的遥测导出到后端。

网关约每分钟轮换连接，watchdog 每五分钟轮换一次，以便客户端能访问新增的副本。在各接收器的 `otelcol.config` 下配置该间隔。

## 向 NKTP 发送遥测

- **OTLP：** 发送到发送方所在节点的节点 Collector，gRPC 地址为 `nktp-daemonset-collector.<namespace>:4317`，HTTP 地址为 `http://nktp-daemonset-collector.<namespace>:4318`。若本地没有就绪的 Collector，流量会失败而不会转到其他节点。
- **Prometheus 指标：** 创建 `ServiceMonitor` 或 `PodMonitor`。它可以位于任意命名空间，无需 NKTP 专用标签。若有 NetworkPolicy 保护指标端口，需放行发布命名空间。若 Monitor 引用了 Secret 或 CA ConfigMap，需将其命名空间列入 `daemonset.kubernetes.targetAllocator.prometheusCR.secretNamespaces`，以便节点 Target Allocator 读取。为兼容引用 Collector 容器内文件的 Monitor，节点和大型目标分配器都将 `prometheusCR.denyFSAccessThroughSMs` 默认设为 `false`。将其设为 `true` 前，请将所选 Monitor 中的文件引用替换为 Secret 或 CA ConfigMap 引用；否则受影响的抓取任务会被丢弃。
- **大型抓取响应：** 给 Monitor 打上 `nktp.nvidia.io/large-target: "true"` 标签，改由大型目标池抓取（每次抓取预留更多内存），而非节点 Collector。
- **日志：** 无需配置即可从每个节点采集 Pod 的 stdout 和 stderr。

## 配置

每个 Collector 组件具有相同的 values 结构：

| 键 | 内容 |
| --- | --- |
| `<component>.otelcol.config` | 完整的原生 Collector 配置。 |
| `<component>.otelcol.image` | 仅覆盖此组件的镜像。 |
| `<component>.kubernetes` | `OpenTelemetryCollector` 设置，如副本数、资源、env、volumes、autoscaler 和 Target Allocator。 |

Chart 按原样渲染每个 Collector 配置。其他值不会改变它，因此要修改流水线，就修改其配置。

每个网关默认发送队列约 0.48 GiB（按序列化字节计），占其 4 GiB 容器限制的 12%。堆内存用量大于序列化大小；调整内存限制时请检查队列容量和处理余量。

Helm 会整体替换 values 文件中的列表。覆盖 `kubernetes.env`、`kubernetes.volumes` 或接收器的 `scrape_configs` 等列表时，请从 `values.yaml` 复制默认列表并修改副本。每个替换的 `env` 列表都要保留 `GOMEMLIMIT` 和 `GOGC`。集群事件读取器还需要 `POD_NAMESPACE`。

watchdog、节点 Collector 和大型目标池需要各自的 Target Allocator。对其中任一设置 `targetAllocator.enabled: false` 会导致渲染失败。

### 遥测脱敏

在 `values.yaml` 的默认 Collector 配置中启用。

- 屏蔽日志正文（包括嵌套正文）中可识别的凭据，以及日志属性、指标数据点属性、Span 属性和事件、资源/作用域属性中的凭据。使用 `****`，包括带引号的 JSON 和 Basic/Bearer 文本。保留键名和计数器（如 `token_count`）。不做哈希。
- 在默认源和网关流水线中最后运行。在 `<workload>.otelcol.config.processors` 下自定义规则，或从相关流水线的 processor 列表中移除其引用以禁用脱敏。自定义流水线必须显式包含脱敏。
- 基本匹配不覆盖 OTLP `Metric.metadata`、exemplar 的 `filtered_attributes`、Span 名称、Span 链接属性、属性映射和数组内的递归键匹配、未知凭据名称，以及复杂和编码的密钥。属性映射和数组仍可通过其文本表示匹配值模式。
- Watchdog 对其他 Collector 的诊断导出（`service.telemetry`）进行脱敏，因为内部日志可能包含被拒数据的样本。将这些导出直接发送到后端会绕过脱敏。
- Watchdog 的边车对 watchdog 自身的日志、指标和链路，以及其自身的日志和指标进行脱敏。在 `watchdog.sidecar.otelcol.config` 下配置。Watchdog 保留 `basic` 链路追踪，边车自身的链路追踪保持禁用。
- 脱敏改变的是导出的副本。原始日志文件和容器 stdout（包括 `kubectl logs`）仍可能包含密钥。

遵循 OWASP 日志指南，尽可能从日志中排除敏感数据。当字段名称或存在与否本身敏感时，在源头省略这些字段。

### 镜像仓库

设置 `global.imageRegistry` 可覆盖所有组件的镜像仓库，包括 Collector（`otelcol_image`）和 Target Allocator（`targetallocator_image`）。注意：`metrics-server` 依赖、嵌套的 OpenTelemetry Operator 以及卸载 Hook 的 kubectl 镜像（`opentelemetry-kube-stack.cleanupJob.image`）**不会**读取此设置。需要时请显式配置它们的镜像仓库。

### 集群名称

设置 `global.clusterName`，可在遥测离开 NKTP 发往后端时添加 `k8s.cluster.name`。已带 `k8s.cluster.name` 的遥测保留原值。`global.clusterName` 为空时，该值为 `unknown`。

## 生产环境前

- **禁用内置 LGTM 后端。** 它将数据保存在内存中，授予 Grafana 匿名管理员访问权限，且不可扩展。设置 `otel-lgtm.enabled: false`，然后在每个网关、watchdog 及 watchdog 边车的 `otlp` exporter 中设置你的后端端点。gRPC 用 `otlp`，HTTP 用 `otlp_http`，与后端支持的接收协议一致。更改协议时，也要更新每个受影响流水线的 `exporters` 列表。
- **保护后端导出。** 默认 LGTM 后端使用明文 OTLP 供本地开发。将网关流量发往生产后端前启用 TLS；若后端要求 mTLS，请将客户端证书挂载到网关。
- **将凭据存放在 Secret 中。** 将 Secret 添加到 Collector 的 `kubernetes.env`，并在 exporter 中用 `${env:NAME}` 引用，例如在 `headers` 中。
- **提供资源指标 API**，以便网关、大型目标池和 watchdog 自动扩缩容。
- **检查资源请求和限制**，位于 `<component>.kubernetes.resources` 和 `<component>.kubernetes.targetAllocator.resources`。自动扩缩容目标相对于请求值。

## 内置依赖

| Chart | 版本 | 安装内容 | 默认启用 | Values 键 |
| --- | --- | --- | --- | --- |
| opentelemetry-kube-stack | 0.25.1 | OpenTelemetry Operator Chart 0.126.0、Operator 0.161.0 和 CRD | 是 | `opentelemetry-kube-stack` |
| prometheus-node-exporter | 4.56.1 | node-exporter | 是 | `node-exporter` |
| metrics-server | 3.13.1 | metrics-server 0.8.1 | 否 | `metrics-server` |

Collector、Target Allocator、kube-state-metrics 和 LGTM 镜像在 `values.yaml` 中分别通过 `otelcol_image`、`targetallocator_image`、`kube-state-metrics.image` 和 `otel-lgtm.image` 设置。

## 开发 Chart

| 路径 | 内容 |
| --- | --- |
| `values.yaml` | 默认值，包括所有 Collector 配置 |
| `templates/` | 每个组件一个模板，外加 RBAC 和证书 |
| `dashboards/` | 预置到 LGTM 的 Grafana 仪表盘和示例 |

在仓库根目录运行 Chart 检查：

```bash
make nktp-chart-deps   # download the dependencies into charts/
make nktp-chart-check  # lint, render and validate the chart; the merge gate runs this
```

在代码检出中，`nktp/AGENTS.md` 涵盖 Chart 的约束（包括版本升级和依赖固定），`nktp/hack/e2e-validate.sh` 在 Kind 集群上安装 Chart 并断言每种信号都能到达。当变更改变组件行为或 values 时，请在同一 Pull Request 中更新 `docs/nktp/components/` 下对应的页面。

本 README 会打包进 Chart，其中指向其他文件的链接无法解析。请保持其自包含：除下方链接外，不得链接到其他文档。

## 更多信息

[NKTP 文档](https://github.com/dsx-ai-factory/nvke/tree/main/docs/nktp)涵盖每个组件、配置指南、运维和设计文档。