# 管理集群遥测：配置

NVKE 遥测流水线（NKTP）Chart 的默认值面向多节点集群，且对 NVKE 一无所知。管理集群所需的差异化配置全部位于
[`deploy/nico/nktp/values.yaml`](https://github.com/dsx-ai-factory/nvke/blob/main/deploy/nico/nktp/values.yaml)，
由 `nktp` 项覆盖到 Chart 之上。本页说明它改了什么、为什么改，以及站点如何覆盖。

## 站点值改了什么

| 设置 | 原因 |
|---|---|
| **Operator 的 finalizer 授权**（`opentelemetry-kube-stack.opentelemetry-operator.extraObjects`） | 一个 Role 和 RoleBinding，授予 Operator 对其自身 Deployment 的 `deployments/finalizers` 的 `update` 权限。缺少它，Operator 在 NVKE 集群上会反复崩溃重启；见[故障排查](troubleshooting.md#the-operator-crash-loops-with-cannot-set-blockownerdeletion)。 |
| **watchdog、其 sidecar 和三个网关上的 `k8s.cluster.name`** | 从 ConfigMap `nktp-cluster` 读取的 `K8S_CLUSTER_NAME` 环境变量，该 ConfigMap 由 `nvke-base-config` 从 BootstrapConfig 的 `ClusterName` 渲染。Chart 的 `transform/cluster` 处理器读取它，而 Chart 仅在 `env` 列表未包含它时才从 `global.clusterName` 设置。NVKE 的 kubeadm 配置将每个集群都命名为 `kubernetes`，因此名称必须来自集群外部。 |
| **API 审计日志路径**（`filelog/kube-api-audit.include`） | NVKE 的 kubeadm 配置将 API 服务器的审计日志写入 `/var/log/kubernetes_audit/audit.log`，而非 Chart 的默认路径。 |
| **控制平面监控**（`kubeScheduler`、`kubeControllerManager`、`kubeProxy` 关闭） | NVKE 将调度器和控制器管理器绑定到 `127.0.0.1`，且 OVN 集群不运行 kube-proxy。若保持开启，它们会显示为永久宕机的目标。API 服务器、etcd 和 CoreDNS 监控保持开启且正常工作。 |
| **metrics-server 开启** | 管理集群上没有其他组件提供 `metrics.k8s.io`，而 Collector 自动扩缩器需要读取它。 |
| **规格** | 所有组件单副本，自动扩缩器从 1 开始，每个 Collector 和 Target Allocator 使用较小的 requests 和 limits，所有对象类型共用一个 kube-state-metrics Deployment 而非五个。按小型管理集群规模设定。 |

每个网关的容器内存限制为 1 GiB。其 exporter 队列容纳 128,849,018 序列化字节（该限制的 12%，约 123 MiB），而非继承基础 Chart 中为 4 GiB 容器设定的队列大小。`GOMEMLIMIT` 为 384 MiB，低于内存限制器 55% 的软拒绝阈值。序列化载荷在堆上可能占用其大小的数倍；更改容器限制时，请同时调整队列和运行时内存目标。

## 连接真实后端

默认情况下，所有 exporter 都指向 `nktp` 命名空间中的开发后端 `lgtm`。除评估外，请将它们指向你自己的后端并关闭 `lgtm`：

1. **为站点编写 values 文件。** 键列于
   [设置后端](../getting-started.md#set-a-backend)：
   watchdog、watchdog 的 sidecar 以及每个网关的 `otlp` exporter，外加 `otel-lgtm.enabled: false`。
2. **通过 `--values-for nktp=<file>` 传给部署路径。** 它在项自身的 values 之后应用，因此优先生效。
3. **凭据不要写入 values。** 将其放入 `nktp` 中的 Secret，从工作负载的 `kubernetes.env` 引用，并在 exporter 配置中通过 `${env:NAME}` 读取。

后端必须接受：

- **基于 gRPC 的 OTLP**，用于指标、日志和追踪；或将 exporter 切换为 `otlp_http` 后基于 HTTP。
- **Delta 指标。** Target Allocator 0.159.0 即使请求的是累积时间性，也会将部分计数器导出为 delta。开发后端使用 Prometheus 的 `otlp-deltatocumulative` 功能转换它们；基于 Prometheus 的后端也需要相同处理。
- **`k8s.cluster.name` 资源属性**，当多个站点导出到同一后端时，这是区分各站点数据的唯一依据。

## 安全地覆盖站点值

`--values-for nktp=<file>` 由 Helm 合并到站点值之上。这有两个后果：

1. **Map 合并，列表替换。** 设置列表中的一项会替换整个列表，包括 Chart 或站点值放入的每一项。站点值已经重述了 Chart 的几个列表，因为它们要向其添加内容：
   - 节点 Collector 的 `env`
   - watchdog、其 sidecar 以及每个网关的 `env`，其中携带 `K8S_CLUSTER_NAME`
   - `opentelemetry-kube-stack.opentelemetry-operator.extraObjects`，其中携带 Operator 的 finalizer 授权

   若要覆盖其中之一，请从 `nktp/values.yaml` 复制整个列表并在那里修改。否则你会静默丢弃集群名称或 Operator 启动所需的授权。
2. **站点专属值应放在站点专属文件中。** 这同样适用于其他每一项。部署路径通过 `--values-for` 最后传入站点自身的 values，例如 SPIRE OIDC `additionalDomains` 中的站点 IP。切勿仅用 `deploy/nico` overlay 和 `--reuse-values` 手动升级单个 release：overlay 的列表会替换站点专属列表。见[故障排查](troubleshooting.md#spire-oidc-discovery-rejects-requests-by-ip-after-a-hand-upgrade)。

## 不同的主机布局

节点 Collector 假定 NVKE 参考节点镜像的布局：

- journal 位于 `/var/log/journal`
- `/usr/bin/journalctl`
- `/etc/machine-id`
- 认证日志位于 `/var/log/auth.log` 或 `/var/log/secure`

对于仅将 journal 保存在内存中的节点，添加一个 `/run/log/journal` 主机卷挂载到 `/hostfs/run/log/journal`，并设置
`daemonset.otelcol.config.receivers.journald.directory: /run/log/journal`。卷列表会替换站点值的列表，因此请完整重述，如
[`nktp/hack/e2e-values.yaml`](https://github.com/dsx-ai-factory/nvke/blob/main/nktp/hack/e2e-values.yaml) 为 Kind 所做的那样。

## 扩容

站点值将 NKTP 按少量工作负载的小型管理集群规模设定。随着站点增长：

- **大目标池**承载 kube-state-metrics 和控制平面监控。对象数量增长时，同时提高 `large_target_metrics.kubernetes.resources` 及其 `GOMEMLIMIT`。
- **kube-state-metrics** 在站点值中是单个 `objects` Deployment。Chart 自身的默认将其拆分为五组（`workloads`、`networking`、`storage`、`configuration`、`cluster`），每组都可分片。
- **网关**根据 CPU 和内存自动扩缩至最多 128 个副本，因此其 limits 比副本数更重要。