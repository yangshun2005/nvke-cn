# 提案：自带操作系统（BYO-OS）

> 状态：**提案** — 供讨论。描述从 NVKE 自建 OS 镜像转向由运维方负责的节点就绪契约，集群软件随后由 NVKE 控制器通过 `nvke-agent` 安装。

---

## 1. 摘要

目前 NVKE 将集群供给与节点 OS 的构建和引导方式紧密耦合：我们发布预装软件包的自定义 qcow 镜像，和/或在供给时通过 cloud-init 驱动软件包安装和 agent 启动（通过 `profiles/osProfiles/` 下的 OS 配置文件）。这通常包括 kubeadm、CNI 镜像包、Teleport 等——固化在镜像或首次启动的 user-data 中。

本提案将该关系重构为两个清晰阶段：

1. **BYO-OS（运维方负责）：** 节点满足最小就绪契约（Ubuntu 24.04、NVKE apt 仓库、`nvke-agent`、`spire-agent`、TPM）。*如何*实现不在 NVKE 范围内。
2. **供给后引导（NVKE 负责）：** 一旦 agent 可通过 SPIFFE mTLS 访问，控制器通过 `nvke-agent` 驱动其余工作——kubeadm 软件包、CNI、Teleport、集群身份配置及生命周期操作。

运维方可通过 Packer 构建的 qcow、首次启动的 cloud-init、实例创建后的 Ansible、黄金 AMI 或任何其他机制满足阶段 1。该选择**不在 NVKE 团队范围内**。

---

## 2. 动机

### 当前模型

控制器通过选择 **OS 配置文件** 来供给节点，该配置文件通常包括：

- 指向 NVKE 构建的 Ubuntu qcow 的 iPXE / 镜像 URL
- cloud-init `userData` 模板，从 NVKE apt 仓库安装软件包（`nvke-agent`、`spire-agent`、kubeadm 发行版软件包、CNI 镜像包、Teleport 等），写入 SPIRE / Teleport 配置，启用 systemd 单元，并发出就绪信号（如 `phone_home`、`/run/cloud-init.done`）

该路径可行，但使 NVKE 团队负责：

- 每个支持的 Ubuntu / 架构 / FDE 变体的镜像构建、发布和版本迭代
- 在运维方难以替换的 cloud-init 模板中编码引导策略
- 在基础镜像 / 首次启动路径中交付集群软件（kubeadm、CNI、Teleport），将 OS 来源与 Kubernetes 版本和 CNI 选择耦合
- 调试实为 OS / 打包 / 站点本地配置问题的供给失败

### 期望模型

将节点视为任何其他平台依赖：**发布最小就绪契约，验证它，然后作为 NVKE 引导步骤配置节点。**

| 阶段 | 负责方 | 职责 |
|-------|--------|----------------|
| OS / agent 上线 | **运维方** | 以任何方式满足 BYO-OS 契约 |
| 集群引导与生命周期 | **NVKE（控制器 → nvke-agent）** | 安装 kubeadm / CNI / Teleport 等，注入身份，运行 kubeadm，管理节点 |

NVKE 继续负责：

- agent gRPC API 及其安全模型（SPIFFE mTLS）
- 通过 agent 进行供给后软件包安装和集群编排
- 发布 apt 软件包并记录所需版本 / 仓库
- 可选的参考镜像或示例引导片段（非规范性）

NVKE **不**负责：

- Ubuntu 如何安装或加固
- 最小 agent 软件包是固化在镜像中还是创建后安装
- 站点本地工具（Packer、Ansible、Ignition、自定义 CI 等）

---

## 3. 目标与非目标

### 目标

1. 定义清晰、最小的**节点就绪契约**，控制器可在 NVKE 引导开始前检查。
2. 使 OS 构建**由运维方负责**；将集群软件安装保留为 agent 启动后的 **NVKE 负责**步骤。
3. 停止将 kubeadm、CNI、Teleport 及类似集群/运维软件包作为规范性路径放入基础镜像或 cloud-init 安装列表。
4. 保留当前 NVKE qcow + cloud-init 配置文件的支持路径，作为阶段 1 的一种*实现*，而非唯一实现。
5. 随时间减少 NVKE 镜像和 cloud-init 的表面积。

### 非目标

