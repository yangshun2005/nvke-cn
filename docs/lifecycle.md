# Day 0 → Day 1 → Day 2 生命周期

NVKE 运维分为三个阶段，对应你对基础设施所做的操作：构建、配置和运行。

```
Day 0            Day 1                      Day 2
──────────────   ────────────────────────   ─────────────────────────────
引导管理集群      安装站点栈                  运维租户集群
和首个租户集群    和平台
```

## Day 0 — 引导

**目标：** 将一组已准备好的裸节点变为一个运行中的 Kubernetes 集群，用于承载管理平面。

`nvkectl bootstrap site` 完成此操作。它在 init 节点上创建一个临时 Kind 集群，通过 `nvke-agent` gRPC 驱动各节点执行 `kubeadm init` 和 `kubeadm join`，完成后销毁临时集群。最终得到：

- 一个 Kubernetes 集群（`<sitename>.kubeconfig`）
- 一个 etcd 加密密钥（`<sitename>-etcd-encryption-key.json`）——请立即将其移出引导主机

此时集群只是一个普通的 kubeadm 集群，未安装任何 NVKE 服务。

**Day 0 的前提条件：**
- Ubuntu 24.04 或 26.04 节点，且 `nvke-agent` 正在运行（参见[预置节点](site-bootstrap/pre-provisioned-nodes.md)或[自定义基础 OS](site-bootstrap/custom-base-os.md)）
- init 节点上安装 `nvkectl`、`nerdctl`、`kind`
- init 节点到所有节点的 15005 端口网络可达

**参见：**[管理集群引导](site-bootstrap/overview.md) · [你的第一个集群 — 真实站点](first-cluster.md#real-site)

## Day 1 — 配置

**目标：** 将裸集群变为一个可自管理、能创建租户集群的 NVKE 站点。

此阶段安装使集群成为 NVKE 站点所需的全部内容：

1. **部署 NVKE 组件栈** — 包括控制器、Temporal、PostgreSQL、SPIRE、NVKE 遥测流水线（NKTP）和 CAPNICo，按 `deploy/nico` 部署路径所列。`deploy/nvke-deploy.sh` 根据站点 spec 和栈配置安装，若从发布版本安装则还依据 NVKR 发布描述，是简短命令列表中的一步；你的 GitOps 工具也可交付相同内容。参见[在一台机器上端到端运行 NVKE](get-started/local-site.md)和[部署路径](site-bootstrap/deploy-repository.md)。
2. **配置 CAPNICo** — 部署路径已在第 1 步安装它；使用同一步渲染的 `nico-credentials` Secret 将其连接到你的 NICo 站点。运行 `nvkectl get providers` 确认 `nico: CONFIGURED`。
3. **将站点纳入 NVKE** — NVKE 团队注册站点并应用 Day 0 的 etcd 加密密钥。此步骤需与 NVKE 团队协调；联系你的 NVKE 入门对接人，提供站点的 kubeconfig 和 Day 0 的 etcd 加密密钥文件。
4. **创建首个租户集群** — `nvkectl create cluster -f cluster.yaml`。参见[你的第一个集群 — 路径 C，第 8 步](first-cluster.md#8-create-a-tenant-cluster)。

**Day 1 的前提条件：**
- 一个正常运行的 NICo 站点，且已为集群 spec 使用的实例类型分配容量
- Day 0 的站点 spec、保存管理平面输入的栈配置，以及从发布版本安装时的 NVKR 发布描述
- Day 0 的 etcd 加密密钥

**参见：**[你的第一个集群](first-cluster.md) · [NICo 文档](https://docs.nvidia.com/infra-controller/documentation/home)

## Day 2 — 运维

**目标：** 保持平台运行、向租户交付 Kubernetes 集群，并保持版本最新。

站点纳入后，Day 2 运维通过 NVKE 进行：

| 操作 | 方式 |
|---|---|
| 创建租户集群 | `nvkectl create cluster -f spec.yaml` |
| 列出租户集群 | `nvkectl get clusters` |
| 获取租户 kubeconfig | `nvkectl get kubeconfig <name>` |
| 添加节点池 | `nvkectl create nodepool -f spec.yaml` |
| 扩缩节点池 | `nvkectl patch nodepool <name> --cluster <cluster> --patch 'desiredNodeCount: N'` |
| 升级集群的 Kubernetes 版本 | `nvkectl patch cluster <name> --patch 'controlPlaneKubernetesVersion: "1.34.0"'` |
| 删除租户集群 | `nvkectl delete cluster <name>` |
| 检查基础设施提供商状态 | `nvkectl get providers` |

节点健康和故障修复策略由 [NVSentinel](https://github.com/NVIDIA/NVSentinel) 或你自己的故障修复系统处理——NVKE 提供生命周期 API；cordon、drain 和 SLO 策略由你执行。参见[节点生命周期](tenant-clusters/node-lifecycle.md)。

**Day 2 的前提条件：**
- 一个健康且容量可用的 NICo 站点
- 管理平面栈运行正常
- CAPNICo 处于 `CONFIGURED` 状态（`nvkectl get providers`）