# 站点规格文件

`nvkectl bootstrap site` 读取单个 YAML 站点规格文件，其中描述预置节点及要在其上构建的 Kubernetes 集群。文件在任何节点被操作前加载并完整校验——拼写错误或缺失字段会立即报错失败。

`nvkectl generate sitespec <name> --nodes hosts.csv` 可从节点 CSV 生成该文件，因此很少需要手工编写——参见[预置节点](pre-provisioned-nodes.md#step-2-run-bootstrap-on-init-node)。本页概述 Schema；完整说明见 CLI 的[站点规格参考](../../cli/docs/sitespec.md)。

## 示例

```yaml
clusterName: "nvke-site"

cluster:
  # Pre-provisioned control-plane nodes.
  nodes:
    - name: "ctrl-1"
      ip: "10.0.0.1"
    - name: "ctrl-2"
      ip: "10.0.0.2"
    - name: "ctrl-3"
      ip: "10.0.0.3"

  k8sVersion: "1.36.3"     # plain semver, no leading "v"
  cniName: "ovn-kubernetes"
  podCidr: "192.168.0.0/16"
  serviceCidr: "10.96.0.0/12"

  kubeletConfig:
    systemReservedMemory: "16Gi"
```

## 字段参考

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `clusterName` | string | 是 | 小写 RFC 1123 标签。 |
| `infra.provider` | string | 否 | `provisioned`（默认）或 `nico`。使用 `nico` 时，CAPNICo 分配控制平面机器，由 `infra.nico` 描述；参见 CLI 的[站点规格参考](../../cli/docs/sitespec.md)。 |
| `cluster.nodes` | list | 是（使用 `provisioned` 时） | 必须包含奇数个控制平面节点且至少一个节点。使用 `infra.provider: nico` 时不使用，此时改用 `infra.nico.controlPlane.count`，并拒绝此列表。 |
| `cluster.nodes[].name` | string | 是（每项） | 小写 RFC 1123 标签。所有节点中必须唯一。 |
| `cluster.nodes[].ip` | string | 是（每项） | 有效的 IPv4 或 IPv6 地址。所有节点中必须唯一。 |
| `cluster.nodes[].tpmEkThumbprint` | string | 否 | 节点 TPM 背书密钥的 SHA-256：64 个小写十六进制数字。列出的每个指纹都会加入 bootstrap 部署的 SPIRE 服务器的允许列表；没有指纹的节点无法证明，文件中没有指纹则不部署 TPM 证明器。仅适用于 `provisioned`：使用 `infra.provider: nico` 时没有节点列表，证明器始终开启，其允许列表由 CAPNICo 创建的节点填充。使用 `--disable-mtls` 时忽略。 |
| `cluster.k8sVersion` | string | 是 | 纯 semver，例如 `1.36.3`——不带前导 `v`。 |
| `cluster.cniName` | string | 否 | 默认为且必须为 `ovn-kubernetes`。 |
| `cluster.cniVersion` | string | 否 | CNI 插件版本。为空时使用 agent 内置默认值。 |
| `cluster.controlPlaneVIP` | string | 多节点时必填 | kube-vip 控制平面 VIP。当 `cluster.nodes`（或 nico 下的 `infra.nico.controlPlane.count`）多于一项时必填。必须是可路由的单播 IPv4 地址，节点可通过 BGP 访问，且在 `podCidr` 和 `serviceCidr` 之外。可以等于已声明的节点 IP，但 CLI 接受并不证明 bootstrap 或网络可用。运行时约束见 [CLI 校验规则](../../cli/docs/sitespec.md#validation-rules)。单节点 `provisioned` 集群可省略，使用该节点 IP 作为固定端点；使用 `nico` 时即使单节点也需要，因为节点在 NICo 分配前没有地址，且站点集群创建后无法再添加。 |
| `cluster.podCidr` | string | 是 | 有效 CIDR。双栈时允许逗号分隔列表。 |
| `cluster.serviceCidr` | string | 是 | 有效 CIDR。 |
| `cluster.nodeAgentPort` | string | 否 | nvke-agent 监听的 gRPC 端口。默认 `15005`。仅当向 agent 传递了 `--secure-server-port` 时设置。 |
| `cluster.etcdEncryption` | bool | 否 | 静态加密 Kubernetes API 的 etcd 数据。默认 `true`。仅在 bootstrap 时决定：以 `false` 构建的集群之后无法加密。 |
| `cluster.kubeletConfig.systemReservedMemory` | string | 否 | 例如 `16Gi`，也是模板默认值。kubelet 从容量中扣除它，因此在 16Gi 或更少的节点上无可分配资源，节点永远无法达到 `Ready`；请按硬件设置。 |
| `cluster.oidcProviders` | list | 否 | 站点集群的 OIDC 提供方。之后无法修改。缺失时忽略。`cluster.k8sVersion` 低于 `1.34.0` 时拒绝。每项需要 `name`、`issuerURL`、`audiences`、`usernameClaim`、`usernamePrefix` 和 `caBundle`（内联 PEM）。可选：`groupsClaim`、`groupsPrefix`、`requiredClaims`。 |
| `spire.upstreamCA` | bool | 否 | 当站点的 SPIRE 链接到你提供的 CA 时为 `true`。此后每次 bootstrap、校验和安装运行都需要 `NVKECTL_SPIRE_CA_CERT_FILE` 和 `NVKECTL_SPIRE_CA_KEY_FILE`，未设置时拒绝这些变量。 |
| `spire.trustDomain` | string | 否 | SPIFFE 信任域，由 bootstrap 和控制器栈安装共享。在 bootstrap 前设置一次，或不设置：两者都使用 `nvkectl-bootstrap.nke.local`，安装时会说明。设置了 `NVKECTL_BOOTSTRAP_CLUSTER_NAME` 的 bootstrap 运行会派生另一个，因此此时需设置。 |
| `spire.externalAddress` | string | 使用 `nico` 时必填，除非 `--disable-mtls` | CAPNICo 创建的节点用于访问 bootstrap SPIRE 服务器的 IP 地址或 DNS 名称，端口为 NodePort 30081。使用 `provisioned` 时拒绝。 |

未知字段会被拒绝。字段名拼写错误会导致加载失败，而非被静默忽略。

## 环境变量覆盖

任何字段都可通过环境变量提供或覆盖。已设置的变量优先于文件值。

| 变量 | 字段 |
|---|---|
| `CLUSTER_NAME` | `clusterName` |
| `CLUSTER_K8S_VERSION` | `cluster.k8sVersion` |
| `CLUSTER_CNI_NAME` | `cluster.cniName` |
| `CLUSTER_CNI_VERSION` | `cluster.cniVersion` |
| `CLUSTER_CONTROL_PLANE_VIP` | `cluster.controlPlaneVIP` |
| `CLUSTER_POD_CIDR` | `cluster.podCidr` |
| `CLUSTER_SERVICE_CIDR` | `cluster.serviceCidr` |
| `CLUSTER_SYSTEM_RESERVED_MEMORY` | `cluster.kubeletConfig.systemReservedMemory` |
| `CLUSTER_NODE_AGENT_PORT` | `cluster.nodeAgentPort` |
| `CLUSTER_ETCD_ENCRYPTION` | `cluster.etcdEncryption` |
| `NVKECTL_SPIFFE_TRUST_DOMAIN` | `spire.trustDomain` |
| `NVKECTL_SPIRE_EXTERNAL_ADDRESS` | `spire.externalAddress` |
| `NVKECTL_INFRA_PROVIDER` | `infra.provider` — `provisioned` 或 `nico` |
| `NVKECTL_NICO_CREDENTIALS` | 非站点规格字段：保存 NICo API 令牌的文件路径，在 bootstrap 主机上读取。`bootstrap site` 和 `validate sitespec --bootstrap-only` 在 `infra.provider: nico` 时必填 |
| `NVKECTL_NICO_ALLOW_INSECURE_ENDPOINT` | 非站点规格字段：设为 `1` 允许 `infra.nico.remote.endpoint` 为 http URL。bootstrap 会向其发送 NICo 令牌，因此未设置时要求 https；用于本地或模拟 NICo |
| `NVKECTL_CLUSTER_NODES` | `cluster.nodes` — 格式 `name:ip,name:ip`，仅当文件未列出节点时应用 |

不支持 `INIT_NODE_IP` 和 `INIT_NODE_NAME`。清单不为 `kubeadm init` 选择节点；bootstrap 过程在下游选择。

## 凭据处理

bootstrap 从环境读取 SPIRE CA，因此站点规格可提交并在任何机器上原样使用。`NVKECTL_SPIRE_CA_CERT_FILE` 和 `NVKECTL_SPIRE_CA_KEY_FILE` 指定 SPIRE 链接自身 CA 的文件，这样在预置时获得该证书的节点可在首次连接时验证服务器。在站点规格中设置 `spire.upstreamCA: true` 并配合它们，此后每次运行都需要它们；两个文件在加载时检查。没有它们时 SPIRE 自签名。

```bash
NVKECTL_SPIRE_CA_CERT_FILE=/etc/nke/spire-ca.crt NVKECTL_SPIRE_CA_KEY_FILE=/etc/nke/spire-ca.key \
  nvkectl bootstrap site -f sitespec.yaml
```

使用 `infra.provider: nico` 时，bootstrap 还从 `NVKECTL_NICO_CREDENTIALS` 指定的文件读取 NICo 令牌，并从 `NVKECTL_NODE_USER` 和 `NVKECTL_NODE_PASSWORD` 读取渲染到每个节点的登录信息；参见 [CLI 参考](../../cli/docs/sitespec.md#credential-handling)。

私有镜像仓库的镜像通过 nerdctl 拉取，因此 bootstrap 前请用 `nerdctl login <registry>` 登录仓库。

控制器栈安装的凭据列在[站点规格参考](../../cli/docs/sitespec.md#credential-handling)中。

## 校验站点规格

`nvkectl validate sitespec` 按 `bootstrap site` 读取文件的方式检查文件，包括 `spire.upstreamCA` 要求的 SPIRE CA 变量，并打印 `valid`。所有问题一次性报告。使用 `infra.provider: nico` 时，仅 bootstrap 主机持有的内容（NICo 令牌文件、节点登录、发布文件、`spire.externalAddress`，以及对 `infra.nico.remote.endpoint` 的 https 检查）会被报告但不改变退出状态，因为控制器栈安装在缺少这些输入的主机上运行此命令；添加 `--bootstrap-only` 使其失败。

```bash
nvkectl validate sitespec -f sitespec.yaml
nvkectl validate sitespec -f sitespec.yaml --bootstrap-only   # on the bootstrap host, with nico
nvkectl validate sitespec -f sitespec.yaml --bootstrap-only --disable-mtls   # as "bootstrap site --disable-mtls" would: no spire.externalAddress needed with nico
```

`bootstrap site --dry-run` 加载并校验 bootstrap 字段，打印解析后的清单，不派发任何工作流或接触任何节点。所有校验错误一次性报告。

```bash
nvkectl bootstrap site -f sitespec.yaml --dry-run
```

控制器栈安装也读取站点规格，用于站点名称、控制平面 VIP 和 SPIRE 信任域，并从自己的文件（栈配置）获取仅栈需要的内容；[deploy/README.md](../../deploy/README.md) 有描述。安装从 `NVKECTL_NICO_TOKEN`、`NVKECTL_NODE_USER`、`NVKECTL_NODE_PASSWORD` 和 `NVKECTL_DATABASE_PASSWORD` 读取 NICo 令牌、节点登录和 PostgreSQL 密码，因此两个文件都不保存凭据。

## Bootstrap 之后

管理集群启动后，先在其上安装 Cluster API 和基础设施提供方，再部署 NVKE 服务（CWE、Temporal、CNPG）——参见[基础设施提供方](infrastructure-providers.md)。NVKE 服务部署使用基于 GitOps 的部署仓库和 ArgoCD；结构和应用清单见[部署仓库](deploy-repository.md)。