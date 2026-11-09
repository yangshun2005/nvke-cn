# docs

关于 NVKE 的全部文档，按你要做的事分组。部分文档放在其所描述的代码旁边，而非本目录，因此本页同时索引两处。

## 运维指南

已发布的运维指南，按阅读顺序排列。它涵盖的内容与下方的站点引导和租户集群页面相同，并将取代后者。

| 文档 | 内容 |
|---|---|
| [overview/what-is-nvke.md](overview/what-is-nvke.md) | NVKE 是什么、由哪些部分组成，以及从 Day 0 到 Day 2 各角色的职责 |
| [overview/supported-platforms.md](overview/supported-platforms.md) | 本版本支持的 Kubernetes 版本、网络、基础设施提供商、节点操作系统和工具 |
| [overview/preview-status.md](overview/preview-status.md) | 预览版支持的功能及其已知限制 |
| [get-started/quickstart-local.md](get-started/quickstart-local.md) | 在单台 Linux 主机上使用 mock-NICo 评估 NVKE |
| [get-started/local-site.md](get-started/local-site.md) | 在单台机器上端到端走完运维流程：引导容器节点、安装管理平面、在模拟 NICo 上创建工作负载集群 |
| [operate/plan-installation.md](operate/plan-installation.md) | 安装前需要决定和收集的事项 |
| [operate/build-node-images.md](operate/build-node-images.md) | 构建节点操作系统镜像、通过 HTTP(S) 托管其启动文件、校验下载内容，并注册以供置备 |
| [operate/build-and-verify-node-image.md](operate/build-and-verify-node-image.md) | 从 `os-image/` 构建参考节点镜像、启动它，并用 `nvke-os-image-check` 检查 |
| [operate/prepare-nodes.md](operate/prepare-nodes.md) | 将管理集群的节点从首次启动准备到满足 `nvkectl bootstrap site` 的要求 |
| [operate/bootstrap-management-cluster.md](operate/bootstrap-management-cluster.md) | 编写站点 spec 并运行 `nvkectl bootstrap site` |
| [operate/install-management-plane.md](operate/install-management-plane.md) | 在管理集群上安装 Cluster API 和 `deploy/nico` 部署路径，并向 NVKE API 准入网络 |
| [operate/provider-nico.md](operate/provider-nico.md) | 确认管理平面能向 NICo 请求机器、其凭据，以及集群请求所需的 NICo ID |
| [operate/api-authentication.md](operate/api-authentication.md) | 为 NVKE API 启用 OIDC 认证 |
| [workload-clusters/ovn-kubernetes.md](workload-clusters/ovn-kubernetes.md) | 构建、自托管和部署 OVN-Kubernetes 构建产物 |
| [workload-clusters/create.md](workload-clusters/create.md) | 创建工作负载集群 |
| [workload-clusters/access.md](workload-clusters/access.md) | 获取工作负载集群的管理员 kubeconfig、授予客户访问权限，并使用 `kubectl` 连接 |
| [workload-clusters/oidc.md](workload-clusters/oidc.md) | 让工作负载集群接受来自管理平面和你自己的 OpenID Connect (OIDC) 身份提供商的令牌，以及它们不得声明的身份 |
| [workload-clusters/node-pools.md](workload-clusters/node-pools.md) | 添加、扩缩容和移除节点池 |
| [工作负载集群 Kubernetes 证书续期](../controller/docs/certificate-renewal.md) | 定时和强制叶子证书续期、故障诊断和所有权校验的恢复 |
| [operate/observe.md](operate/observe.md) | NVKE 及其集群的指标、仪表盘、日志和状态 |
| [operate/troubleshoot.md](operate/troubleshoot.md) | 常见故障的症状、原因和修复方法 |
| [operate/debug-nicoidentity-credentials.md](operate/debug-nicoidentity-credentials.md) | 用 `kubectl` 读取 CAPNICo 对 `NicoIdentity` 的凭据检查：结果是否为最新、各原因的含义，以及结果为何过期或缺失 |

## 从这里开始

