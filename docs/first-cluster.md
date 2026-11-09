# 你的第一个集群

NVKE 是一个管理平面。一个**管理集群**——一个持久的 Kubernetes 集群，也称为 cell——按需铸造**租户集群**，每个租户集群都有自己的控制平面、etcd 和节点。`nvkectl` 是你驱动它的方式。

三条路径——选择与你现有条件匹配的那条：

| 路径 | 适用对象 | 需要 | 得到 |
|---|---|---|---|
| [**真实站点**](#real-site) | 搭建生产 NVKE 站点的**运维人员** | 已镜像的节点、nerdctl、一个 NICo 站点 | 一个管理集群，铸造由 CAPNICo 支持的租户集群 |
| [**Mock NICo**](#mock-nico) | 在没有硬件的情况下测试完整创建集群循环的**贡献者** | Docker、kind、约 16 GB 可用空间，以及 CAPNICo 检出或 NGC 账户 | 针对真实 CAPNICo 的真实创建集群循环，无需硬件 |
| [**仅 API**](#api-only) | 探索 CWE REST API 的**贡献者** | Go、Docker、kind | 几分钟内运行 CWE 并探索其 API。无租户集群。 |

## 开始之前

```bash
git clone https://github.com/dsx-ai-factory/nvke.git
cd nvke
```

| 工具 | 用于 | 备注 |
|---|---|---|
| [Go](https://go.dev/dl/) 1.27.3+ | 全部 | `go.work` 固定了它。较旧的 Go 会拉取正确的工具链，除非 `GOTOOLCHAIN=local` |
| [Docker](https://docs.docker.com/get-docker/) 24+ | 仅 API、Mock NICo | 不支持用于真实站点引导 |
| [nerdctl](https://github.com/containerd/nerdctl) 配合 containerd | 真实站点 | 引导通过它驱动 kind 并拉取镜像；无需 Docker 守护进程 |
| GNU Make 4.0+ | 全部 | macOS 自带 3.81；`brew install make` 提供 `gmake` |
| [temporal](https://docs.temporal.io/cli) | 仅 API | `brew install temporal`，或 `curl -sSf https://temporal.download/cli.sh \| sh` |
| [kind](https://kind.sigs.k8s.io/)、`kubectl` | 全部 | 真实站点：`nvkectl bootstrap site` 运行两者 |
| `jq` | Mock NICo | 没有它 `dev-check` 退出码为 1 |
| [helm](https://helm.sh/) | Mock NICo、真实站点 | `nvkectl bootstrap site` 为 Postgres、Temporal、SPIRE 和 CWE 发布版本运行它 |
| [tilt](https://docs.tilt.dev/install.html) | Mock NICo | |
| [clusterctl](https://cluster-api.sigs.k8s.io/user/quick-start) | Mock NICo、真实站点 | `make capd-up` 将其安装到 `controller/.local/bin`；`nvkectl bootstrap site` 运行 `clusterctl init` |
| `dpkg-deb` | Mock NICo（macOS） | `brew install dpkg`；bundle 镜像构建需要它 |
| [Flox](https://flox.dev/) | 可选 | 固定整个工具链；从不必需 |

macOS 或 Linux；不支持 Windows。

**在 Kind 或 Tilt 之前，先准备一次 Docker 主机。** 在 macOS 上，使用 Colima 而非 Docker Desktop。`controller/Makefile.dev` 将每个 `kind`、`docker` 和 `clusterctl` 调用都指向 Colima socket，因此 `capd-up` 无法在 Docker Desktop 下工作。普通的 `colima start` 会给出 2 CPU、2 GiB 的 VM，而此命令随后会拒绝调整其大小。在 Linux 上，同一命令会在主机内核上加载 `tpm_vtpm_proxy`，容器节点需要它来提供软件 TPM。

```bash
make -C controller host-up
```

**仅 API** 不需要任何内部资源，也不需要 NGC 账户。

**Mock NICo** 以两种方式之一安装 CAPNICo，**只有其中一种无需 NGC 账户即可工作**。将 `CAPNICO_SOURCE_DIR` 设置为 [`dsx-ai-factory/cluster-api-provider-nico`](https://github.com/dsx-ai-factory/cluster-api-provider-nico) 的检出（该仓库是公开的），它便从源码构建。不设置它，则改为从 NGC 安装已发布的 chart，这需要 `NGC_API_KEY`。如果你不在 NVIDIA 内部，应选择源码路线。

## 仅 API

CWE 针对本地 Postgres 和 Temporal 运行，使用自签名令牌而非 NGC 认证。无租户集群，无硬件。

```bash
# 任何集群都可以。CWE 会监视一个集群，没有 kubeconfig 会在启动时退出。
kind create cluster

# Postgres、Temporal，然后是 CWE。
make -C controller run-local
```

- Swagger — http://localhost:8080/api/docs/（保留末尾斜杠）
- `curl http://localhost:8080/api/clusters` → `{"clusters":[]}`。此处无需令牌。
- Temporal — http://localhost:8233

## Mock NICo

与真实站点运行相同的栈：真实 CAPNICo，由 **nico-mock** 应答 NICo API，租户节点作为容器。

Tilt 默认安装已发布的 CAPNICo chart，它从 NGC 拉取，因此需要 `NGC_API_KEY`。若要从检出构建 CAPNICo 且无需 NGC 账户，请在 `controller/tilt_config.json` 中设置 `capnico_source_dir`，或导出 `CAPNICO_SOURCE_DIR`。

如果你先运行了**仅 API**，请停止它——它的 Postgres 和 Temporal 占用 5432 和 7233，Tilt 的探针会针对错误的服务器通过，而其端口转发会永远重试：

```bash
make -C controller dev-stop-local
kind delete cluster            # 仅 API 中的普通集群
```

```bash
cp controller/tilt_config.json.example controller/tilt_config.json

# 凭据。缺少全部五个时 Tilt 会快速失败；本地值即可。
cp controller/.env.tilt.example controller/.env.tilt
# 然后编辑 controller/.env.tilt 并填写五个空值

# 仅当 CAPNICo 检出不是本仓库的兄弟目录时。绝对路径：
# Tilt 从 controller/ 运行 provider 脚本，因此相对路径会在仓库内解析。
export CAPNICO_SOURCE_DIR="$PWD/../cluster-api-provider-nico"

# kind 集群、Cluster API，以及节点镜像构建读取的 bundle 镜像。
# Tilt 检查这三者，缺少则拒绝启动。
make -C controller capd-up

cd controller && tilt up      # 保持运行
```

该配置选择 `["capi_core", "bgp-containers", "nico"]`，即 [NICo E2E 通道](../.github/workflows/nico-e2e.yml)运行的列表。单个 `nico` provider 启动 nico-sim，并在 `capnico-system` 中安装一个 CAPNICo 控制器。

[`controller/README.md`](../controller/README.md) 完整介绍了 Tilt 循环。

### 用 nvkectl 驱动它

将 CLI 指向管理平面：

```bash
go build -o bin/nvkectl ./cli/cmd/nvkectl

bin/nvkectl config set-site --name local \
  --url http://localhost:8080 \
  --kubeconfig ~/.kube/config
bin/nvkectl config view
```

`--kubeconfig` 是管理集群。`get providers` 直接读取它，CWE API 命令（`get`/`create`/`patch`/`delete` clusters）在认证开启时用它为 `nke-clients/nke-client` 铸造 bearer 令牌。Tilt 在 `controller/deploy/k8s/nvke-auth-dev.yaml` 中应用该身份。你无需为 `nvkectl` 导出令牌。

在 Tilt 下，API 需要令牌，与**仅 API** 不同——直接 `curl` 到 `/api/clusters` 返回 401。`nvkectl get clusters` 则不需要，一旦 `set-site` 有了 kubeconfig：它从 `nvke-controller-config` ConfigMap 中的 `apiAuth` 发现 audience（当该部分关闭时回退到 `nvke-auth-config`），并通过 TokenRequest API 铸造。

```bash
bin/nvkectl get providers      # 等待 nico: CONFIGURED
bin/nvkectl create cluster -f cli/examples/nico-mock-cluster.yaml
bin/nvkectl get clusters
bin/nvkectl get clusters <name>
bin/nvkectl patch cluster my-cluster --patch 'controlPlaneKubernetesVersion: "1.31.0"'
bin/nvkectl patch cluster my-cluster --patch-file patch.yaml
bin/nvkectl delete cluster <name>
```

[`cli/examples/nico-mock-cluster.yaml`](../cli/examples/nico-mock-cluster.yaml) 可直接使用——其 UUID 是 nico-mock 的种子标识符。

集群顶层的 `nico` 或 `infra` 载荷设置其 provider，控制平面和每个节点池必须携带相同的那个。**Mock NICo** 使用 `infra`，直接提供 `NicoCluster` 和 `NicoMachineTemplate` 清单；真实站点使用 `nico`，由 CWE 构建它们。参见[控制平面配置](#control-plane-configuration)；完整 schema 见 `/api/docs`。

对于 `curl`，在栈启动后用 `hack/nvke-token.sh` 铸造令牌：

```bash
TOKEN="$(controller/hack/nvke-token.sh)"
curl -H "Authorization: Bearer ${TOKEN}" http://localhost:8080/api/clusters
```

`NKE_AUTH_ENABLED=false tilt up` 关闭认证。此时 `nvkectl` 和 `curl` 都不需要令牌——`nvkectl` 发现 `apiAuth` 已禁用且没有 `nvke-auth-config` ConfigMap，便以未认证方式发送请求。

### 获取租户 kubeconfig

```bash
nvkectl get kubeconfig <cluster> --output-file tenant.kubeconfig
```

这会读取 `nvke-system` 中的 `<cluster>-admin-kubeconfig` Secret，其中包含 `nvke:breakglass` 管理员 kubeconfig。旁边的 `-kubeconfig` Secret 是 CWE 消费的那个，当 OIDC 开启时它会变成 SPIRE exec-auth kubeconfig。

## 真实站点

真实硬件、真实 NICo 站点，以及一个比你的终端更持久的管理集群。步骤 1 到 4 已在租赁的裸金属上端到端运行至 `Ready` 控制平面节点；步骤 5、6 和 8 已针对 mock NICo 而非真实站点运行，步骤 7 两者都未针对。新的管理集群使用 OVN-Kubernetes。若此处出现意外，请视为本页的 bug 并报告。

### 开始之前

**节点。** 奇数个——一、三、五；偶数会被拒绝。每个节点已镜像并运行 `nvke-agent` 和 `spire-agent`。你需要每个节点的 IP 及其 **TPM EK 指纹**：站点集群通过 TPM 认证节点，没有指纹的节点无法加入。

**每个节点上的 kubeadm，由你的 cloud-init 安装。** 从 [`os-image/`](https://github.com/dsx-ai-factory/nvke/tree/main/os-image) 构建的镜像仅将包*暂存*到 `/usr/lib/nvke/debs` 的本地 apt 仓库；对于预配置节点，没有任何东西会安装它们，因为 CWE 不会为其未创建的主机编写 cloud-init。请自行安装：

```yaml
runcmd:
  - DEBIAN_FRONTEND=noninteractive apt-get install -y nvke-kubeadm-distro-<k8s-version>
  - DEBIAN_FRONTEND=noninteractive apt-get install -y nvke-cni-ovn-images-<stream> nvke-cni-ovn-charts-<stream>
```

跳过此步，引导会运行完整的 30 分钟后在 `NodePreflight` / `check_cloud_init_status` 处失败，而该错误从不提及 kubeadm。

**一个 SPIRE CA。** 你通过 `NVKECTL_SPIRE_CA_CERT_FILE` 和 `NVKECTL_SPIRE_CA_KEY_FILE` 传入的证书和密钥，站点 spec 中需有 `spire.upstreamCA: true`，其证书还需通过 cloud-init 到达每个节点。没有它，SPIRE 每次运行都会铸造新 CA，而没有任何节点信任它。

**一个 SPIFFE 信任域，当启用 SPIRE/SPIFFE 引导时。** 在站点 spec 中设置 `spire.trustDomain`。它必须与每个主机 SPIRE agent 上已配置的 `trust_domain` 以及永久站点 SPIRE 服务器将使用的信任域匹配。不要依赖引导集群名称来选择此值：引导后更改它会改变每个 SPIFFE ID，并阻止现有 agent 和工作负载条目迁移到永久服务器。

**一个指向引导节点的 SPIRE VIP。** 引导期间，SPIRE 服务器在你运行 `nvkectl bootstrap site` 的机器上运行，发布在 **30081**。每个节点的 `spire-agent` 在配置时你烘焙进其配置的地址处访问它，因此**该 VIP 必须指向你运行引导的节点。** 使用浮动 IP 而非节点自身地址，以便引导完成后仍能解析。

**网络。** CWE 必须能在 **15005** 上访问每个节点，并在 **6443** 上访问新 API 服务器；有 TPM 指纹的节点必须能在 **30081** 上访问本机，这是 `nvkectl` 发布引导 SPIRE 服务器的固定 NodePort。

**在已配置的裸金属机器之一上运行引导**，在 `tmux` 下——而非会休眠的笔记本电脑。`nvkectl` 直接在该节点上运行；没有跳板机。

**该机器上的 nerdctl 配合 containerd。** `nvkectl` 驱动 kind 并通过 nerdctl 拉取每个引导镜像，并在每次 kind 调用时自行导出 `KIND_EXPERIMENTAL_PROVIDER=nerdctl`。**不支持用 Docker 引导**——该机器无需 Docker 守护进程。**不要以 rootless 运行 nerdctl。** `nvkectl` 和 kind 的 nerdctl provider 都执行裸 `nerdctl` 且无 `sudo`，rootful 是 CI 唯一演练的配置；以 root 运行。

**该机器上的镜像仓库登录**：`nerdctl login nvcr.io`。镜像通过 nerdctl 拉取。

**在 NICo 侧：** 一个站点、一个 VPC、一个子网或 VPC 前缀、实例类型，以及针对每个实例类型的**分配**。没有分配的实例类型看起来健康但不提供任何资源。参见 [CAPNICo 的入门指南](https://github.com/dsx-ai-factory/cluster-api-provider-nico/blob/main/docs/getting-started.md)。

### 1. 构建 nvkectl

```bash
make -C cli build      # 不是 `go build`——那会使 Cluster API 版本未设置
export PATH="$PWD/cli/bin:$PATH"   # 二进制落在 cli/bin，不在 PATH 上
```

本路径其余部分直接调用 `nvkectl`，因此要么如上导出路径，要么以 `./cli/bin/nvkectl` 调用。

`nvkectl` 在引导期间还会调用 `helm`、`kubectl` 和 `clusterctl`，以及下面的 nerdctl 和 containerd。运行开始前只检查容器运行时，因此其他三者之一缺失会在引导中途、Kind 集群已启动后暴露。

### 2. 生成站点 spec

```csv
name,ip,tpm_ek_thumbprint
cp-01,10.10.50.2,<sha256 of the node's TPM EK>
cp-02,10.10.50.3,<sha256 of the node's TPM EK>
cp-03,10.10.50.4,<sha256 of the node's TPM EK>
```

```bash
nvkectl generate sitespec my-site --nodes hosts.csv \
  --set cluster.controlPlaneVIP=10.10.50.10 \
  --set cluster.podCidr=10.244.0.0/16 \
  --set spire.trustDomain=site.example.com \
  --set spire.upstreamCA=true > sitespec.yaml

# SPIRE CA 来自环境。
export NVKECTL_SPIRE_CA_CERT_FILE=./spire-ca.crt NVKECTL_SPIRE_CA_KEY_FILE=./spire-ca.key
nvkectl bootstrap site -f sitespec.yaml --dry-run   # 验证，不触碰任何东西
```

`cluster.controlPlaneVIP` 通过 BGP 通告，因此网络必须与节点对等。参见 [sitespec.md](../cli/docs/sitespec.md)。

阅读它渲染的内容。未设置字段会得到静默而非缺失的默认值——Kubernetes 版本和 `systemReservedMemory` 为 16Gi。每个字段都在 [sitespec.md](../cli/docs/sitespec.md) 中。

**对照你实际拥有的节点检查 `systemReservedMemory`。** kubelet 将可分配量计算为容量减去预留，因此在较小的节点上 16Gi 不会留下任何可调度资源，节点永远无法达到 `Ready`。

### 3. 指向你的镜像

`make -C cli images` 在本主机上用 nerdctl 构建两者，标记为 `nke-controller:latest` 和 `cluster-api-provider-provisioned:latest`，引导默认侧载它们。若要改用推送到镜像仓库的镜像，请命名它们：

```bash
export NVKECTL_NVKE_CONTROLLER_IMAGE=<nvke-controller image>
```