# 管理集群遥测：验证

Collector 可以处于 Ready 状态却什么都不导出，因此这份清单检查的是信号本身，而不只是 Pod。请在通过 `deploy/nico` 部署路径安装管理平面之后，或对其 NVKE 遥测流水线（NKTP）values 做任何改动之后运行本清单。它假定使用开发后端 `lgtm`。若使用你自己的后端，请在那里运行相同的查询。

在已有站点上，NVKE 控制器以其改名前的名称运行，即 `nke-controller`，或在更旧的站点上为 `cloud-workflow-engine`。请将本页中所有 `nvke-controller` 替换为该站点实际使用的名称。

所有命令都针对管理集群运行：

```bash
export KUBECONFIG=<site-kubeconfig>
```

## 1. 流水线已启动

```bash
kubectl -n nktp get pods                          # 每个 Pod 均为 Running，重启次数不再增长
kubectl -n nktp get opentelemetrycollectors       # 七个，每个 READY 1/1
kubectl -n nktp get certificates                  # 21 个，全部 Ready
kubectl -n nktp get configmap nktp-cluster -o jsonpath='{.data.name}'; echo   # <site-name>
```

这 21 个证书包括：Operator 的 webhook 证书；三个 Target Allocator 各自的 CA、服务端和客户端证书；以及 chart 的 `certManager.internalTLS` 默认值添加的 11 个内部 TLS 证书——一个 CA、四个服务端和六个客户端证书。[配置 cert-manager](../guides/configure-cert-manager.md) 列出了它们。

**Operator：** 它必须运行且无重启，其 finalizer 授权必须归 Helm 所有：

```bash
kubectl -n nktp get pods -l app.kubernetes.io/name=opentelemetry-operator
kubectl -n nktp get role,rolebinding nktp-operator-deployment-finalizers \
  -o custom-columns=KIND:.kind,MANAGED:.metadata.labels.app\\.kubernetes\\.io/managed-by
kubectl -n nktp logs deploy/nktp-operator | grep -ciE 'forbidden|blockOwnerDeletion'   # 0
```

**监控器：**

```bash
kubectl get servicemonitors,podmonitors -A
```

| 命名空间 | 监控器 |
|---|---|
| `nktp` | `nktp-apiserver`、`nktp-kube-etcd`、`nktp-coredns`、`nktp-node-exporter`，以及 PodMonitor `nktp-ksm-objects`、`nktp-ksm-pods`、`nktp-ksm-unscheduled-pods` |
| `temporal` | `temporal-frontend`、`temporal-history`、`temporal-matching`、`temporal-worker` |
| `spire-server` | PodMonitor `spire-server` |
| `spire-system` | PodMonitor `spire-agent` |
| `nvke-controller` | `nvke-controller-metrics` |

**资源指标 API**，Collector 自动扩缩器会读取它：

```bash
kubectl get apiservice v1beta1.metrics.k8s.io   # Available True
kubectl top nodes
kubectl -n nktp get hpa                         # 每个 HPA 都显示当前 CPU 和内存
```

## 2. 信号已到达

将后端端口转发到 localhost，并定义本页其余部分使用的两个 shell 辅助函数：

- **`promql '<query>'`** 对后端的 Prometheus 运行 PromQL 查询。
- **`logql '<query>'`** 对其 Loki 运行 LogQL 查询。

每个函数为每条结果打印一行：该序列的标识标签（`job`、`service_name` 或 `k8s_cluster_name`），然后是值。请在与端口转发相同的 shell 中运行它们。

```bash
kubectl -n nktp port-forward deploy/lgtm 19090:9090 13100:3100 13200:3200 --address 127.0.0.1 &
promql() { curl -fsSG http://127.0.0.1:19090/api/v1/query --data-urlencode "query=$1" | jq -r '.data.result[] | [(.metric.job // .metric.service_name // .metric.k8s_cluster_name // "value"), .value[1]] | @tsv'; }
logql() { curl -fsSG http://127.0.0.1:13100/loki/api/v1/query --data-urlencode "query=$1" | jq -r '.data.result[] | [(.metric.service_name // .metric.k8s_cluster_name // "value"), .value[1]] | @tsv'; }
```

抓取每 30 秒运行一次，导出是批量进行的，因此新安装后请等几分钟再运行这些查询。

### 指标

```bash
promql 'count by (job) (up == 0)'          # 为空：没有目标处于 down
promql 'count by (job) (up == 1)'
```

