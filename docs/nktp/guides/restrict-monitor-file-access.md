# 限制监控文件访问

要启用 `denyFSAccessThroughSMs: true`，需先将分配器选中的每个监控中的文件引用替换为 Secret 或 ConfigMap 引用。两个分配器均默认 `false`；两个池的准备工作相同。关于 `denyFSAccessThroughSMs` 为何默认 `false`，以及启用后如何防止 Collector 挂载的凭据通过监控文件引用泄露，请参阅[理由与兼容性](../designs/monitor-file-access.md)。

## 准备监控

首先确定每个监控由哪个分配器负责。使用默认选择器时：

| 监控元数据 | 分配器 | Values 键 |
| --- | --- | --- |
| `nktp.nvidia.io/large-target: "true"` | Large-target | `large_target_metrics` |
| 标签缺失或不为 `"true"` | Node | `daemonset` |

Node 分配器还会排除 `app` 为 `kube-prometheus-stack-kubelet` 或 `prometheus-operator-kubelet` 的 ServiceMonitor。若自定义了监控或命名空间选择器，请用这些选择器确定归属。各池内的目标分配见[拓扑](../architecture/topology.md#metrics-scraping)。

启用限制前：

1. 清点**所有命名空间中的所有 ServiceMonitor**，包括其他 chart 安装的，并审计 PodMonitor。检查 `bearerTokenFile` 以及 `tlsConfig.caFile`、`certFile`、`keyFile`，包括继承的 scrape-class 设置。分配器选中的每个监控在启用限制前都必须不含文件引用。审计整个集群，以免后续标签或选择器变更引入不兼容的监控。
2. [创建凭据 Secret](#supply-a-bearer-token-through-a-secret)，包含目标接受的令牌，然后将 `bearerTokenFile` 替换为 `authorization.credentials`。对于 TLS 文件引用，通过 Secret 或 ConfigMap [引用服务证书的 CA](#replace-a-ca-file-with-a-configmap-reference)，客户端证书和密钥通过 Secret 引用。[完整的 vLLM 监控](#vllm-servicemonitor-before-and-after)展示了 bearer 令牌的改动。授予负责的分配器访问所引用对象的权限。Node 分配器见[监控凭据](../components/node-collector.md#monitor-credentials)。Large-target 分配器需单独配置其 `prometheusCR.secretNamespaces` 并提供命名空间 RBAC；chart 的自动凭据授予仅覆盖 Node 分配器。
3. 若其他 chart 无法配置这些引用，通过该 chart 的 values 禁用其生成的 ServiceMonitor，并维护具有相同目标选择、池标签、端口、路径和认证的替代监控。准备 large-target 池时，需包含[内置 Kubernetes 组件监控](../components/large-target-metrics.md#kubernetes-component-metrics)。
4. 在设置仍为 `false` 时验证替代监控能成功抓取。然后通过常规 NKTP Helm 升级为每个已准备的分配器[启用限制](#enable-the-restriction)。检查分配器日志中的[被丢弃的抓取任务](#recognize-a-denied-scrape-job)，并确认预期目标仍上报指标。

## 通过 Secret 提供 bearer 令牌

对于认证 Kubernetes 服务账号令牌并将授权委托给 Kubernetes 的目标（如 [Kubebuilder metrics server](https://book.kubebuilder.io/reference/metrics.html)），选择一个有权读取 `/metrics` 的 ServiceAccount。若 Collector 的 ServiceAccount 已有该权限，可复用它；否则按下文创建专用的 metrics-reader ServiceAccount。创建 Collector ServiceAccount 本身不会授予应用指标的访问权限。然后声明与该 ServiceAccount 关联的令牌 Secret；Kubernetes 会用该账号的令牌填充 Secret。在 ServiceMonitor 所在命名空间创建 ServiceAccount 及其令牌 Secret，以便监控引用该 Secret。该命名空间无需与 Collector 相同；下例使用 `metrics-monitors`。需要 `ClusterRole` 和 `ClusterRoleBinding`，因为 [`nonResourceURLs` 不属于命名空间](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)。采用其他认证方案的目标需由该目标签发并授权凭据。

对于 bearer 认证，引用的 Secret 键包含**实际令牌**，不含 `Bearer` 前缀及其后的空格，也不含文件系统路径。

NKTP 的凭据 RBAC 仅授予**分配器**访问 Secret 和 CA ConfigMap 的权限。chart 不创建示例中独立的 metrics-reader 身份、其目标权限或其令牌 Secret。请随应用配置一并管理这些资源。在当前 Kubernetes 版本上，仅创建 ServiceAccount 不会创建令牌 Secret。需声明类型为 `kubernetes.io/service-account-token`、带 `kubernetes.io/service-account.name` 注解的 Secret；令牌控制器会异步填充 `data.token`。参见 [Kubernetes 令牌 Secret 示例](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#manually-create-a-long-lived-api-token-for-a-serviceaccount)。

编辑命名空间和名称后应用以下资源。`metrics-monitors` 命名空间必须已存在。必须创建下文所示的带注解 Secret；令牌控制器会填充它而非创建它。省略其 `data` 字段，且不要将其加入 ServiceAccount 的 `secrets` 字段：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: example-metrics-reader
  namespace: metrics-monitors
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: example-metrics-reader
rules:
  - nonResourceURLs: [/metrics]
    verbs: [get]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: example-metrics-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: example-metrics-reader
subjects:
  - kind: ServiceAccount
    name: example-metrics-reader
    namespace: metrics-monitors
---
apiVersion: v1
kind: Secret
metadata:
  name: example-metrics-token
  namespace: metrics-monitors
  annotations:
    kubernetes.io/service-account.name: example-metrics-reader
type: kubernetes.io/service-account-token
```

Kubernetes 会向 Secret 添加令牌、ServiceAccount UID 注解以及 CA/命名空间数据。ServiceMonitor 直接引用其 `token` 键；无需复制令牌的命令，也无需在清单中填入凭据值。

这会创建一个**无内置过期时间的长效令牌**。Kubernetes 不会定期轮换它。（不同于 Collector 的投影 ServiceAccount 令牌文件，后者由 kubelet 自动轮换；此令牌 Secret 是长效的，不会自动轮换。）删除 ServiceAccount 会使其令牌失效并触发其令牌 Secret 的清理。Kubernetes 建议在支持自动续期时使用短效 TokenRequest 令牌；该方案需在过期前更新凭据 Secret。参见[管理 ServiceAccount](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#manual-secret-management-for-serviceaccounts)。

`type: Bearer` 告知 Collector 发送 `Authorization: Bearer <token>`。目标验证令牌并授权其标识的 ServiceAccount；认证类型本身不授予任何权限。如下方完整监控所示，使用 `authorization.credentials` 引用 Secret。

将 `metrics-monitors` 加入负责分配器的 `secretNamespaces`，并按上文授予其 Secret 访问权限。对于 TLS 认证，Secret 键必须分别包含 `tlsConfig.ca`、`cert` 或 `keySecret` 引用的 PEM CA 证书、客户端证书或私钥；同时用这些对象引用替换 `caFile`、`certFile` 和 `keyFile`。

## 用 ConfigMap 引用替换 CA 文件

kube-stack API-server ServiceMonitor 使用 `caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt`。将该路径替换为对**监控所在命名空间中现有 `kube-root-ca.crt` ConfigMap** 的引用，键为 `ca.crt`。其值是用于验证 Kubernetes API 服务器的 PEM 编码 CA 证书包，Kubernetes 也会将其投影到该路径的服务账号卷中。参见[管理 ServiceAccount](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)。此 ConfigMap 中不应包含令牌或私钥。

修改前，API-server 监控端点中：

```yaml
tlsConfig:
  caFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  serverName: kubernetes
  insecureSkipVerify: false
```

修改后：

```yaml
tlsConfig:
  ca:
    configMap:
      name: kube-root-ca.crt
      key: ca.crt
  serverName: kubernetes
  insecureSkipVerify: false
```

保留安装中的 `serverName`，它必须与 API 服务器的服务证书匹配。API-server 监控还需将 `bearerTokenFile` 替换为 Secret 支持的凭据；仅修改 `caFile` 不足以启用 deny 标志。在监控所在命名空间中授予分配器 ConfigMap 的 `get` 权限以及凭据 Secret 的访问权限。

对于其他目标，使用签发**该目标服务证书**的 CA。自定义 CA ConfigMap 的引用键必须包含其 PEM CA 包。Kubernetes 根 CA 适用于 API 服务器；它不会自动验证应用单独签发或自签名的证书。

## vLLM ServiceMonitor 修改前后

以下完整示例将 vLLM 上游监控适配到 `production-stack-system` 中的 Service 和 `metrics-monitors` 中的 ServiceMonitor。请编辑命名空间和 Service 标签选择器以匹配你的安装。按前述步骤在 `metrics-monitors` 中创建 `example-metrics-token` Secret。

修改前：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: vllm-operator
  namespace: metrics-monitors
spec:
  namespaceSelector:
    matchNames: [production-stack-system]
  selector:
    matchLabels:
      app.kubernetes.io/name: production-stack
      control-plane: controller-manager
  endpoints:
    - port: https
      scheme: https
      path: /metrics
      interval: 10s
      bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
      tlsConfig:
        insecureSkipVerify: true
```

修改后：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: vllm-operator
  namespace: metrics-monitors
spec:
  namespaceSelector:
    matchNames: [production-stack-system]
  selector:
    matchLabels:
      app.kubernetes.io/name: production-stack
      control-plane: controller-manager
  endpoints:
    - port: https
      scheme: https
      path: /metrics
      interval: 10s
      authorization:
        type: Bearer
        credentials:
          name: example-metrics-token
          key: token
      tlsConfig:
        insecureSkipVerify: true
```

两个版本均保留上游 vLLM 的 `insecureSkipVerify: true`，这会禁用服务器证书验证。要验证 vLLM 服务器的证书，需配置其服务证书并用 `insecureSkipVerify: false` 引用相应 CA；上述 API-server CA 示例仅在该 CA 签发了目标证书时适用。

## 启用限制

替换所选监控中的所有文件引用并验证其成功抓取后，通过 NKTP Helm 升级应用以下 values 覆盖。若一次只启用一个池，仅包含已准备好监控的组件：

```yaml
daemonset:
  kubernetes:
    targetAllocator:
      prometheusCR:
        denyFSAccessThroughSMs: true
large_target_metrics:
  kubernetes:
    targetAllocator:
      prometheusCR:
        denyFSAccessThroughSMs: true
```

## 识别被拒绝的抓取任务

若在替换监控的文件引用之前启用 `denyFSAccessThroughSMs`，分配器会丢弃其抓取任务并记录如下警告：

```json
{
  "level": "warn",
  "ts": "2026-10-20T21:04:51Z",
  "logger": "setup.prometheus-cr-watcher",
  "caller": "watcher/promOperator.go:687",
  "msg": "dropping scrape config that references arbitrary file path",
  "job": "serviceMonitor/metrics-monitors/vllm-operator/0",
  "reason": "authorization.credentials_file: /var/run/secrets/kubernetes.io/serviceaccount/token"
}
```

`job` 标识监控的命名空间、名称和端点索引。`reason` 显示与监控 `bearerTokenFile` 对应的生成的 Prometheus 字段。该任务在抓取前即被丢弃；抓取失败并显示 `up=0` 是另一种结果。