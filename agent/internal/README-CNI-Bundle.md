# CNI 安装：`InstallCNI` 与 `ExecuteKubeadmInit`

本文说明 **agent** 在集群网络方面的行为：CNI 如何迁移到 `InstallCNI` 用于**新**编排，**`noCniInstall`** 如何**保持与旧版 controller/orchestrator 的向后兼容**，以及这与旧的 init 内 CNI 路径（独立的 `InstallCNIPlugin` RPC 已移除）的关系。

## `ExecuteKubeadmInit` 与 `noCniInstall`

`ExecuteKubeadmInit` 执行 `kubeadm init` 及相关设置。CNI 是否在该流程**内部**运行取决于 `KubeadmInitRequest` 上的 **`noCniInstall`**（字段省略时默认 **`false`**，与旧客户端一致）：

- **`noCniInstall` 省略或为 `false`（旧版 / 旧 controller）：** init 成功后，agent 可能走**旧的 init 内**路径：将 `KubeadmInitRequest` 上的 Tigera/Calico/OVN 字段映射到 `InstallCNIPlugin` 并调用相同的安装逻辑（除非设置了 `skipCniPluginInstall`）。此路径用于让从未传 `noCniInstall` 的**旧版 NVKE controller / orchestrator** 无需改代码即可继续工作。
- **`noCniInstall: true`（当前 controller / 推荐）：** init 期间不安装 CNI。之后通过 **`InstallCNI`** 安装集群网络。

`KubeadmInitRequest` 上的 CNI 相关字段（manifests、`cni_config`、`ovn_version`、`cni_plugin_name`、`tigera_version` 等）**仅在**旧的 init 内路径运行时**使用**（`noCniInstall` 不为 true 且未跳过）。新编排应设置 `noCniInstall: true`，留空这些字段，仅使用 **`InstallCNI`**。

## `InstallCNI`：先装前置 deb，再部署 charts

编排层负责安装 Debian 包（通常通过 **`InstallPackages`**）：

| `cni_variant` | Charts/manifest deb（**`InstallCNI`** 前控制平面必需） | 镜像 deb（通常所有节点——仅从镜像仓库拉取时可省略） |
|---------------|-----------------------------|------------------|
| OVN           | `nvke-cni-ovn-charts-<stream>`   | `nvke-cni-ovn-images-<stream>` |
| CALICO        | `nvke-cni-calico-manifests-<stream>` | `nvke-cni-calico-images-<stream>` |

`cni_variant` 为 **UNSPECIFIED** 时按 **OVN** 处理。**`version`**：去掉前导 **`v`**；用于选择 deb 命名中的 **`<stream>`**；**`version`** 为空时选择与 **cni-packages** 匹配的默认值（当前 OVN **`2.4.2`**、Calico **`3.30.1`**）。固定版本仍以 **`=…`** 形式附加到为日志和 **`resolvedAptPackage`** 生成的逻辑包名后。

所选 **`version`** 的 **charts/manifest** deb 存在后，**`InstallCNI`**（仅控制平面角色）会：

- 验证 **`dpkg`** 已注册 charts deb（否则返回 **`FAILED_PRECONDITION`**）。
- **不在 RPC 内执行 `apt install`**，也**不校验 containerd 中的容器镜像**——租户可完全依赖 **`imagePullSecrets` / kubelet 拉取**从私有镜像仓库获取镜像。
- 执行与之前相同的 Helm / manifest 应用逻辑：**OVN**（`helm install`）、**Calico**（视打包方式使用 Helm 或 `kubectl apply` / `apply-manifests.sh`）。

历史 proto 字段（**`cni_apt_role`**、**`apply_to_cluster`**）已废弃；**`CNI_APT_ROLE_IMAGES`** 会以 **`INVALID_ARGUMENT`** 拒绝。**`skip_apt`** 被忽略（此前为保留 / 未实现）。

该 RPC 将 **`resolvedAptPackage`** 设为根据 **`cni_variant`** + **`version`** 构造的 charts/manifest 逻辑包规格。

## 旧的 init 内路径（原 `InstallCNIPlugin`）

`InstallCNIPluginRequest` 保留为 init 内 CNI 的内部请求结构；独立 RPC 已移除。**新**代码的独立调用请优先使用 `InstallCNI`。当 `noCniInstall` 未设置时，它仍是**旧版 init 内 CNI** 的实现，用于**与旧 controller 版本向后兼容**。**新**编排应在 `ExecuteKubeadmInit` 上设置 **`noCniInstall: true`**，按需用 **`InstallPackages`** 安装 CNI Debian 包（镜像和/或 charts deb），并在控制平面上执行一次 **`InstallCNI`** 来应用 charts/manifests。

## 流程摘要（编排器视角）

**当前 NVKE controller（推荐）：**

1. 在引导控制平面节点上执行 **`ExecuteKubeadmInit`**，设置 **`noCniInstall: true`**。
2. 在需要预加载镜像的节点上用 **`InstallPackages`**（**或 apt**）安装 **`nvke-cni-*-images-<stream>`**（若租户策略为仅从镜像仓库拉取则跳过）。
3. 在**一个控制平面**上用 **`InstallPackages`** 安装 **`nvke-cni-*-charts-<stream>`** 或 **`nvke-cni-calico-manifests-<stream>`**。
4. 在**该控制平面**上执行一次 **`InstallCNI`**。
5. **加入流程：** 继续使用 **`ExecuteKubeadmJoin`**；在需要离线镜像包的仅加入节点上重复**步骤 2**（worker 上不执行 **`InstallCNI`**）。

**旧 controller/orchestrator（兼容）：** 可省略 `noCniInstall`，依赖默认的 init 内 CNI（对于仍填充旧 `KubeadmInitRequest` CNI 字段的调用方，行为与 `InstallCNI` 之前的 agent 相同）。