| 文档 | 内容 |
|---|---|
| [dsx-stack.md](dsx-stack.md) | NVKE 在 DSX 组件生态（NICo、AICR、CRE、Nodewright、NVSentinel）中的位置 |
| [concepts.md](concepts.md) | 后续文档使用的术语 |
| [first-cluster.md](first-cluster.md) | 通过 `nvkectl` 驱动到达运行中集群的三条路径 |
| [lifecycle.md](lifecycle.md) | Day 0 → Day 1 → Day 2 运维生命周期 |
| [architecture.md](architecture.md) | 各模块、集群请求如何流经它们，以及各部分部署在哪里 |

## 代码开发

| 文档 | 内容 |
|---|---|
| [getting-started.md](getting-started.md) | 本地开发环境搭建，以及各模块的构建和测试 |
| [development.md](development.md) | 提交 Pull Request 前要运行的检查，以及哪些检查无法在笔记本上复现 |
| [go-workspace.md](go-workspace.md) | Go 工作区、其中包含哪些模块，以及如何保持 `go.work.sum` 最新 |
| [coverage-policy.md](coverage-policy.md) | 收集哪些覆盖率、CI 如何处理它，以及为何没有阈值门控合并 |
| [adrs/20261014-adr-conventions.md](adrs/20261014-adr-conventions.md) | 关于 ADR 范围、位置、格式和生命周期的已接受决策 |
| [issue-triage.md](issue-triage.md) | Issue 如何分诊：`needs-triage` 和 `priority/*` 标签、由谁设置，以及自动化做什么 |
| [../CONTRIBUTING.md](../CONTRIBUTING.md) | 如何让变更通过评审并合并 |
| [../.github/workflows/README.md](../.github/workflows/README.md) | CI 如何接线，以及新工作流必须遵守的不变量 |

## 搭建站点

| 文档 | 内容 |
|---|---|
| [site-bootstrap/overview.md](site-bootstrap/overview.md) | 站点控制器如何引导 |
| [site-bootstrap/installation.md](site-bootstrap/installation.md) | 下载并安装 `nvkectl` |
| [site-bootstrap/configuration.md](site-bootstrap/configuration.md) | 描述站点的站点 spec 文件 |
| [site-bootstrap/pre-provisioned-nodes.md](site-bootstrap/pre-provisioned-nodes.md) | 使用已置备的主机 |
| [site-bootstrap/custom-base-os.md](site-bootstrap/custom-base-os.md) | 在参考镜像之外的基础操作系统上运行 |
| [site-bootstrap/deploy-repository.md](site-bootstrap/deploy-repository.md) | 部署仓库及其内容 |
| [site-bootstrap/infrastructure-providers.md](site-bootstrap/infrastructure-providers.md) | 在站点集群上安装 Cluster API 和基础设施提供商 |
| [../cli/docs/sitespec.md](../cli/docs/sitespec.md) | 站点 spec 文件的每个字段及其类型 |
| [../cli/docs/reference/README.md](../cli/docs/reference/README.md) | 每个 `nvkectl` 命令和标志，由 CLI 自身生成 |
| [../cli/docs/bootstrap-troubleshooting.md](../cli/docs/bootstrap-troubleshooting.md) | 引导失败时证据留在哪里、如何解读错误、常见故障以及何时重新运行 |

## 运行租户集群

| 文档 | 内容 |
|---|---|
| [tenant-clusters/node-lifecycle.md](tenant-clusters/node-lifecycle.md) | 节点在其生命周期中经历什么 |
| [tenant-clusters/kubernetes-api-access.md](tenant-clusters/kubernetes-api-access.md) | 运维人员、自动化和 Site Controller 服务如何访问并认证到租户 Kubernetes API |
| [tenant-clusters/certificate-renewal.md](tenant-clusters/certificate-renewal.md) | 定时和强制 T2 叶子证书续期、预检、终端验证和恢复 |
| [tenant-clusters/kubelet-serving-certificates.md](tenant-clusters/kubelet-serving-certificates.md) | Kubelet 服务身份验证、CSR 签名、安装和故障排查 |
| [../controller/docs/authenticating-to-the-api.md](../controller/docs/authenticating-to-the-api.md) | 认证到 API |
| [../controller/docs/configuring-api-authentication.md](../controller/docs/configuring-api-authentication.md) | 启用 API 认证，包括 `nvkectl` 的 RBAC |
| [design/README.md](design/README.md) | 设计记录和提案；状态在各文档顶部说明 |
| [design/byo-os.md](design/byo-os.md) | 提案：自带操作系统镜像 |
| [design/multi-oidc-provider-support-design.md](design/multi-oidc-provider-support-design.md) | 设计：多个 OIDC 提供商 |

