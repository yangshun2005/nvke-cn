# 基础设施提供商

> **注意：** 安装 Cluster API 基础设施提供商是**引导后的事项**。`nvkectl bootstrap site` 只将 Cluster API 安装到它创建随后又销毁的*临时* Kind 集群中，因此刚引导完成的站点集群上既没有 Cluster API，也没有任何提供商。本页描述的是第二步。

租户集群创建在 Cluster API **基础设施提供商**之上：这是一个运行在站点集群上的控制器，持有后端凭据并在后端创建机器。`nvkectl` 从不安装它。每个站点需在首次执行 `nvkectl create cluster` 之前手动安装一次。

本页涵盖 [CAPNICo](https://github.com/dsx-ai-factory/cluster-api-provider-nico)（NICo 提供商），管理平面默认接受它。[LaunchLayer](#launchlayer) 是另一个受支持的提供商，部署需显式启用后才接受。两节均链接到提供商自身的文档，不再赘述。

## 前提条件

| 要求 | 说明 |
|---|---|
| 已引导的站点集群 | 来自[预置节点](pre-provisioned-nodes.md)，需有对应的 kubeconfig |
| `kubectl` | 指向站点集群 |
| `clusterctl` | 与 NVKE 固定的 Cluster API 版本 `v1.13.6` 一致 |
| `helm` **v4 或更高版本** | Helm 3 无法安装 CAPNICo chart；参见[故障排查](#troubleshooting) |
| 出站网络 | `clusterctl init` 从 `github.com` 拉取 Cluster API 和 cert-manager。气隙站点需要镜像 |
| NICo 站点 | 已安装、机器已注册，且有可为你签发令牌的人员 |

安装本身不需要 `nvkectl`，仅在最后的验证步骤中用到。

## 顺序很重要

按以下顺序安装：

1. Cluster API 核心及 kubeadm 提供商
2. 基础设施提供商：CAPNICo、[CAPLL](#launchlayer)，或两者
3. 管理平面（NVKE 控制器）

**NVKE 控制器必须最后安装。** 它在进程启动时一次性读取 `Machine`、`NicoMachine` 和 `LaunchLayerMachine` CRD，不会重试。缺少 `Machine` CRD 会导致进程停止。当 `controllers.nico.enabled` 为 true 时，缺少 `NicoMachine` CRD 会导致进程停止。缺少 `LaunchLayerMachine` CRD 会跳过 LaunchLayer SPIFFE 控制器。如果控制器已在运行，请在提供商安装完成后重启它。请通过 chart 自身的标签而非硬编码名称查找它，因为 Deployment 名称取决于你安装控制器时使用的 Helm release 名称：

在已有站点上，控制器以 `cloud-workflow-engine` 运行，请在下方命令中将命名空间和标签都替换为 `cloud-workflow-engine`。

```bash
kubectl rollout restart deployment -n nvke-controller -l app.kubernetes.io/name=nvke-controller
```

## 1. 确认控制平面污点已移除

站点集群上的每个节点都是控制平面节点，cert-manager、Cluster API 管理器以及 CAPNICo 管理器都无法容忍控制平面污点。`nvkectl bootstrap site` 会移除它。在任何等待操作之前先确认这一点，否则接下来两步中的等待会挂起直至超时，且没有有用的错误信息：

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.spec.taints}{"\n"}{end}'
```

不应有任何节点列出 `node-role.kubernetes.io/control-plane`。如果有（因为引导过程报告移除失败），请手动移除：

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```

站点集群还需要一个默认 StorageClass，控制器的 Postgres 和 SPIRE 才能调度。这属于安装管理平面的内容，不在本页范围内。

## 2. 安装 Cluster API

使用 NVKE 自身固定的调用方式。**不要**使用 CAPNICo 文档中未固定版本的 `clusterctl init` 命令：未指定版本时，`clusterctl` 会安装最新版本，与 NVKE 固定的版本产生偏差。

```bash
clusterctl init \
  --core cluster-api:v1.13.6 \
  --bootstrap kubeadm:v1.13.6 \
  --control-plane kubeadm:v1.13.6 \
  --wait-providers
```

这也会安装提供商所依赖的 cert-manager。

`v1.13.6` 是 NVKE 在 `make/tool-versions.mk` 中固定的版本。请保持两者一致。

之后再次运行 `clusterctl init` 是受支持的，会跳过已存在的组件，因此可以安全地重复执行。只有*冲突的*提供商实例才会报错。

## 3. 安装 CAPNICo

有两种方式。**选择一种并坚持使用。** 两者都会创建相同的 `capnico-system/controller-manager` Deployment 和相同的资源名称，混用不会得到两个安装——而是两个东西争抢同一批对象。在 `clusterctl` 安装之上使用 Helm 会因所有权元数据直接失败；在 Helm 安装之上使用 `clusterctl` 会修补 Helm 的资源并添加自己的清单条目，导致 release 与集群对已安装内容的认知不一致。

### 使用 Helm，从已发布的 chart 安装

已发布的 chart 是自包含的：发布时已嵌入匹配的控制器镜像，因此无需 `--set manager.image.*`，也无需拉取密钥。chart 及其引用的镜像均为公开。

```bash
helm install capi-provider-nico \
  oci://ghcr.io/dsx-ai-factory/cluster-api-provider-nico/charts/capi-provider-nico \
  --version 0.1.0 \
  --namespace capnico-system --create-namespace \
  --wait
```

请从[仓库的 releases](https://github.com/dsx-ai-factory/cluster-api-provider-nico/releases) 中选择版本，而非跟踪最新版，以便站点能说明其运行的是哪个构建。

> **不要预先创建 `capnico-system` 命名空间。** chart 会无条件创建它。预先创建会导致 Helm 报 `invalid ownership metadata` 失败。这就是凭据 Secret 要在安装*之后*创建的原因（见下一步）。

### 使用 clusterctl，从 release 资产安装

每个 CAPNICo release 都会发布 `infrastructure-components.yaml` 和 `metadata.yaml`，这正是 `clusterctl` 所需的。将提供商添加到 `clusterctl` 配置中：

```yaml
providers:
  - name: nico
    url: https://github.com/dsx-ai-factory/cluster-api-provider-nico/releases/download/v0.1.0/infrastructure-components.yaml
    type: InfrastructureProvider
```

```bash
clusterctl init --infrastructure nico:v0.1.0 --config ./clusterctl.yaml --wait-providers
```

> **显式传入 `--config`。** `clusterctl` 按操作系统惯例解析默认配置路径，因此在 macOS 上**不会**读取 `~/.config/cluster-api/clusterctl.yaml`——它会去 `~/Library/Application Support/cluster-api/` 查找。失败信息很不友好：`failed to get configuration for the InfrastructureProvider with name nico`。使用 `--config` 或 `~/.cluster-api/clusterctl.yaml` 在两个平台上都有效。

## 4. 创建凭据 Secret

此步骤由 CAPNICo 负责。请遵循其入门指南中的 **[3. Get a Machine-Usable Credential](https://github.com/dsx-ai-factory/cluster-api-provider-nico/blob/main/docs/getting-started.md#3-get-a-machine-usable-credential)**，其中列出了确切的键。

在安装**之后**，在 chart 创建的命名空间中创建：

```bash
kubectl create secret generic nico-credentials -n capnico-system \
  --from-literal=endpoint=<nico-api-base-url> \
  --from-literal=orgID=<your-org> \
  ...
```

`nvkectl` 默认在 `capnico-system` 中查找名为 `nico-credentials` 的 Secret。

在创建任何内容之前，请阅读 CAPNICo 的 **[2. Check the Instance Types Have Allocations](https://github.com/dsx-ai-factory/cluster-api-provider-nico/blob/main/docs/getting-started.md#2-check-the-instance-types-have-allocations)** 一节。没有分配的实例类型看起来正常但不会创建任何资源，机器会永远在 `InstanceTypeUnavailable` 中重试。这是首个集群卡住最常见的原因。

## 5. 验证

此处使用 `kubectl`。`nvkectl get providers` 尚无法运行：它需要先用 `nvkectl config set-site` 注册站点，而这需要控制器的 URL，该 URL 要等到下一步安装管理平面后才存在。

```bash
# Cluster API 核心及 kubeadm 提供商
kubectl get pods -n capi-system
kubectl get providers -A

# CAPNICo
kubectl get deployment controller-manager -n capnico-system
kubectl get crd | grep nico
kubectl get secret nico-credentials -n capnico-system
```

CAPNICo Deployment 应处于可用状态，`nicoclusters`、`nicoclustertemplates`、`nicomachines` 和 `nicomachinetemplates` 均应已建立。

管理平面安装完成并运行 `nvkectl config set-site` 后，`nvkectl get providers` 会在一行中报告相同信息，`nico` 应显示 `CONFIGURED`。

> **`CONFIGURED` 的含义比看起来要窄。** 它表示提供商的 CRD 已建立、控制器正在运行、凭据 Secret 存在。它**不**检查 Cluster API 核心是否已安装、凭据是否有效、后端是否可达。CAPNICo 自带 CRD，因此即使集群上完全没有 Cluster API，`nico` 也会显示 `CONFIGURED`。请将其理解为"提供商已安装"，而非"站点已就绪"。

`nvkectl get providers` 的 `CREDENTIALS` 列报告凭据是否有效，依据是 CAPNICo 每隔几分钟对其执行的检查。仅当 CAPNICo 提供 `NicoIdentity`、站点 kubeconfig 能列出 `NicoIdentity` 对象、且其中一个对象指向该凭据 Secret 时，才显示 `HEALTHY`；否则显示 `UNSUPPORTED`、`UNREADABLE` 或 `NO-IDENTITY`。所有取值见[基础设施提供商](../../cli/README.md#infrastructure-providers)。

要在投入硬件之前证明凭据确实有效，请遵循 CAPNICo 的 **[5. Prove the Credentials Before Creating Machines](https://github.com/dsx-ai-factory/cluster-api-provider-nico/blob/main/docs/getting-started.md#5-prove-the-credentials-before-creating-machines)** 一节，它会应用一个一次性的 `NicoCluster` 来验证身份，不创建任何资源。

## 故障排查

**Helm 报 `namespaces "capnico-system" already exists` 或 `invalid ownership metadata`。** 要么你使用的是 Helm 3，要么你预先创建了命名空间。chart 要求 **Helm 4**；NVKE 固定 `v4.2.4`。没有任何错误信息会提到 Helm 版本。

**`clusterctl init` 或 `helm --wait` 挂起。** 节点上仍有控制平面污点。参见步骤 1。

**`clusterctl` 找不到 `nico` 提供商。** 配置文件不在 `clusterctl` 查找的位置。传入 `--config`。参见步骤 3。

**租户集群被接受但没有任何反应。** `nvkectl create cluster` 只校验请求的结构，不校验是否存在提供商来处理它。API 返回成功，但没有任何协调发生。先检查 `nvkectl get providers`。

**机器停留在 `InstanceTypeUnavailable`。** 该实例类型没有 NICo 分配。参见步骤 4。

关于提供商侧的状态——`WaitingForBootstrapData`、`WaitingForIdentitySecret` 等——参见 CAPNICo 的 **[7. When It Stalls](https://github.com/dsx-ai-factory/cluster-api-provider-nico/blob/main/docs/getting-started.md#7-when-it-stalls)** 一节。

## LaunchLayer

[CAPLL](https://github.com/dsx-ai-factory/cluster-api-provider-launchlayer) 在 LaunchLayer 计算资源上运行租户集群。按步骤 2 安装 Cluster API，然后安装 CAPLL 并按[其 README](https://github.com/dsx-ai-factory/cluster-api-provider-launchlayer#against-a-real-launchlayer-deployment) 的描述将其指向你的 LaunchLayer 部署：管理器需要 `--base-url` 和一个持有 OAuth2 客户端凭据的 `--identity-secret`。NVKE 不提供这两者。

之后管理平面必须接受 LaunchLayer 请求。在此之前，它会以 400 拒绝这些请求。设置以下 chart 值：

| 值 | 设置 |
|---|---|
| `supportedInfraProviders` | 在现有值后追加 `launchLayer`，而非替换。部署 overlay 提供的是 `"nico,infra"`，应变为 `"nico,infra,launchLayer"` |
| `capi.rbac.create` | `true`。默认为 `false`，不设置则 chart 不渲染任何 CAPI 或 LaunchLayer RBAC |
| `capi.rbac.launchLayer` | `true`（默认值）。当 `capi.rbac.create` 为 `true` 时，授予控制器访问 LaunchLayer CRD 的权限。只要 CAPLL 的 CRD 仍安装，就应保持 `true`，即使已从 `supportedInfraProviders` 中移除 `launchLayer`：只要 `LaunchLayerMachine` CRD 存在，NVKE 控制器就会运行 LaunchLayer SPIFFE 控制器，因此现有集群的新机器仍能加入；若没有该授权，其 watch 会被拒绝，控制器会退出 |
| `tenantAccess.enabled` | `true`（默认值）。控制器只能通过此选项发布的 kubeconfig Secret 访问 LaunchLayer 集群的 API；关闭后，LaunchLayer 集群会一直停留在 `Provisioning` |
| `nodeAgent.spireAttestationMethod` | 当 `spiffe.enabled` 为 `true` 时设为 `dsxImds`。LaunchLayer 节点只能通过 DSX IMDS 向 SPIRE 证明身份，因此启用 SPIFFE 的部署若使用其他方法，会以 400 拒绝 LaunchLayer 请求 |
| `nodeAgent.dsxImds.operatorTenant` | 使用 `dsxImds` 时必填；缺少时 chart 拒绝渲染。必须与 DSX Identity 的 `global.operatorTenant` 一致 |
| `nodeAgent.dsxImds.endpoint` | 使用 `dsxImds` 时必填。节点证明器查询的元数据端点；默认值 `http://169.254.169.254` 是标准链路本地地址 |

`nodeAgent.spireAttestationMethod` 适用于所有提供商的新节点，不仅限于 LaunchLayer。NICo 接受任何方法，因此在 NICo 节点使用 TPM 证明的站点上，设置 `dsxImds` 也会将新的 NICo 节点固定为该方法，它们只在证明时才会失败。使用 TPM 证明的 NICo 部署无法通过此方式启用 LaunchLayer。

DSX IMDS 仅识别裸金属实例。在 SPIFFE 开启且 `nodeAgent.externalMTLS.enabled` 关闭时，管理平面会以 400 拒绝任何设置了 `vmProfile` 的 LaunchLayer 节点池或控制平面，因为其节点永远无法完成证明。

SPIRE 需要两侧都配置 `dsx_imds` 节点证明器：

- **服务端。** 在 SPIRE server 中添加 `dsx_imds` 节点证明器插件，`jwks_url` 和 `issuer` 指向 DSX Identity 的签名密钥和签发者，`audience` 设为 `spiffe://<trust-domain>/spire/server`。
- **节点。** 节点镜像必须在 `/usr/local/lib/spire-agent/dsx-imds-node-attestor-agent` 安装 agent 插件，并在 `/etc/spire-agent/spire-agent.env` 中提供其校验和作为 `DSX_IMDS_PLUGIN_CHECKSUM`，或设置 `nodeAgent.dsxImds.pluginChecksum`。控制器会自行写入 agent 的证明器配置。

上述顺序规则在此同样适用：NVKE 控制器在进程启动时一次性检查 `LaunchLayerMachine` CRD，缺失时跳过 LaunchLayer SPIFFE 控制器。请在控制器之前安装 CAPLL，或在安装后重启控制器。

## 本页未涵盖的内容

- **管理平面（NVKE 控制器）安装。** 独立步骤，必须在本步骤之后进行。
- **`cluster-api-provider-provisioned`。** 它随本仓库发布，但不是租户集群基础设施提供商，也不安装在站点集群上。
- **在租户集群上安装 GPU 运行时。** 租户集群创建后，驱动、Operator 和系统配置通过 [NVIDIA AI Cluster Runtime](https://github.com/NVIDIA/aicr)（独立项目）安装。

## 后续步骤

- [部署仓库](deploy-repository.md) — NVKE 服务栈的 ArgoCD 应用结构
- [CAPNICo：路径 C，针对现有 NICo 站点的生产部署](https://github.com/dsx-ai-factory/cluster-api-provider-nico/blob/main/docs/getting-started.md#path-c-production-against-an-existing-nico-site) — 提供商自身的生产站点指南
- [你的第一个集群](../first-cluster.md) — 提供商配置完成后创建租户集群