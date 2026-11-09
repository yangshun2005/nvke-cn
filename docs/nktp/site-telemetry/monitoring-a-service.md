# 管理集群遥测：监控你的服务

NVKE 遥测流水线（NKTP）无需额外配置，即可采集每个 Pod 的日志、每个 Pod 的状态以及节点指标。应用指标则不同：只有当服务通过 `ServiceMonitor` 或 `PodMonitor` 声明如何被抓取时，NKTP 才会抓取该服务。NKTP 不为其他组件提供监控器。**每个服务自带监控器**，通常来自该服务的 Helm chart，因此监控器会随其描述的服务一起进行版本管理、打标签和删除。

目前，NVKE controller、Temporal 和 SPIRE 通过各自的 `deploy/nico` overlay 启用了监控器。cert-manager、NodeLocal DNSCache、OVN-Kubernetes 和 CAPNICo 没有启用，因此在它们的负责人添加监控器之前，其指标不会被采集。

## 添加监控器

大多数 chart 已有对应开关，这是首先要查找的内容：

| 组件 | 配置项 |
|---|---|
| 本仓库的 controller chart | `serviceMonitor.enabled` |
| Temporal | `server.metrics.serviceMonitor.enabled` |
| SPIRE | `spire-server.telemetry.prometheus.podMonitor.enabled`、`spire-agent.telemetry.prometheus.podMonitor.enabled` |
| cert-manager | `prometheus.servicemonitor.enabled` |

否则，你需要自行提供监控器。`ServiceMonitor` 按标签选择 Service，并指定提供指标的 Service 端口：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-service
  namespace: my-namespace
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: my-service
  endpoints:
    - port: metrics        # the Service port's name, not its number
      interval: 30s
```

- **当没有 Service 暴露指标端口时，使用 `PodMonitor`**。它选择 Pod 并指定容器端口。
- **将监控器放在你自己的命名空间中。** NKTP 会发现每个命名空间中的监控器，监控器不需要 NKTP 专用标签。需要凭据的监控器是例外；参见[需要认证的端点](#authenticated-or-https-endpoints)。

**顺序。** 监控 CRD 随 NKTP 一起提供。在管理集群上，`nktp` 项会在 `deploy/nico/nvke-deploy.yaml` 中位于其后的所有项之前安装这些 CRD，因此在路径中稍后安装的 chart 可以创建其监控器。在 NKTP 之前安装的组件，或在路径之外安装的组件（通过 `clusterctl init` 安装的 cert-manager 和 CAPI，通过 bootstrap 安装的 OVN 和 CoreDNS），必须在 NKTP 启动后添加其监控器。

## 由哪个 Collector 抓取

| 监控器 | 抓取者 | 适用场景 |
|---|---|---|
| 无标签（默认） | 每个目标所在节点上的 Collector | 几乎所有场景：每个 Pod 一个目标 |
| 标记为 `nktp.nvidia.io/large-target: "true"` | large-target 池，将目标分散到其副本上 | 响应非常大的目标，如 kube-state-metrics，或不在集群节点上运行的目标 |

节点 Collector 将每个目标分配给其所在节点，因此无法解析到节点的目标（例如集群外的静态地址）会保持未分配状态。为这些监控器打上 large-target 池的标签。

## 允许 Collector 访问

如果你的服务通过 NetworkPolicy 限制谁可以访问其指标端口，请放行 `nktp` 命名空间。controller chart 通过 `metrics.networkPolicy.additionalNamespaces: [nktp]` 实现这一点。

## 需要认证或 HTTPS 的端点

- **凭据来自 Secret，而非文件。** 节点 Collector 会拒绝引用 Collector 容器内文件的监控器端点（`bearerTokenFile`、`tlsConfig.caFile` 等），因此监控器无法借用 Collector 自身的 ServiceAccount token。请改为引用 Secret，例如 `authorization.credentials` 或 `tlsConfig.ca.secret`。
- **必须允许分配器读取这些 Secret。** 监控器的 Secret 引用在其自身命名空间中解析，而分配器只读取 NKTP values 中 `daemonset.kubernetes.targetAllocator.prometheusCR.secretNamespaces` 列出的命名空间中的 Secret。随后 chart 会在该命名空间创建 Role，授予其 Secret 和 ConfigMap 读取权限，并通过 mTLS 将凭据传递给其 Collector。
- **将监控器与其凭据放在一起。** 该授权覆盖命名空间中的所有 Secret，因此与其列出服务的命名空间，不如将监控器及其凭据 Secret 放在专用命名空间中，只列出该命名空间，并通过 `spec.namespaceSelector.matchNames: [<service-namespace>]` 将监控器指回你的服务。
- **自签名证书**可通过 `tlsConfig.insecureSkipVerify: true` 抓取。连接仍然加密，但目标未经过认证。从监控器可引用的 CA 提供证书是更好的做法。

## 检查

监控器创建几分钟后，其目标应报告 `up`。按照[验证](verification.md#2-the-signals-arrive)中的说明，对后端进行端口转发并定义 `promql` 辅助函数，然后执行：

```bash
promql 'up{job="<service-name>"}'     # 1 per target; a PodMonitor's job is <namespace>/<monitor>
promql 'count({job="<service-name>", __name__!="up"})'   # the service's own series
```

- **完全没有 `up` 序列：** 监控器没有匹配到任何目标（检查其选择器和端口名），或者目标未被分配。
- **`up` 为 0：** Collector 尝试访问目标但失败了。Collector 通过 OTLP 抓取并推送，因此后端没有失败原因的记录。抓取该目标的 Collector 会将每次失败记录为 `Failed to scrape Prometheus endpoint`，并附带错误和目标标签：

```bash
kubectl -n nktp logs -l app.kubernetes.io/component=opentelemetry-collector \
  --prefix --max-log-requests 20 --since 10m \
  | grep 'Failed to scrape Prometheus endpoint' | grep '<service-name>'
```