# 管理集群遥测：入门

NVKE 遥测流水线（NKTP）采集管理集群的指标、日志和链路，并通过 OTLP 导出。它由 `deploy/nico` 部署路径安装，与管理平面其余部分相同。无需手动安装任何内容。

本指南介绍 NKTP 为管理集群提供什么、部署路径需要你提供什么，以及如何初步查看。本目录中的其他指南涵盖[配置](configuration.md)、[验证](verification.md)和[故障排查](troubleshooting.md)。

## 你将获得什么

[`deploy/nico`](https://github.com/dsx-ai-factory/nvke/blob/main/deploy/nico/README.md) 中的 `nktp` 项从同一检出安装 [NKTP chart](https://github.com/dsx-ai-factory/nvke/blob/main/nktp/deploy/charts/nktp/README.md)，安装到命名空间 `nktp`：

- **OpenTelemetry Operator** 及其 CRD，以及 Prometheus 监控 CRD（`ServiceMonitor`、`PodMonitor` 等）。
- **Collector**，每个都是 Operator 运行的 `OpenTelemetryCollector`：
  - 每个节点一个，用于 Pod 日志、systemd journal、主机认证和审计日志、kubelet 和 cAdvisor，以及调度到该节点的每个监控器
  - 每种信号一个网关（指标、日志、链路），即导出到后端的 OTLP 入口点
  - 一个大目标池，用于 kube-state-metrics 和控制平面监控器
  - 一个 Kubernetes Event 读取器
  - 一个看门狗，接收每个 Collector 自身的遥测
- **kube-state-metrics、node-exporter 和 metrics-server。** metrics-server 是站点唯一的 `metrics.k8s.io` 提供者，Collector 自动扩缩器读取它。
- **一个开发后端** `lgtm`：Grafana、Prometheus、Loki 和 Tempo 在一个 Pod 中，附带 NKTP 仪表板。数据保存在内存中，无身份验证，因此仅供评估。参见[连接真实后端](configuration.md#connecting-a-real-backend)。

部署路径还将管理平面其余部分接入其中：

| 组件 | 信号 | 方式 |
|---|---|---|
| NVKE controller | 指标、日志、链路 | ServiceMonitor、Pod 日志、OTLP 链路到节点 Collector；其指标 NetworkPolicy 允许 `nktp` |
| Temporal | 指标 | 每个服务器角色一个 ServiceMonitor，来自 Temporal 自己的 chart |
| SPIRE server 和 agent | 指标 | PodMonitor，来自 SPIRE 自己的 chart |
| API server、etcd、CoreDNS | 指标 | chart 的控制平面监控器 |
| API server | 审计日志 | 从节点读取，分类为 `k8s.api.audit` |
| 每个 Pod | 日志、状态 | 来自节点的 Pod 日志；来自 kube-state-metrics 的状态 |
| 节点 | 指标、journal | node-exporter、kubelet 和 cAdvisor、systemd journal |

所有导出的数据都带有 `k8s.cluster.name`，设为站点名称。

**不采集：**
- **cert-manager、NodeLocal DNSCache、OVN-Kubernetes 和 CAPNICo。** NKTP 不为其他组件提供监控器：每个服务的所有者提供自己的监控器。这四个尚未提供；参见[监控你的服务](monitoring-a-service.md)。
- **CAPI core 和 kubeadm 提供者，** 它们仅向持有 bearer token 的调用方提供指标。
- **调度器和控制器管理器，** 它们在 NVKE 集群上监听 `127.0.0.1`。
- **Postgres，** 它不运行指标导出器。

它们所有 Pod 的状态仍通过 kube-state-metrics 到达。

## 部署前

部署路径与管理平面其余部分一起安装 NKTP，因此其前提条件是路径自身的（[运行前](https://github.com/dsx-ai-factory/nvke/blob/main/deploy/nico/README.md#before-this-runs)），外加：

- **cert-manager。** `clusterctl init` 安装它。它签发 Operator 的 webhook 证书以及每个 Collector 与其 Target Allocator 之间的 mTLS 证书。
- **BootstrapConfig 中的 `ClusterName`**，通过 `--config` 传入。它成为每个信号上的 `k8s.cluster.name`。如果缺失，安装在 `nvke-base-config` 项停止：
  ```json
  { "ClusterName": "<site-name>", "Nico": { "...": "..." } }
  ```
- **运行安装的节点的出站访问：**
  - 到 `open-telemetry.github.io`、`prometheus-community.github.io` 和 `kubernetes-sigs.github.io`，因为安装在安装前从 `Chart.lock` 构建 chart 的依赖
  - 到 `docker.io`、`ghcr.io`、`quay.io` 和 `registry.k8s.io`，用于镜像
- **节点上的空间。** 站点值将 NKTP 规模定为小型管理集群。其 Pod 总共请求约 1.6 CPU 和 3.9Gi 内存，第一个节点之后每个节点额外约 125m 和 320Mi：节点 Collector 和 kube-state-metrics 的节点 Pod 在每个节点上运行。开发后端不设请求，但空闲时使用约 1.2Gi。

没有其他要求。部署路径创建 `nktp` 命名空间，使用节点 Collector 所需的 `privileged` Pod 安全级别，管理集群所需的每个设置都在 [`nktp/values.yaml`](https://github.com/dsx-ai-factory/nvke/blob/main/deploy/nico/nktp/values.yaml) 中。

## 安装顺序

`nktp` 紧接在 `capnico` 之后，在一切创建监控器的项之前：

```text
nvke-base-config -> spire-tpm-pubhashes -> spire-server-nodeport
  -> spire-oidc-provider-nodeport -> capnico -> nktp
  -> postgresql -> spire-crds -> temporal -> spire -> controller
```

- **为何在此位置：** chart 安装监控 CRD，`temporal`、`spire` 和 `controller` 项需要它们来创建监控器。
- **一个副作用：** 先安装意味着 NKTP 也会记录管理平面其余部分的启动。
- **代价：** `nktp` 安装失败会在此处停止路径。

## 初步查看

在管理集群上，使用其 kubeconfig：

```bash
export KUBECONFIG=<site-kubeconfig>
kubectl -n nktp get pods                       # all Running
kubectl -n nktp get opentelemetrycollectors    # seven, each READY 1/1
kubectl -n nktp get certificates               # all Ready
```

要打开 Grafana，端口转发后端。仅绑定到 localhost：Grafana 给匿名访客管理员权限。

```bash
kubectl -n nktp port-forward deploy/lgtm 3000:3000 --address 127.0.0.1
```

如果你通过跳板机工作，请通过 SSH 转发该端口，而不是在站点网络上绑定。**NKTP Health** 仪表板显示流水线本身是否健康。

### NVKE 参考仪表板

NVKE 还以 JSON 文件形式提供[参考 Grafana 仪表板](https://github.com/dsx-ai-factory/nvke/blob/main/controller/deploy/dashboards/README.md)。这些是调查站点的起点，不是完整的监控或告警设置。查看其 **Requirements and scope** 面板，检查你的后端是否采集所需指标，并根据运营需求调整仪表板和告警。空的可选面板可能意味着其指标未被采集。

要使用这些仪表板，打开 Grafana **Dashboards → New → Import**，上传其 JSON 文件（或粘贴 JSON），并在提示时选择 Prometheus 数据源。这些文件不会自动安装到管理集群。开发时可将它们导入捆绑的 `lgtm` Grafana，但站点运营请导入到你的[真实遥测后端](configuration.md#connecting-a-real-backend)：`lgtm` 将数据保存在内存中且无身份验证。完整目录和导入详情参见[仪表板 README](https://github.com/dsx-ai-factory/nvke/blob/main/controller/deploy/dashboards/README.md)。

完整检查清单参见[验证](verification.md)。

## 后续步骤

- [配置](configuration.md)：站点值改变什么、连接真实后端，以及安全地覆盖值。
- [监控你的服务](monitoring-a-service.md)：服务如何采集自己的指标。
- [验证](verification.md)：端到端检查每种信号。
- [故障排查](troubleshooting.md)：真实管理集群上见到的故障及其修复。
- [观测 NVKE 及其集群](../../operate/observe.md)：健康检查、控制器日志和指标，NKTP 补充而非替代这些。