| Job | 说明 |
|---|---|
| `apiserver`、`kube-etcd`、`coredns` | 控制平面监控器 |
| `kubelet`、`kubelet-cadvisor` | 节点的 kubelet，由节点 Collector 抓取 |
| `node-exporter` | 节点 CPU、内存、文件系统和网络 |
| `nktp/nktp-ksm-objects`、`nktp/nktp-ksm-pods`、`nktp/nktp-ksm-unscheduled-pods` | 来自 kube-state-metrics 的对象状态 |
| `nktp-operator` | OpenTelemetry Operator，由 watchdog 抓取 |
| `nvke-controller` | 控制器 |
| `temporal-frontend-headless`、`temporal-history-headless`、`temporal-matching-headless`、`temporal-worker-headless` | Temporal 的服务端角色 |
| `spire-server/spire-server`、`spire-system/spire-agent` | SPIRE |

PodMonitor 的 job 为 `<namespace>/<monitor>`；ServiceMonitor 的 job 为 Service 的名称。还会出现 `otelcol-contrib`：它是开发后端自己的 Collector。

```bash
promql 'count by (k8s_cluster_name) (up)'                           # 几乎所有序列上都是 <site-name>
promql 'count(node_cpu_seconds_total)'                              # > 0
promql 'count(kube_pod_info)'                                       # > 0
promql 'sum by (service_name) ({__name__=~"otelcol_exporter_send_failed_.*", service_name=~"nktp-.*"})'   # 为空
```

少量序列没有 `k8s_cluster_name` 是正常的。开发后端抓取自身时不经过 NKTP。

### 日志

```bash
logql 'sum by (k8s_namespace_name) (count_over_time({k8s_namespace_name=~".+"}[15m]))'   # 每个有 Pod 的命名空间
logql 'sum by (service_name) (count_over_time({service_name=~"journald|auth|kube-apiserver"}[15m]))'
logql 'sum(count_over_time({service_name=~".+"} | k8s_event_reason=~".+" [15m]))'  # Kubernetes Events
logql 'sum by (k8s_cluster_name) (count_over_time({k8s_cluster_name=~".+"}[15m]))'       # <site-name>
```

- **`journald`** 是节点的 systemd journal。
- **`auth`** 是 `/var/log/auth.log` 或 `/var/log/secure`。
- **`kube-apiserver`** 是 API 服务器的审计日志。

日志类别可以通过 `nvidia_telemetry_class` 标签以相同方式检查：`system.auth`、`system.audit`、`k8s.control_plane` 和 `k8s.api.audit`。[日志分类](../components/node-collector.md#log-classification) 列出了每个类别涵盖的内容。`/var/log/audit/audit.log` 仅在运行 auditd 的地方存在，因此 `system.audit` 可能为空。

### 链路

控制器为每个 API 请求发送一个 span：

```bash
for i in 1 2 3; do curl -fsS -o /dev/null http://<site-ip>:30080/api/version; done; sleep 10
curl -fsSG http://127.0.0.1:13200/api/search --data-urlencode 'q={ resource.service.name = "nvke-controller" }' | jq '.traces | length'   # > 0
curl -fsS http://127.0.0.1:13200/api/search/tag/service.name/values | jq -r '.tagValues[]'   # 包含 nvke-controller 和 nktp-* Collectors
```

## 3. 站点其余部分不受影响

NKTP 的安装会改动其他组件的配置，因此请检查它们仍能正常工作：

```bash
curl -fsS http://<site-ip>:30080/readyz; echo                                        # 数据库和 temporal 正常
curl -sk -o /dev/null -w '%{http_code}\n' https://<site-ip>:30443/.well-known/openid-configuration   # 200
nvkectl get providers                                                                  # CONFIGURED
```

- **第一条：** 控制器的指标 NetworkPolicy 不得切断其 API。
- **第二条：** SPIRE 的 OIDC 发现必须仍能在站点 IP 上响应。
- **第三条：** CAPNICo 仍处于已配置状态。

## 4. 重复安装不改变任何东西

部署路径应当可以重复运行。记录 Pod 启动时间，用相同输入重新运行，然后比较：

```bash
kubectl get pods -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,START:.status.startTime > before.txt
# 重新运行部署路径
kubectl get pods -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,START:.status.startTime > after.txt
diff before.txt after.txt   # 没有 NKTP、Temporal 或 SPIRE Pod 重启
```

每次运行 Helm revision 都会增加；这是预期行为。