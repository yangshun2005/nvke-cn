# 管理集群遥测：故障排查

在管理集群上启动 NVKE 遥测流水线（NKTP）时遇到的故障，及其原因和修复方法。用于判断是否出问题的检查项，见[验证](verification.md)。

## Operator 反复崩溃重启，报 "cannot set blockOwnerDeletion"

**症状：** `nktp-operator` 反复重启，带 `--wait` 的 `helm install` 始终无法完成。日志显示创建其 NetworkPolicy 被拒绝：
`cannot set blockOwnerDeletion if an ownerReference refers to a resource you can't set finalizers on`。

**原因：**
1. Operator 为自己的 Pod 创建 NetworkPolicy，其所有者是其 Deployment，并设置 `blockOwnerDeletion: true`。
2. NVKE 构建的每个集群都运行 `OwnerReferencesPermissionEnforcement` 准入插件（`controller/internal/kubeadmconfig` 中的 `defaultEnabledAdmissionPlugins`）。只有当请求方能够 `update` 所有者的 `deployments/finalizers` 时，该插件才允许该字段。
3. Operator 自身的安装清单授予了该权限。NKTP 打包的 OpenTelemetry Operator Helm chart 没有授予。

没有该插件的集群（包括 Kind）不会出现此问题。

**修复：** 站点 values 通过 `opentelemetry-kube-stack.opentelemetry-operator.extraObjects` 添加该授权。如果 Operator 仍然崩溃重启，检查 `extraObjects` 是否仍包含这两个条目。设置 `extraObjects` 的 values 文件会替换整个列表，从而丢弃该授权。

```bash
kubectl auth can-i update deployments.apps/nktp-operator --subresource=finalizers \
  -n nktp --as=system:serviceaccount:nktp:opentelemetry-operator   # yes
```

## 安装停在 nvke-base-config："`{{ .ClusterName }}` has no value"

**原因：** 安装没有站点名称。使用 `--sitespec` 时，站点 spec 的 `clusterName` 为空；不使用它时，通过 `--config` 传入的 BootstrapConfig 没有 `ClusterName`。
`nktp-cluster` ConfigMap 由它渲染，渲染器拒绝空值，而不是给所有信号打上空标签。

**修复：** 在站点 spec 中设置 `clusterName: <site-name>`，或在 BootstrapConfig 中添加 `"ClusterName": "<site-name>"`。

## nktp 项在构建 chart 依赖时失败

**原因：** 安装会在安装 chart 之前运行 `helm dependency build`。这需要访问 `open-telemetry.github.io`、`prometheus-community.github.io` 和 `kubernetes-sigs.github.io` 的出站流量。

**修复：**
- **允许该出站流量，** 或
- **提前构建依赖**，在路径安装所用的检出目录中运行 `make nktp-chart-deps`，或
- **从发布版本安装。** `nvke-deploy-local.sh install --release …` 将发布版本携带的打包 nktp chart 放到 `charts/` 下，且不会对归档运行 `helm dependency build`。

## Collector 不出现，或 Operator 的 webhook 失败

**原因：** cert-manager 尚未就绪，因此 Operator 的 webhook 证书或 Target Allocator 的 mTLS 证书未签发。

**修复：** 检查 `kubectl -n nktp get certificates` 和 cert-manager 自身的 Pod。在管理集群上，`clusterctl init` 会在部署路径运行前安装 cert-manager；如果被跳过，请运行它。

## 抓取目标宕机