## 运维站点

| 文档 | 内容 |
|---|---|
| [operations/upgrade.md](operations/upgrade.md) | 指向预览状态中的升级限制 |
| [operations/crd-group-migration.md](operations/crd-group-migration.md) | 在现有站点上将现有 CR 复制到 `internal.nvke.nvidia.com` 组 |
| [operations/label-domain-migration.md](operations/label-domain-migration.md) | 迁移到 `nvke.nvidia.com` 标签和注解域对现有站点意味着什么 |
| [site-operations/certificate-renewal.md](site-operations/certificate-renewal.md) | 面向 NVKE 集群运维人员的站点管理证书续期 |
| [operations/observability.md](operations/observability.md) | Temporal UI、CWE 日志、数据库健康和 CAPNICo 诊断 |
| [operations/credential-lifecycle.md](operations/credential-lifecycle.md) | OSS alpha 凭据清单、支持的操作流程、CA 限制、监控、审计和证据 |
| [../controller/docs/etcd-encryption-key-rotation.md](../controller/docs/etcd-encryption-key-rotation.md) | 现有 T2 密钥轮换、分阶段恢复和既有对象重写证明 |
| [../controller/docs/credential-monitoring.md](../controller/docs/credential-monitoring.md) | 实际过期检查、配置边界、指标和告警投递 |
| [../controller/docs/credential-audit.md](../controller/docs/credential-audit.md) | 版本化生命周期审计、手动证据工具、账本导出和保留限制 |
| [operations/faq.md](operations/faq.md) | 常见的引导和租户集群故障模式 |
| [nktp/site-telemetry/getting-started.md](nktp/site-telemetry/getting-started.md) | 管理集群遥测：NKTP 收集什么以及部署路径需要什么 |
| [nktp/site-telemetry/configuration.md](nktp/site-telemetry/configuration.md) | 站点的 NKTP 值改变什么、连接真实后端，以及安全地覆盖值 |
| [nktp/site-telemetry/monitoring-a-service.md](nktp/site-telemetry/monitoring-a-service.md) | 服务所有者如何让 NKTP 抓取服务的指标 |
| [nktp/site-telemetry/verification.md](nktp/site-telemetry/verification.md) | 安装后端到端检查每个遥测信号 |
| [nktp/site-telemetry/troubleshooting.md](nktp/site-telemetry/troubleshooting.md) | 管理集群上出现的遥测故障及其修复 |

## 运行 NVKE 遥测流水线（NKTP）

