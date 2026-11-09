# 节点 Collector

节点 Collector（`daemonset`）在每个节点上运行。它收集节点日志和指标，并转发工作负载发送给它的 OTLP 数据。

## OTLP 端点

节点 Collector 通过 gRPC 在 `nktp-daemonset-collector:4317`、通过 HTTP 在 `http://nktp-daemonset-collector:4318` 接收 OTLP 日志、指标和追踪。Service 仅将流量路由到发送方节点上的 Collector。若本地 Collector 未就绪，流量会失败，而不会到达其他节点。Collector 设置 `k8s.node.name`，并将每种信号转发到对应的网关。从其他命名空间发送时，需用 NVKE 遥测流水线（NKTP）命名空间限定 Service 名称。

## Pod 日志和指标

DaemonSet 从 `/var/log/pods` 读取 Pod 日志并发送到日志网关。专用 Target Allocator 仅将受监控的指标端点分配给同一节点上的 Collector。指标通过 OTLP/gRPC 发送到 `nktp-gateway-metrics-collector:4317`，这需要指标网关。默认不收集每进程主机指标。Node exporter 提供主机指标，kubelet cAdvisor 抓取提供每容器资源使用情况。参见[启用主机进程指标](../guides/enable-host-process-metrics.md)以启用 `hostmetrics` 进程抓取器。

每个 DaemonSet Collector 还会抓取自身节点上的 kubelet `/metrics` 和 `/metrics/cadvisor` 端点。参见[配置 kubelet 指标](../guides/configure-kubelet-metrics.md)以更改这些抓取、验证 kubelet 证书，或在 kube-prometheus-stack 旁运行 NKTP。

## Journal

DaemonSet 还会收集 info 优先级及以上的 systemd journal 条目并转发到日志网关。节点必须提供 `/usr/bin/journalctl` 和 `/etc/machine-id`，journal 位于 `/var/log/journal`。Collector 使用选择性只读主机挂载和 `SYS_CHROOT` 能力来运行主机可执行文件。`/lib64` 主机卷还挂载在 `/hostfs/usr/lib64` 以支持 Red Hat 库路径。收集从新条目开始。默认情况下，journal 游标不会在 Collector 重启后持久化。参见[为节点 Collector 启用日志检查点](../guides/enable-daemonset-log-checkpoints.md)。

在 `daemonset.otelcol.config.receivers.journald` 下覆盖 `root_path`、`journalctl_path` 和 `directory` 以适配不同的主机布局。相应更新 `daemonset.kubernetes.volumes` 和 `volumeMounts`，替换任一列表时保留其他挂载。对于运行时 journal，添加挂载在 `/hostfs/run/log/journal` 的 `/run/log/journal` 主机卷，并将接收器目录设置为 `/run/log/journal`。