来自[验证](verification.md#2-the-signals-arrive)的 `promql 'count by (job) (up == 0)'` 列出失败的 job。后端只接收 Collector 推送的内容，因此无法说明原因。抓取目标的 Collector 会记录每次失败，包含错误和目标标签：

```bash
kubectl -n nktp logs -l app.kubernetes.io/component=opentelemetry-collector \
  --prefix --max-log-requests 20 --since 10m \
  | grep 'Failed to scrape Prometheus endpoint'
```

| 目标 | 原因 | 修复 |
|---|---|---|
| `kube-scheduler`、`kube-controller-manager` | 在 NVKE 上监听 `127.0.0.1`，kubeadm 集群通常如此 | 站点 values 已关闭这些监控。确认覆盖没有重新开启它们。 |
| `kube-proxy` | OVN 集群不运行 kube-proxy | 站点 values 中也已关闭 |
| `kube-etcd` | etcd 仅在 `127.0.0.1` 上提供指标 | NVKE 设置 `listen-metrics-urls: http://0.0.0.0:2381`；其他环境请关闭该监控 |
| 刚消失的目标 | 其监控已删除，但 Prometheus 会保留最后一个样本最多五分钟 | 等待即可 |

## 没有 API 审计日志

**原因：**
- **路径错误：** 接收器读取的路径不存在。NVKE 将审计日志写入 `/var/log/kubernetes_audit/audit.log`，站点 values 将 `filelog/kube-api-audit` 指向那里。写入其他位置的集群需要相同的覆盖。
- **日志关闭：** API 审计日志已关闭，因此根本没有文件。

## 没有 journal 日志

**原因：** 节点将 journal 保存在 `/run/log/journal`（仅内存中），而不是 `/var/log/journal`。

**修复：** 挂载 `/run/log/journal`，并将 `journald` 接收器的 `directory` 指向它；见[配置](configuration.md#a-different-host-layout)。采集从新条目开始，因此空闲节点可能需要一段时间才会显示任何内容。

## 自动扩缩器不显示指标

**原因：** 没有服务提供 `metrics.k8s.io`，或两个提供者争抢它。

**修复：** 站点 values 开启 metrics-server，因为管理集群没有其他提供者。如果之后有其他东西安装了提供者，请关闭两者之一。`v1beta1.metrics.k8s.io` APIService 只有一个所有者，两个提供者会让它在两者之间来回切换。

## 指标被真实后端拒绝

**原因：** Target Allocator 0.159.0 将某些计数器导出为 delta，即使请求的是 cumulative 时序。

**修复：** 让后端接受 delta 指标，如 Prometheus 的 `otlp-deltatocumulative` 特性为开发后端所做的那样；见[配置](configuration.md#connecting-a-real-backend)。

## 手动升级后 SPIRE OIDC 发现拒绝按 IP 的请求

**症状：** 有人手动升级 `spire` 发布版本后，`https://<site-ip>:30443/.well-known/openid-configuration` 不再返回 200。

**原因：** 升级使用了带 `--reuse-values` 的 `deploy/nico` overlay。该 overlay 的 `spiffe-oidc-discovery-provider.config.additionalDomains` 列表替换了按站点设置的列表，而部署路径上次传入的列表包含站点 IP。

**修复：**
1. `helm rollback spire <last good revision> -n spire-server`
2. 仅用 `--set` 应用你本意的更改。

一般而言，通过部署路径更改站点，而不是逐个发布版本更改；部署路径最后传入按站点的 values。

## Chart 升级不会获取新的 CRD 字段

**原因：** chart 将 OpenTelemetry 和 Prometheus CRD 放在 `crds/` 中，Helm 只安装一次，从不升级它们。升级 chart 并提升 Operator 后，现有站点仍保留旧 CRD。API 服务器随后会静默丢弃新自定义资源中它们不认识的字段。

**修复：** 此类升级后，自行从新 chart 包应用 CRD：

```bash
helm template nktp <nktp-x.y.z.tgz> --include-crds \
  | yq 'select(.kind == "CustomResourceDefinition")' \
  | kubectl apply --server-side -f -
```

## 镜像仓库后仍有部分镜像从公共仓库拉取

**原因：** `global.imageRegistry` 将 Collector 和 Target Allocator 镜像移到镜像仓库，但三个镜像不读取它：metrics-server 的、OpenTelemetry Operator 的，以及 opentelemetry-kube-stack 卸载钩子的 kubectl 镜像。

**修复：** 在气隙站点上，通过它们各自的 values 将这三个指向镜像仓库：

```yaml
metrics-server:
  image:
    repository: <mirror>/metrics-server/metrics-server
opentelemetry-kube-stack:
  opentelemetry-operator:
    manager:
      image:
        repository: <mirror>/open-telemetry/opentelemetry-operator/opentelemetry-operator
  cleanupJob:
    image:
      repository: <mirror>/rancher/kubectl
```

## Chart 失败，报 `image tag ... must be a quoted string`

**原因：** 类似 `tag: 1.0` 的数字形式标签未加引号，YAML 将其读作数字。

**修复：** 加引号：`tag: "1.0"`。