| 文档 | 内容 |
|---|---|
| [nktp/getting-started.md](nktp/getting-started.md) | 在任意集群上使用 NKTP：安装 chart、初步了解，以及通常需要修改的内容 |
| [nktp/guides/send-telemetry.md](nktp/guides/send-telemetry.md) | 面向服务所有者：工作负载的日志、OTLP 和 Prometheus 指标如何到达 NKTP，以及添加 sidecar Collector |
| [nktp/guides/enable-auto-instrumentation.md](nktp/guides/enable-auto-instrumentation.md) | 一个共享的 `Instrumentation`，向 Java、Node.js、Python 和 .NET Pod 注入 OpenTelemetry 代理 |
| [nktp/guides/enable-ebpf-instrumentation.md](nktp/guides/enable-ebpf-instrumentation.md) | 运行 OpenTelemetry eBPF Instrumentation (OBI)，以追踪任意语言的请求并发送到节点 Collector |
| [nktp/architecture/topology.md](nktp/architecture/topology.md) | 各 Collector、每种信号如何流向其后端，以及抓取如何分配 |
| [nktp/architecture/backpressure.md](nktp/architecture/backpressure.md) | Collector 饱和时队列和内存限制器的行为 |
| [nktp/architecture/self-telemetry.md](nktp/architecture/self-telemetry.md) | 缓冲每个 Collector 自身遥测的 sidecar，以及它如何采样追踪 |
| [nktp/guides/size-resources.md](nktp/guides/size-resources.md) | 默认请求和限制、运行时内存目标，以及容量规划指导 |
| [nktp/guides/configure-cert-manager.md](nktp/guides/configure-cert-manager.md) | NKTP 使用 cert-manager 做什么，以及不安装它时如何运行 |
| [nktp/guides/secure-gateway-export.md](nktp/guides/secure-gateway-export.md) | 网关和 watchdog 到后端的 TLS 和 mTLS |
| [nktp/guides/configure-kubelet-metrics.md](nktp/guides/configure-kubelet-metrics.md) | 更改 kubelet 抓取、验证 kubelet 证书，以及与 kube-prometheus-stack 并行运行 |
| [nktp/guides/restrict-monitor-file-access.md](nktp/guides/restrict-monitor-file-access.md) | 在启用文件访问限制前迁移监控凭据 |
| [nktp/guides/configure-persistent-queue.md](nktp/guides/configure-persistent-queue.md) | 让 watchdog 和网关导出器队列在重启后保留在 PersistentVolume 上 |
| [nktp/guides/enable-daemonset-log-checkpoints.md](nktp/guides/enable-daemonset-log-checkpoints.md) | 持久化节点 Collector 的日志读取位置，使重启后的 Collector 继续读取而非跳过行 |
| [nktp/guides/enable-host-process-metrics.md](nktp/guides/enable-host-process-metrics.md) | 按进程的主机指标、所需权限及其基数 |
| [nktp/components/node-collector.md](nktp/components/node-collector.md) | 节点 Collector（`daemonset`）：OTLP 端点、Pod 日志、journal、日志分类和监控凭据 |
| [nktp/components/gateways.md](nktp/components/gateways.md) | 日志、指标和追踪网关（`gateway_*`）及其端点 |
| [nktp/components/large-target-metrics.md](nktp/components/large-target-metrics.md) | 大目标池（`large_target_metrics`）、大规模抓取的容量规划，以及 Kubernetes 组件指标 |
| [nktp/components/cluster-events.md](nktp/components/cluster-events.md) | 集群事件读取器（`cluster_events`）及其领导者选举 |
| [nktp/components/watchdog.md](nktp/components/watchdog.md) | watchdog（`watchdog`），承载流水线自身的遥测 |
| [nktp/components/kube-state-metrics.md](nktp/components/kube-state-metrics.md) | kube-state-metrics（`kube-state-metrics`）：收集组、分片和可选的 Secret 指标 |
| [nktp/components/otel-lgtm.md](nktp/components/otel-lgtm.md) | 内置的开发后端（`otel-lgtm`）、其仪表盘和设置 |
| [nktp/components/dependencies.md](nktp/components/dependencies.md) | 内置的 Operator、CRD、node-exporter 和 metrics-server |
| [nktp/design-principles.md](nktp/design-principles.md) | 为何 chart 完全按所写内容渲染每个 Collector 配置 |
| [nktp/designs/monitor-file-access.md](nktp/designs/monitor-file-access.md) | 为何监控文件访问默认保持启用，以及如何限制它 |
| [nktp/designs/target-allocation.md](nktp/designs/target-allocation.md) | 为何每个抓取 Collector 都需要 Target Allocator |
| [nktp/designs/kube-state-metrics.md](nktp/designs/kube-state-metrics.md) | 为扩展 kube-state-metrics 收集而考虑的各种方案 |

## 各模块

每个模块都有自己的 README。`go.work` 是记录哪些模块存在的权威列表。

| 模块 | 文档 |
|---|---|
| Cloud Workflow Engine | [../controller/README.md](../controller/README.md) |
| 节点代理 | [../agent/README.md](../agent/README.md) |
| `nvkectl` | [../cli/README.md](../cli/README.md) |
| 生成的 API | [../controller/api/README.md](../controller/api/README.md) |
| `nico-sim` VM 和容器模拟器 | [../controller/nico-sim/README.md](../controller/nico-sim/README.md) |
| 参考节点操作系统镜像 | [../os-image/README.md](../os-image/README.md) |
| Cluster API 提供商 | [../cluster-api-provider-provisioned/README.md](../cluster-api-provider-provisioned/README.md) |
| CAPD 重启测试辅助工具 | [../controller/hack/c