- 在 v1 中支持任意 Linux 发行版（仅 Ubuntu 24.04）。
- 用更弱的认证模型替换 SPIRE TPM 证明或 mTLS。
- NVKE 拥有或认可特定的镜像构建器 / 配置管理工具。
- 要求运维方在黄金镜像中预装 kubeadm、CNI、Teleport 或其他集群软件。

---

## 4. 节点就绪契约（阶段 1）

当以下所有条件为真时，节点**BYO-OS 就绪**。它们如何变为真与 NVKE 无关。

| # | 要求 | 说明 |
|---|-------------|--------|
| 1 | **Ubuntu 24.04** | 按节点池要求为 `amd64` / `arm64`。内核和用户空间必须是受支持的 24.04 LTS 基线。 |
| 2 | **NVKE apt 仓库已配置** | 源和签名密钥环存在，以便后续 `apt-get install`（来自 agent）可解析 NVKE 软件包。确切仓库 URL / 套件因环境而异，随构建产物发布流水线记录。 |
| 3 | **`nvke-agent` 已安装并运行** | systemd 单元处于活动状态；gRPC 监听配置的安全端口（默认 `15005`）；生产部署不得禁用 mTLS。 |
| 4 | **`spire-agent` 已安装并运行** | Workload API 可供 `nvke-agent` 使用。需要 TPM 证明时存在 TPM 节点证明器插件。 |
| 5 | **TPM 工具与启用** | 硬件或固件 TPM 2.0 对客户机/主机可见（视情况而定）；证明所需的用户空间工具可用（如 `tpm2-tools`）。FDE 注意事项见 §8。 |

### 明确**不**属于 BYO-OS 契约的部分

这些**不得**要求出现在运维方的基础镜像中。它们由 NVKE（控制器 → `nvke-agent`）随后安装和配置：

| 组件 | 为何供给后安装 |
|-----------|-------------------|
| **kubeadm / Kubernetes 发行版软件包** | 随集群而非 OS 镜像版本化。将其与黄金镜像耦合是长期痛点。 |
| **CNI 软件包 / chart 镜像**（OVN-Kubernetes 等） | 集群网络选择和版本属于集群配置，而非 OS。 |
| **Teleport**（及类似运维 agent） | 站点/环境策略由 NVKE 引导在 agent 可安全安装软件包和写入配置后应用。 |
| 其他集群范围软件包 | 任何版本或存在取决于集群配置文件而非“此节点能否与控制器通信”的软件包。 |

### 阶段边界

```text
┌─────────────────────────────────────────────────────────────────┐
│              阶段 1 — 运维方负责（BYO-OS）                       │
│                                                                 │
│  Packer/qcow  ──┐                                               │
│  cloud-init   ──┼──► Ubuntu 24.04 + apt 仓库 + nvke-agent        │
│  Ansible      ──┤      + spire-agent + TPM 就绪                 │
│  手动 / CI    ──┘                                               │
└───────────────────────────────┬─────────────────────────────────┘
                                │ 契约满足；agent 可达
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│     阶段 2 — NVKE 负责（控制器 → nvke-agent 引导）               │
│                                                                 │
│  • SPIRE 信任材料 / agent 配置（如尚未存在）                     │
│  • 安装 kubeadm 发行版软件包                                    │
│  • 安装 CNI 软件包                                              │
│  • 安装/配置 Teleport（站点启用时）                             │
│  • kubeadm init/join、CNI 部署、加密等                          │
└─────────────────────────────────────────────────────────────────┘
```

**运维方：** 基础 OS + 仓库 + `nvke-agent` + `spire-agent` + TPM 启用。

**NVKE：** 从“agent 启动”起的一切——目前常位于 cloud-init `runcmd` 中的软件包安装，以及集群生命周期。

运维方可自行应用额外站点配置；控制器仅要求节点能证明并通过 mTLS 通信，以便 NVKE 引导可继续。

---

## 5. 运维方实现模式（示例）

以下所有模式对**阶段 1**均有效。NVKE 不偏好任何一种。

