# 核心概念

NVKE 文档假定你熟悉 Kubernetes 及其生态。本页是快速导览——不是教程——介绍你在这些文档中会遇到的概念和工具。每节都附有上游资源链接，供你深入了解。

如果你已了解 Kubernetes、ArgoCD 和 Helm，可跳过本页。

---

## Kubernetes

Kubernetes（K8s）是 NVKE 所基于的编排系统。它在多台机器上调度容器化工作负载、处理故障，并提供一致的 API 来管理计算、网络和存储。

在 NVKE 语境下，有两种 Kubernetes 集群：

- **管理集群** — 一个小型、高可用的三节点 K8s 集群，运行 NVKE 自身的管理服务（工作流引擎、数据库、安全）。由运维人员管理，租户不可见。
- **租户集群** — 由 NVKE 供应的 GPU 加速 K8s 集群，供租户运行 AI 工作负载。

**了解更多：** https://kubernetes.io/docs/concepts/overview/

---

## kubeadm

`kubeadm` 是用于引导 Kubernetes 集群的上游工具——它初始化控制平面、生成证书并加入节点。NVKE 通过 `nvke-agent` 在底层使用 `kubeadm` 来启动管理集群和租户集群。

你不需要直接运行 `kubeadm`，NVKE 会替你运行。但了解它的作用有助于阅读引导日志或排查节点加入失败。

**了解更多：** https://kubernetes.io/docs/reference/setup-tools/kubeadm/

---

## Helm

Helm 是 Kubernetes 的包管理器。应用以 *chart* 形式打包——即带模板化配置的、有版本号的 Kubernetes 清单集合。NVKE 自身的服务（NVKE 控制器、Temporal、Postgres、SPIRE、NVKE 遥测流水线（NKTP））以 Helm chart 形式安装，来自 NVKR 发布版本或其公共仓库。

`nvkectl` 引导集群后，`deploy/nvke-deploy.sh` 从 `deploy/nico` 部署路径安装它们，这是简短命令列表中的一步：每一项要么是带 values 的 chart，要么是普通清单。参见[在单台机器上端到端运行 NVKE](get-started/local-site.md)。

**了解更多：** https://helm.sh/docs/intro/using_helm/

---

## ArgoCD

ArgoCD 是 Kubernetes 的 GitOps 持续交付工具。它监视 git 仓库中的 Kubernetes 清单，使集群与其中声明的状态保持同步。当你推送变更时，ArgoCD 检测到偏差并协调集群。

NVKE 不要求使用 ArgoCD。部署路径是带 values 的 Helm chart 和普通清单，因此 GitOps 工具可以交付同样的内容并负责持续协调；[部署路径](site-bootstrap/deploy-repository.md#applying-it-another-way)列出了执行器需要做的事。

**了解更多：** https://argo-cd.readthedocs.io/en/stable/core_concepts/

---

## Temporal

Temporal 是持久化工作流编排平台。它执行长时间运行、容错的工作流——例如"供应一个三节点集群"或"逐个升级 200 个工作节点"——并自动重试失败步骤而不丢失进度。

NVKE 控制器是一个 Temporal worker。当你通过 NVKE 供应租户集群时，控制器向 Temporal 提交一个工作流来驱动整个生命周期。如果某一步中途失败，Temporal 会从该点恢复，而不是从头开始。

运维人员可以端口转发 Temporal 的 Web UI 来查看运行中的工作流或诊断故障。

**了解更多：** https://docs.temporal.io/concepts/what-is-temporal/

---

## PostgreSQL

控制器和 Temporal 将状态保存在 PostgreSQL 中。部署路径将其作为 `postgresql` 项安装，即 `postgres` 命名空间中的 Bitnami chart，使用集群默认 StorageClass 的持久卷。数据库密码在渲染部署路径时来自安装环境；仓库中没有任何 values 文件保存密码。

**了解更多：** https://www.postgresql.org/docs/

---

## Cluster API (CAPI)

Cluster API 是 Kubernetes 的子项目，提供用于供应和管理 Kubernetes 集群的声明式 API。你无需运行脚本来创建集群，而是创建 Kubernetes 自定义资源（`Cluster`、`Machine`、`MachineDeployment`），由控制器协调至期望状态。

NVKE 生产部署的主要 CAPI 基础设施提供者是 [`cluster-api-provider-nico`](https://github.com/dsx-ai-factory/cluster-api-provider-nico)（CAPNICo），它与 [NVIDIA Infra Controller (NICo)](https://github.com/dsx-ai-factory/infra-controller) 集成，用于分配和管理裸金属机器。可用的 NICo 站点是 NVKE 运维（Day-2）的前提——租户集群创建、节点增删和升级都经由 NICo。

**了解更多（CAPI）：** https://cluster-api.sigs.k8s.io/introduction

**NICo 文档：** https://docs.nvidia.com/infra-controller/documentation/home

---

## SPIFFE / SPIRE

SPIFFE（Secure Production Identity Framework for Everyone）是为工作负载签发加密身份的标准。SPIRE 是其参考实现。

在 NVKE 中，每个节点和每个工作负载都获得由 TPM 证明支持的 SPIFFE 身份。工作流引擎和 `nvke-agent` 使用这些身份通过 mTLS 相互认证——组件之间不传递静态凭据。文档中提到的"SPIFFE mTLS"指两个端点在交换数据前已相互验证对方的硬件根身份。

**了解更多：** https://spiffe.io/docs/latest/spiffe-about/overview/

---

## GitOps 与部署路径

GitOps 是将 git 仓库作为基础设施状态唯一可信来源的实践。你在 git 中声明期望状态，控制器使集群与之匹配。

NVKE 提供**部署路径** `deploy/nico`：一个按安装顺序列出 NVKE 服务的目录，每项要么是带 values 的 Helm chart，要么是普通清单。`nvkectl` 引导集群后，`deploy/nvke-deploy.sh` 应用它，这是简短命令列表中的一步。ArgoCD 等 GitOps 工具可以交付同样的内容并负责持续协调。格式见[部署路径](site-bootstrap/deploy-repository.md)，安装见[在单台机器上端到端运行 NVKE](get-started/local-site.md)。

---

## 无需了解的内容

NVKE 会替你处理以下事项——操作 NVKE 无需深入掌握：

- **kubeadm 内部机制** — NVKE 通过 `nvke-agent` 驱动
- **证书生命周期** — 由 `cert-manager` 和 `nvke-agent` 管理
- **etcd 运维** — 由 NVKE 管理，内置加密，并使用 `etcd-manager` 进行压缩和快照
- **CNI 配置** — NVKE 在管理集群引导期间配置 OVN-Kubernetes

---

准备好继续？从[架构](architecture.md)了解组件概览，或直接前往[管理集群引导](site-bootstrap/overview.md)搭建管理集群。