默认审计和控制平面日志源参见[日志分类](#日志分类)。

## 日志分类

NKTP 使用 `nvidia.telemetry.class` 资源属性按来源对日志分类。来自已分类来源的所有记录都获得相同的类，无论其内容如何。该类会转发到日志网关，用于路由规则。

| 来源 | 分类 | `service.name` | 识别方式 |
| --- | --- | --- | --- |
| sudo 和 SSH journal 条目 | `system.auth` | `journald` | Journal 生产者标识为 `sudo` 或 `sshd`。 |
| Linux 认证文件 | `system.auth` | `auth` | 来自 `/var/log/auth.log` 或 `/var/log/secure` 的所有记录。 |
| Linux auditd | `system.audit` | `auditd` | 来自 `/var/log/audit/audit.log` 的所有记录，包括认证记录。 |
| Kubernetes 控制平面日志 | `k8s.control_plane` | 不由 NKTP 设置 | `kube-system` 中具有匹配 Pod 名称前缀和容器名称的 Pod，对应 `kube-apiserver`、`kube-controller-manager`、`kube-scheduler` 或 `etcd`。 |
| Kubernetes API 审计日志 | `k8s.api.audit` | `kube-apiserver` | 来自 `/var/log/kubernetes/audit/audit.log` 的记录。必须在 API 服务器上启用 API 审计日志。 |

其他本地收集的日志保持未分类。所有 journal 记录使用 `journald` 作为 `service.name`。共享认证文件使用 `auth`。服务名称随分类一起到达日志网关。

## 自身遥测

Collector 自身的指标、日志和追踪通过其[边车](../architecture/self-telemetry.md)发送到 watchdog。Target Allocator 遥测发送到 watchdog。每个 Collector 还会读取其节点上发布命名空间中 Target Allocator 的 stdout 和 stderr 日志，并通过 `logs/targetallocator` 发送到 watchdog。Operator manager 容器日志通过 `logs/operator` 发送到 watchdog。

## 配置

使用 `daemonset.otelcol.config` 配置 Collector，使用 `daemonset.kubernetes` 配置 Kubernetes 工作负载。Pod 日志挂载为只读，但主机文件权限要求 Collector 以 root 运行。文件读取器跟随新记录，默认不会在重启后持久化偏移量。参见[为节点 Collector 启用日志检查点](../guides/enable-daemonset-log-checkpoints.md)。替换其环境变量列表时保留 `GOMEMLIMIT` 和 `GOGC`。若列表省略了 `NODE_NAME`、`POD_NAMESPACE` 和 `OTEL_K8S_NODE_IP`，Chart 会添加它们。

节点 Collector 继承 `priorityClass.name`（默认 `nktp-telemetry-critical`，值为 `1000000000`，低于 system-critical Pod），包括其边车但不包括其 Target Allocator。使用现有类时设置 `priorityClass.create: false`，或覆盖 `daemonset.kubernetes.priorityClassName`（`""` 表示退出）；设置 `priorityClass.preemptionPolicy: Never` 以禁用抢占。

## Target Allocator

将 `daemonset.kubernetes.targetAllocator.enabled` 设为 `false` 会导致 Helm 渲染失败。

标记为 `nktp.nvidia.io/large-target: "true"` 的监控 CR 会被排除。未标记的监控仍符合条件。没有匹配 Collector 节点的目标保持未分配。

Target Allocator 默认将 `prometheusCR.denyFSAccessThroughSMs` 设为 `false`，以兼容引用本地 bearer token 或 TLS 文件的现有 ServiceMonitor 和 PodMonitor。引用的文件必须存在于每个抓取 Collector 容器中。编写监控时优先使用 Secret 或 CA ConfigMap 引用。启用 `denyFSAccessThroughSMs` 前，确保分配器选中的监控均未引用 Collector 容器中的文件。参见[限制监控文件访问](../guides/restrict-monitor-file-access.md)。

### 监控凭据

要授予分配器访问监控引用的 Secret 和 CA ConfigMap 的权限，列出包含这些监控及其引用对象的命名空间：

```yaml
daemonset:
  kubernetes:
    targetAllocator:
      prometheusCR:
        secretNamespaces: [metrics-monitors]
```

`daemonset.rbac.create` 默认为 `true`。对于每个显式列出的命名空间，Helm 创建一个 Role，授予 Secret 的 `get/list/watch` 和 ConfigMap 的 `get` 权限，并绑定到发布命名空间中的分配器 ServiceAccount。绑定遵循 `daemonset.kubernetes.targetAllocator.serviceAccount`。省略列表或设为 `[]` 不会创建命名空间 RBAC。设置 `daemonset.rbac.create: false` 以在外部管理这些授权。

授权覆盖每个列出命名空间中的所有 Secret。请使用监控作者可信的命名空间。Secret 和 CA 引用在监控所在命名空间中解析。命名空间必须已存在。`secretNamespaces` 控制 Secret 监视。需单独配置监控和命名空间选择器。