| 模式 | 概要 |
|---------|--------|
| 预构建镜像 | Packer（或等效工具）固化 Ubuntu 24.04，配置 NVKE apt 仓库，安装 `nvke-agent` / `spire-agent` / TPM 工具。**不**固化 kubeadm、CNI 或 Teleport。 |
| 供给时安装 | 标准 Ubuntu cloud 镜像；运维方在创建后仅安装契约软件包。 |
| 创建后配置管理 | 最小镜像 + Ansible（或类似工具）在控制器引导前收敛契约软件包/服务。 |

---

## 6. 对控制器和 OS 配置文件的影响

### 当前

OS 配置文件是**镜像选择**和**引导策略**（iPXE 镜像 URL + 安装 kubeadm、CNI、Teleport、agent 等的 cloud-init）的主要载体。

### BYO-OS 下

OS 配置文件（或后续的“节点环境”对象）应演进为：

1. **指向运维方提供的镜像 / 启动模板**（或“已运行实例”注册），而非打包集群软件的 NVKE 自有 qcow URL。
2. **薄首次启动覆盖层**仅在不可避免处（主机名、网络、足以启动 mTLS 的 SPIRE 配置）——而非 kubeadm/CNI/Teleport 的完整软件包引导脚本。
3. **agent 后引导活动**在控制器工作流中：`ConfigureAptRepo`（如需要）、`InstallPackages` 安装 kubeadm / CNI / Teleport、写入 Teleport / 剩余 SPIRE 配置，然后现有 kubeadm / `InstallCNI` 流程。
4. **就绪门禁**断言阶段 1 契约——agent 可达性、`ExecuteNodePreflight`、SPIRE SVID 可用性——在阶段 2 软件包安装开始前。

NVKE 可继续为实验室和 nico-vms 提供**参考** OS 配置文件和示例 user-data。这些是阶段 1 的便利实现，非要求。参考镜像应偏好薄契约而非“大杂烩”软件包列表。

### 建议的就绪检查（控制器）

阶段 2 引导（kubeadm 软件包、CNI、Teleport、init/join）前：

1. 到 `nvke-agent` 的 TCP / gRPC 可达性（`NodeAgentReady`）
2. 可选 `ExecuteNodePreflight`（和/或运维方记录的哨兵）
3. mTLS 握手成功（双方均存在 SPIRE 签发的 SVID）

失败模式：将节点留在明确的“OS 契约未满足”状态，而非进入不稳定的 kubeadm 失败。

---

## 7. NVKE 仍发布的内容

BYO-OS 不意味着运维方从零发明软件包。

| 构建产物 | 负责方 | BYO-OS 下的角色 |
|----------|--------|-------------------|
| `nvke-agent` `.deb` | NVKE | **阶段 1** — 引导前节点必需 |
| `spire-agent`（NVKE 打包 / 启用 TPM 的构建，视情况） | NVKE / 平台 | **阶段 1** — 必需 |
| NVKE apt 仓库元数据 + 签名密钥 | NVKE | **阶段 1** — 必须配置仓库以便阶段 2 安装 |
| kubeadm / Kubernetes 发行版软件包 | NVKE | **阶段 2** — 就绪后通过控制器 → agent 安装 |
| CNI 软件包 / chart 镜像 | NVKE | **阶段 2** — 就绪后通过控制器 → agent 安装 |
| Teleport（启用时） | NVKE / 站点策略 | **阶段 2** — 通过控制器 → agent 安装/配置 |
| 参考 qcow / cloud-init 示例 | NVKE（可选） | 非规范性阶段 1 辅助 |
| 特定站点的生产黄金镜像 | **运维方** | 仅实现阶段 1 |

---

## 8. 注意事项：全盘加密（FDE）

FDE **不**能干净地融入“运维方带来任意 Ubuntu，我们 apt 安装其余”。磁盘布局、LUKS、解锁策略，通常还有镜像本身，都在 `nvke-agent` 可达**之前**决定。将 FDE 视为 BYO-OS 上的显式覆盖层，而非最小契约的隐式部分。

### 为何特殊

- 加密和分区发生在镜像写入 / 首次启动时（当前：qcow-imager、LUKS 密钥材料、OS 配置文件中的 LVM 布局）。
- 解锁可能依赖 TPM 密封、clevis/tang 或口令注入，必须在正常多用户引导*之前*工作。
- 运维方“标准 cloud 镜像”若无预期布局，不会仅因阶段 2 软件包安装成功就成为 FDE 节点。

