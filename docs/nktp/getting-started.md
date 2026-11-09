# NKTP 入门

NVKE 遥测流水线（NKTP）是一个 Helm chart，用于采集 Kubernetes 集群的指标、日志和链路，并通过 OTLP 导出。本指南介绍如何安装、连接后端，并说明各集群间最常需要调整的设置。[chart README](https://github.com/dsx-ai-factory/nvke/blob/main/nktp/deploy/charts/nktp/README.md) 概述了 chart 部署的内容，[拓扑](architecture/topology.md) 链接到各组件及其配置值的页面。在 NVKE 管理集群上，`deploy/nico` 会为你部署 NKTP；参见 [管理集群遥测：入门](site-telemetry/getting-started.md)。

## 安装前准备

完整列表见 [前置条件](https://github.com/dsx-ai-factory/nvke/blob/main/nktp/deploy/charts/nktp/README.md#prerequisites)。最常需要准备的几项：

- **cert-manager。** 默认配置下必需；请按 [cert-manager 安装说明](https://cert-manager.io/docs/installation/) 单独安装。如需不使用它，参见 [不使用 cert-manager 安装](guides/configure-cert-manager.md#installing-nktp-without-cert-manager)。
- **资源指标 API**（`metrics.k8s.io`），Collector 自动扩缩器会读取它。若无其他组件提供，NKTP 可安装 metrics-server。
- **Operator 的准入 Webhook 必须可从 API server 访问**，端口为 TCP 9443。托管集群可能通过防火墙规则或安全组将其阻断。
- **容量。** 默认配置按大型集群规模设定。需为 Collector、[自遥测 sidecar](architecture/self-telemetry.md)、Target Allocator 及捆绑依赖预留容量，包括自动扩缩带来的增长。参见 [资源规格](guides/size-resources.md)。

## 安装

从 GHCR 安装已发布的 chart。若 `helm` 返回 `unauthorized`，先执行 `helm registry login ghcr.io`：

```bash
kubectl create namespace nktp
kubectl label namespace nktp pod-security.kubernetes.io/enforce=privileged
helm install nktp oci://ghcr.io/dsx-ai-factory/nvke/charts/nktp \
  --version <version> --namespace nktp --values my-values.yaml --wait
```

如需从代码检出安装，先在仓库根目录构建 chart 依赖。其归档文件未提交到仓库：

```bash
make nktp-chart-deps
kubectl create namespace nktp
kubectl label namespace nktp pod-security.kubernetes.io/enforce=privileged
helm install nktp ./nktp/deploy/charts/nktp --namespace nktp --values my-values.yaml --wait
```

- **命名空间需要 `privileged` Pod Security 级别，** 因为节点 Collector 和 node-exporter 使用宿主路径和命名空间。`--create-namespace` 创建的命名空间不带此标签。
- **也要等待 Collector。** `--wait` 只覆盖 release 自身的资源，Operator 在它返回后才创建 Collector 工作负载。运行 `kubectl get opentelemetrycollector -n nktp`，重复直到每个 Collector 的 `READY` 列显示所有预期副本就绪，如 `2/2`。若就绪停滞，用 `kubectl get pods -n nktp` 检查 Collector Pod。安装了 cert-manager 时，还需运行 `kubectl get certificate -n nktp`，等待每个 NKTP Certificate 显示 `READY=True`。若证书就绪停滞，用 `kubectl describe certificate -n nktp` 检查条件。
- **`my-values.yaml`** 保存下文各节的设置。省略则使用默认值安装。
- **版本：** 每个 release 是一个 `nktp/vX.Y.Z` 标签，发布为 chart 版本 `X.Y.Z`。将 `X.Y.Z` 传给 `--version`。[RELEASE.md](https://github.com/dsx-ai-factory/nvke/blob/main/RELEASE.md) 描述了该流程。

<details>
<summary>安装了 cert-manager 时的 Collector 和证书就绪示例</summary>

```bash
kubectl get opentelemetrycollector -A
kubectl get certificate -A
```

chart 版本 `0.1.0` 的示例输出；名称和副本数取决于 chart 版本和配置值。`READY=2/2` 表示两个预期 Collector 副本均就绪；证书应显示 `READY=True`。

```text
NAMESPACE   NAME                  MODE          VERSION   READY   AGE   IMAGE                                      MANAGEMENT
nktp        nktp-cluster-events   statefulset   0.161.0   2/2     23h   otel/opentelemetry-collector-k8s:0.162.0   managed
...
```

```text
NAMESPACE   NAME                         READY   SECRET                          AGE
nktp        nktp-cluster-events-client   True    nktp-cluster-events-client-tls  23h
...
```

</details>

## 初步查看

默认情况下，所有数据导出到 release 命名空间中捆绑的开发后端 `lgtm`：Grafana、Prometheus、Loki 和 Tempo 运行在同一个 Pod 中。数据保存在内存中，Grafana 授予匿名管理员访问权限，因此仅用于评估。

Grafana 位于 `lgtm` Service 的 3000 端口，Prometheus、Loki 和 Tempo 作为指标、日志和链路的数据源。将其端口转发绑定到 localhost：

```bash
kubectl -n nktp port-forward svc/lgtm 3000:3000 --address 127.0.0.1
```

默认包含的仪表盘见 [默认仪表盘](https://github.com/dsx-ai-factory/nvke/blob/main/nktp/deploy/charts/nktp/dashboards/README.md#default-autoloaded-into-lgtm)。

## 设置后端

所有数据通过五个 Collector 离开集群：三个网关（每种信号一个）、watchdog（承载其他 Collector 自身的遥测），以及 watchdog 的 [sidecar](architecture/self-telemetry.md)（承载 watchdog 自身的遥测）。将它们全部指向你的后端，并关闭默认的 LGTM 后端。默认导出器使用 OTLP/gRPC。请确认后端对每种信号接受哪种协议；HTTP 配置见 [安全网关导出](guides/secure-gateway-export.md)。指标后端还必须接受 Target Allocator 发送的 delta 时序指标。

```yaml
otel-lgtm:
  enabled: false
watchdog:
  otelcol: {config: {exporters: {otlp: {endpoint: "<backend>:4317", tls: {insecure: false}}}}}
  sidecar:
    otelcol: {config: {exporters: {otlp: {endpoint: "<backend>:4317", tls: {insecure: false}}}}}
gateway_logs:
  otelcol: {config: {exporters: {otlp: {endpoint: "<backend>:4317", tls: {insecure: false}}}}}
gateway_metrics:
  otelcol: {config: {exporters: {otlp: {endpoint: "<backend>:4317", tls: {insecure: false}}}}}
gateway_traces:
  otelcol: {config: {exporters: {otlp: {endpoint: "<backend>:4317", tls: {insecure: false}}}}}
```

- **到后端的 TLS 和 mTLS**：参见 [安全网关导出](guides/secure-gateway-export.md)。
- **凭据**放在 Secret 中，绝不放在 values 中。将其加入 Collector 的 `kubernetes.env`，并在导出器中用 `${env:NAME}` 引用，例如在 `headers` 中。`env` 是列表，因此需重述 chart 的 `GOMEMLIMIT` 和 `GOGC` 条目。

## 设置集群名称

设置 `global.clusterName`，使本集群的遥测携带 `k8s.cluster.name`。未设置时，NKTP 将其设为 `unknown`。

```yaml
global:
  clusterName: <cluster>
```

网关、watchdog 和 watchdog sidecar 会为尚无该属性的遥测添加它。

## 采集你的服务

NKTP 无需配置即采集所有 Pod 的日志和状态，以及节点指标。服务自身的指标仅在存在 `ServiceMonitor` 或 `PodMonitor` 时才会被采集，它们可位于任意命名空间，无需 NKTP 专用标签。若 NetworkPolicy 保护指标端口，需放行 release 命名空间。需要凭据的 monitor 引用 Secret，节点 Target Allocator 仅在 `daemonset.kubernetes.targetAllocator.prometheusCR.secretNamespaces` 列出的命名空间中读取它；参见 [节点 Collector](components/node-collector.md#monitor-credentials)。[向 NKTP 发送遥测](guides/send-telemetry.md) 是可分享给服务所有者的指南。

## 升级

Helm 从不升级从 chart 的 `crds/` 目录安装的 CRD。OpenTelemetry 和 Prometheus CRD 来自该目录，因此升级 Operator 的 chart 版本会保留旧 CRD，只有新版本才认识的字段会被静默丢弃。请在 `helm upgrade` 前应用新的 CRD。