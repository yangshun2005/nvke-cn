# 使用预置节点

> **这是哪条路径？** 本指南介绍在自备节点上进行引导——不使用 SPIRE 工作负载 API，不使用 TPM，`nvke-agent` 上使用 `--disable-mtls`。如果你的节点由 NVKE OS 镜像构建（预装 SPIRE agent，默认启用 mTLS），镜像契约见 [`os-image/SPEC.md`](../../os-image/SPEC.md)，完整流程见 [`docs/first-cluster.md`](../first-cluster.md#real-site) 的 Real Site 路径。

如果你已有预置并运行的节点，可以手动准备它们用于 tier-0 引导。

> **已测试平台：** 预置节点上的 tier-0 引导在 **Ubuntu 24.04 (Noble) 和 26.04 (Resolute)** 上测试。发布认证目标为 **Kubernetes 1.36**，`nvkectl generate sitespec` 默认写入 `1.36.3`，除非你设置 `cluster.k8sVersion`；参见[支持的平台和版本](../overview/supported-platforms.md)。Ubuntu 22.04 (Jammy) 已不再支持。

## 自引导模式：托管与非托管

引导后得到什么，取决于你的环境是否包含 [NVIDIA Infra Controller (NICo)](https://github.com/dsx-ai-factory/infra-controller) 站点，并通过 [`cluster-api-provider-nico`](https://github.com/dsx-ai-factory/cluster-api-provider-nico) 与 NVKE 集成。

> **注意：** 引导期间，NVKE 使用 `cluster-api-provider-provisioned` 作为临时 CAPI provider，从预置主机引导站点控制器集群。对于生产 day-2 运维（租户集群创建、节点增删、滚动升级），NICo 是受支持的基础设施 provider，通过 CAPNICo 集成。自托管路径是受支持的生产部署模型。非托管路径会产出一个可用的 Kubernetes 集群，但无法进行任何 NVKE day-2 运维——它不是受支持的 NVKE 部署。

| | 自引导，**自托管** | 自引导，**非托管** |
|---|---|---|
| **环境** | NICo 基础设施可用或将要部署 | 无 NICo 基础设施，也无部署计划 |
| **你的责任** | 提供包含 NVKE 及其依赖（NICo、Temporal、Postgres、SPIRE 及相关组件）的 SBOM | 只需提供在 Kubernetes 上运行工作负载所需的内容 |
| **引导后** | 在新集群上安装你的 SBOM，使 NVKE 能通过 NICo 驱动站点 | 直接使用该 Kubernetes 集群 |
| **Day-2 运维** | 支持——升级集群、增删节点、创建租户集群 | **不支持**——NVKE 没有基础设施 provider 来执行这些操作 |

**示例（自托管）：** 运维人员自引导一个站点控制器集群，然后安装 NVKE SBOM（NVKE、NICo、Temporal、Postgres、SPIRE）。集群变为自管理并支持 day-2 运维。

**示例（非托管）：** 团队在无 NICo 的环境中于节点上引导 Kubernetes。他们得到一个可用的集群，但不应期待 NVKE day-2 功能。

## `nvkectl bootstrap site` 做什么

在预置节点上，`nvkectl bootstrap site` 只有一项工作：把你准备好的节点变成 **Kubernetes 集群**。它不安装你的站点栈、GitOps 控制器或应用工作负载。

引导完成后：

1. **验证集群**健康（见[步骤 3](#step-3-verify-the-cluster)）。
2. **如果是自托管**，安装你的环境 SBOM（NVKE、NICo、Temporal、Postgres、SPIRE 等）。
3. **部署你的 GitOps 工具**——ArgoCD、Flux 或你用于 app-of-apps 层的任何工具。

## 前提条件

- 奇数个 Ubuntu 24.04 或 26.04 节点——1、3、5 个；偶数会被拒绝
- 对所有节点具有 sudo 权限的 SSH 访问
- 节点可访问互联网以下载软件包和构建产物
- 所有节点之间网络连通
- 如果是自托管，在安装管理平面之前，为集群默认 StorageClass 准备一个存储 provisioner。引导不安装任何存储 provisioner，而管理平面的数据库需要 40 GiB 卷
- 仅当你从私有镜像仓库拉取下面两个镜像时需要镜像仓库凭据，并在 init 节点上用 `nerdctl login` 登录；仅当该仓库是私有 `nvcr.io` 仓库时需要 NGC API key。在 init 节点上构建的镜像不需要
- 在 **init 节点**上：`kind` 二进制和已安装 containerd 的 `nerdctl`——`nvkectl` 用它们创建临时引导 Kind 集群。不支持 Docker。kind 声明其 nerdctl provider 最低要求 1.7.0；CI 验证的版本是 2.3.5
- `nvkectl` 侧载的两个镜像：`make -C cli images` 在 init 节点上构建它们。要改用镜像仓库中的镜像，传入 `--nvke-controller-image` 和 `--provisioned-provider-image`，或设置 `NVKECTL_NVKE_CONTROLLER_IMAGE`（例如 `nvcr.io/<org>/nvke-controller:edge`）和 `NVKECTL_CAPI_PROVISIONED_IMAGE`（例如 `nvcr.io/<org>/cluster-api-provider-provisioned:edge`）
- 在 **init 节点**上，`PATH` 中还需有：`helm`、`kubectl` 和 `clusterctl`。`nvkectl`
  在引导期间会调用这三者——Helm 用于 Postgres、Temporal、SPIRE
  和 NVKE controller 的 release，`kubectl` 用于清单，`clusterctl init` 用于
  Cluster API provider。运行开始前只检查 nerdctl；缺少 `kind` 会在
  针对残留引导集群的预检中失败，此时尚未创建任何东西；缺少 `helm`、`kubectl` 或 `clusterctl`
  会在引导中途暴露，此时 Kind 集群已经启动
- init 节点必须能访问 `go.temporal.io` 和 `spiffe.github.io`（Temporal
  和 SPIRE chart）、`github.com`（`clusterctl init` 和 cert-manager），以及引导镜像来源的
  镜像仓库——Docker Hub、`ghcr.io`、
  `registry.k8s.io` 和 `cgr.dev`——此外还有你自己的镜像仓库。
  `nvkectl get bootstrap-manifest` 列出这些 chart 和镜像。它不包含
  `clusterctl init` 安装的内容：Cluster API 和 cert-manager 来自
  `github.com`，其镜像来自各自的镜像仓库

如果目标是自托管集群，请提前准备环境 SBOM 和[部署仓库](deploy-repository.md)。

## 步骤 1：准备所有节点

在**所有节点**（join 节点和 init 节点）上运行这些命令：

### 安装 Kubernetes 组件

如何安装 kubeadm 取决于节点是否由 NVKE OS 镜像构建。

**由 NVKE OS 镜像构建的节点**——镜像在本地 apt 仓库 `/usr/lib/nvke/debs` 中暂存了带版本的 Kubernetes Debian 包，但不安装它们。在节点交给引导之前，通过 cloud-init `runcmd` 安装：

```yaml
runcmd:
  - DEBIAN_FRONTEND=noninteractive apt-get install -y nvke-kubeadm-distro-<k8s-version>
  - DEBIAN_FRONTEND=noninteractive apt-get install -y nvke-cni-ovn-images-<stream> nvke-cni-ovn-charts-<stream>
```

将 `<k8s-version>` 替换为与站点 spec 中 `cluster.k8sVersion` 匹配的版本，`<stream>` 替换为对应 Kubernetes 版本的 CNI stream。如果跳过此步骤，引导会跑满 30 分钟超时，并在 `NodePreflight` / `check_cloud_init_status` 处失败——错误信息不会提到 kubeadm。

**非 NVKE OS 镜像构建的节点**——按以下步骤从零安装 kubeadm 及所有必需的运行时依赖。

> **警告：** 如果跳过或部分完成以下步骤，`nvkectl bootstrap site` 会到达 `cluster state status=Provisioning`，然后静默等待完整的 `--timeout`（默认 30 分钟）后失败。日志不会有任何有用信息。以下所有步骤都是必需的。

```bash
# Create keyrings directory
sudo mkdir -p -m 755 /etc/apt/keyrings

# Add Kubernetes repository (replace v1.34 with your target minor version)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

# Add Docker repository (for containerd.io).
# Skip this block if containerd is already present on your nodes
# (e.g. installed via DOCA or a pre-baked OS image) — use whatever
# version is already there and proceed to the configuration step below.
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl containerd.io conntrack cri-tools
sudo apt-mark hold kubelet kubeadm kubectl
```

> **警告：** 仓库的次版本号（上面的 `v1.34`）决定安装哪个 `kubeadm`，且必须与引导配置中的 `cluster.k8sVersion` 匹配。`kubeadm` 拒绝创建比自身次版本旧超过一个次版本的控制平面。

### 配置容器运行时

containerd 默认禁用 systemd cgroup 驱动。kubeadm 需要它。没有这一步，kubelet 和 containerd 对 cgroup 管理不一致，控制平面永远不会就绪。

```bash
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

### 配置内核模块和 Sysctl

kubeadm 的预检要求 `br_netfilter`、`overlay` 和三个 sysctl。现在加载它们并持久化到重启后。

```bash
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Disable swap — kubeadm requires it
sudo swapoff -a
sudo sed -i '/\bswap\b/s/^/#/' /etc/fstab
```

### 安装 CNI 插件二进制

`kubernetes-cni`（作为 kubeadm 依赖安装）不会按 containerd 期望的布局填充 `/opt/cni/bin`。显式安装 containernetworking 插件：

```bash
CNI_VERSION="v1.6.0"
ARCH=$(dpkg --print-architecture)
sudo mkdir -p /opt/cni/bin
curl -fsSL "https://github.com/containernetworking/plugins/releases/download/${CNI_VERSION}/cni-plugins-linux-${ARCH}-${CNI_VERSION}.tgz" | \
  sudo tar -C /opt/cni/bin -xz
```

### 安装 Node-Agent

`nvke-agent` 将从 NVKE apt 仓库安装。以下命令适用于 Ubuntu 24.04 (Noble)；在 26.04 (Resolute) 上使用 `nvke-apt-resolute` 前缀和 `nvke-resolute` suite。

> **尚未发布——以下命令不可用。**
> `packages.nvidia.com/nvke` 既不提供 key 也不提供 suite：`curl`
> 返回 HTTP 404（`NoSuchKey`），因此 `apt-get install nvke-agent` 会失败。
> 在仓库存在之前没有 apt 安装方式。在此期间要准备节点，
> 从本仓库构建 agent：`make -C agent package`
> 交叉编译到 `agent/bin/linux_<arch>/nvke-agent`，并与
> `agent/scripts/agent-installer.sh` 打包，后者会放置二进制、systemd
> unit 和 `/etc/default/nvke-agent`。
> [`controller/hack/capd-kindest-node/`](https://github.com/dsx-ai-factory/nvke/tree/main/controller/hack/capd-kindest-node)
> 以这种方式为引导 e2e 的节点安装 agent，是最接近的完整示例。
> **预览版已知限制：** apt 仓库及其签名 key 未发布。
> `make -C agent deb-amd64` 或 `deb-arm64` 从检出构建
> `nvke-agent_<version>_<arch>.deb`，`apt-get install` 可从本地路径安装。

```bash
# Configure the NVKE apt repository
sudo curl -fsSL -o /etc/apt/keyrings/nke.asc \
  https://packages.nvidia.com/nvke/nvke-apt-public.key

echo 'deb [signed-by=/etc/apt/keyrings/nke.asc] https://packages.nvidia.com/nvke/apt nvke-noble main' | \
  sudo tee /etc/apt/sources.list.d/nke-apt.list

sudo apt-get update
sudo apt-get install -y nvke-agent

# Nodes prepared this way run no spire-agent, so the agent speaks plaintext gRPC.
# nvkectl bootstrap site must then be run with --disable-mtls too (Step 2).
echo 'AGENT_ARGS="--disable-mtls"' | sudo tee /etc/default/nvke-agent
sudo systemctl restart nvke-agent

# Verify agent is running
sudo systemctl status nvke-agent
```

> **警告：** 使用 `nvke-agent` **0.1.12 或更高版本**。旧版 agent 不实现 kubeadm 预检，配置问题只会在引导中途暴露。

> **明文模式。** agent 上的 `--disable-mtls` 会关闭 SPIFFE，因此
> `nvkectl bootstrap site` 也必须用 `--disable-mtls` 运行——否则 controller
> 会通过 mTLS 拨号 agent，永远连不上。`nvkectl` 将此模式标记为仅限开发。
> 对于认证路径——每个节点运行 `spire-agent`、CSV 中每台主机有 TPM EK 指纹、
> 以及来自 `NVKECTL_SPIRE_CA_CERT_FILE` 和 `NVKECTL_SPIRE_CA_KEY_FILE` 的 SPIRE CA——请遵循
> [你的第一个集群，Real Site](../first-cluster.md#real-site)。

### 安装 Open vSwitch（仅 OVN-Kubernetes）

如果 `cluster.cniName` 为 `ovn-kubernetes`，在每个节点上安装 Open vSwitch：

```bash
sudo apt-get install -y openvswitch-switch
```

NVKE 参考节点镜像使用 NVIDIA 的 DOCA 构建版 Open vSwitch，即 `doca-openvswitch-switch`，来自 DOCA apt 仓库。agent 接受任一包。

## 步骤 2：在 Init 节点上运行引导

**仅在 init 节点上**：

1. [安装 `nvkectl`](installation.md)。
2. 生成你的[站点 spec 文件](configuration.md)。在 CSV 中列出主机，让 `nvkectl` 写入 spec：

```bash
cat > hosts.csv <<'EOF'
name,ip,tpm_ek_thumbprint
cp-01,10.10.50.2,
cp-02,10.10.50.3,
cp-03,10.10.50.4,
EOF

nvkectl generate sitespec my-tier0-site --nodes hosts.csv > sitespec.yaml
```

结果无需编辑。本指南使用的明文模式下，`tpm_ek_thumbprint` 留空；`--disable-mtls` 会忽略它。CSV 未包含的内容用 `--set`——例如 `--set cluster.podCidr=10.244.0.0/16`。所有字段见 [configuration.md](configuration.md)，并对照你的硬件检查模板的 `systemReservedMemory` 默认值 `16Gi`：kubelet 会从容量中减去它，因此在较小的节点上没有任何可分配资源，节点永远不会达到 `Ready`。

3. 运行引导：

```bash
# Validate the site spec first (no nodes or clusters touched)
nvkectl bootstrap site -f sitespec.yaml --dry-run

# Run bootstrap. --disable-mtls matches the plaintext agents from Step 1.
NVKECTL_NVKE_CONTROLLER_IMAGE="nvcr.io/<org>/nvke-controller:edge" \
NVKECTL_CAPI_PROVISIONED_IMAGE="nvcr.io/<org>/cluster-api-provider-provisioned:edge" \
  nvkectl bootstrap site -f
```