### 建议立场

| 模式 | 期望 |
|------|-------------|
| **非 FDE BYO-OS** | 仅阶段 1 契约。NVKE 对 LUKS/LVM 布局无要求。 |
| **启用 FDE 的节点池** | 运维方（或 NVKE 参考镜像）必须提供已实现所需磁盘加密布局和解锁方案的镜像 / 启动路径。阶段 1 的 TPM 启用仍必要，但本身不充分。 |

NVKE 应为 FDE 节点池记录：

1. 所需磁盘布局 / 解锁方法（视配置文件而定）。
2. 控制器是否仍参与密钥注入，或仅消费已加密磁盘。
3. 哪些 TPM / clevis 软件包必须在阶段 1 存在 vs 后续安装（启动时解锁或证明所需的一切优先阶段 1）。

FDE 参考镜像可能比通用 BYO-OS 镜像更久地作为 NVKE 发布的构建产物，因为加密方案更难让运维方正确重新实现。这**不**会将 kubeadm/CNI/Teleport 放回那些镜像——FDE 镜像仍应瞄准薄的阶段 1 软件包集。

### 待决 FDE 问题

- FDE 是否始终为运维方提供镜像 + 记录的布局，还是 NVKE 无限期继续发布 FDE qcow 作为支持路径？
- 多少 LUKS/TPM 解锁自动化留在 iPXE / imager 中，多少在首次启动后移至 agent 之后？
- Clevis/tang vs 仅 TPM 解锁：哪些是 BYO-OS FDE 的一等公民？

---

## 9. 安全考虑

- **信任边界不变：** 具有有效 SPIFFE 身份的可达 `nvke-agent` 对该节点仍等效于 root。BYO-OS 不得削弱 mTLS 或 TPM 证明要求。
- **供应链：** 固化镜像的运维方负责验证 apt 仓库密钥和阶段 1 软件包来源。阶段 2 软件包由 NVKE 在 agent 启动后从同一签名 apt 仓库安装。
- **TPM：** 依赖 TPM 节点证明（或 FDE 解锁）的站点必须确保客户机有可用 TPM；实验室软件 TPM（如 nico-vms swtpm）可用于开发，除非明确接受，否则不能替代生产。
- **集群密钥：** 信任包、Teleport 加入令牌及类似内容不得固化在跨信任域复用的共享黄金镜像中；优先通过控制器 → agent（或等效安全通道）进行阶段 2 注入。

另见 `agent/docs/SECURITY.md`。

---

## 10. 迁移概要

1. **记录**此契约（本提案 → 接受的设计）。
2. **将** kubeadm / CNI / Teleport 安装从规范性 cloud-init `runcmd` 移至控制器 → `nvke-agent` 引导活动。
3. **在**控制器供给中**验证**阶段 1（显式条件 / 预检），不移除现有 OS 配置文件。
4. **精简**参考镜像和 OS 配置文件至阶段 1 软件包集；按需保留 FDE 参考镜像（§8）。
5. **提供** BYO-OS 作为早期运维方的可选节点池 / 配置文件模式。
6. **弃用** NVKE 对站点特定非 FDE 黄金镜像流水线的所有权，一旦参考 + 契约足够；保留 nico-vms / CI 镜像作为测试夹具。

无需大爆炸式切换。当前 qcow + cloud-init 路径仍是满足阶段 1 的有效方式，同时阶段 2 工作落地。

---

## 11. 待决问题

1. **版本锁定：** 阶段 1 要求精确软件包版本（`nvke-agent=…`）还是仅“≥ N / 来自通道 X”？
2. **首次启动的 SPIRE 配置：** 使 mTLS 在阶段 2 前工作的最小覆盖层（信任包 / 服务器地址）——cloud-init vs agent 优先的先有鸡还是先有蛋？
3. **注册 API：** 运维方仅向控制器提供镜像 ID，还是在不使用 cloud-init 时也提供“引导完成”webhook / phone_home 等效？
4. **Ubuntu 22.04：** 一旦仅 24.04 成为契约，任何剩余 22.04 配置文件的日落时间线？
5. **FDE：** 见 §8 待决问题。

---

## 12. 决策请求

批准以下方向：

1. NVKE 指定 **Ubuntu 24.04 + NVKE apt