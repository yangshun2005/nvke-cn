# deploy/nico

在 NICo 管理的硬件上部署带 SPIRE 的 NVKE 管理集群的部署路径。这是客户部署路径；`controller/deploy/` 下的 e2e 通道不是。

这不是参考实现：条目列表及其顺序来自一次实际配置运行——该运行在裸金属上搭建了管理集群，并由该集群配置了一个租户。当该运行与 `capnico-e2e` 不一致时，本路径以该运行为准——它是在真实硬件上验证过的。

## 所处位置

四个步骤，三个负责方：

| 步骤 | 负责方 | 产出 |
|---|---|---|
| 1. 创建并镜像实例 | **不是 NVKE**——由 NICo API、iPXE 和 cloud-init 完成 | 一台运行 `nvke-agent` 的已启动主机 |
| 2. `nvkectl bootstrap site` | 实例上的 nvkectl | 一个**原生** Kubernetes 集群 |
| 3. 本部署路径 | 节点上的 `deploy/nvke-deploy.sh` | 管理平面 |
| 4. `nvkectl create cluster` | 管理平面自身 | 租户集群 |

步骤 2 有意只留下裸 Kubernetes。`bootstrap site` 会启动一个临时 kind 集群，其中携带 Postgres、Temporal、SPIRE、CAPI 和 CWE，用该 CWE 通过节点代理在主机上驱动 `kubeadm init`，然后删除它——连同其集群记录一起删除。这就是为什么站点控制器之后无法通过 API 扩缩或升级，也是步骤 3 从零开始的原因。

步骤 3 在节点**上**运行，而不是从工作站对节点运行。这正是镜像方案成本低的原因：节点的 containerd **就是**集群的 containerd，因此在那里导入的镜像配合 `imagePullPolicy: IfNotPresent` 即可解析，无需镜像仓库凭据；CAPNICo 的镜像在 GHCR 上公开，或按摘要来自发布版本，因此本路径不渲染任何拉取密钥。

节点所需的一切，参考实例类型上均已具备：

- `helm`、`kubectl`、`git`、`python3`（执行器的条目解析器和模板渲染器是 Python），以及用于前置条件的 `clusterctl`
- 为站点安装：`nvkectl`（`make -C cli build` 写出 `cli/bin/nvkectl`；`NVKECTL` 指定它，否则用 PATH）。`deploy/check-site.sh --sitespec sitespec.yaml --stack-config stack-config.json --kubeconfig site.kubeconfig` 报告仍缺什么；`deploy/nvke-deploy.sh install --deploy-path deploy/nico --sitespec sitespec.yaml --stack-config stack-config.json --kubeconfig site.kubeconfig` 从这两个文件为站点渲染本部署路径并应用：站点 spec 提供站点名、VIP 和信任域，stack config（`deploy/examples/stack-config.json`，填写后）提供 NICo API 以及可选的镜像和镜像仓库。渲染结果位于私有临时目录，脚本退出时删除，因此不会留下渲染出的凭据。没有发布版本时，stack config 设置 `images.profiles`：本部署路径启用 profiles 内容镜像但不指定其名称
- 一个布局使 `deploy/` 和 `controller/` 为同级目录的检出——`controller/release.yaml` 以 `../../controller/deploy/charts/controller` 访问其 chart
- 到 `registry-1.docker.io`（Bitnami OCI chart）、`spiffe.github.io` 和 `go.temporal.io`（spire-crds 和 temporal 仓库）的出站访问，以及若 spire chart 在那里物化时的 GitHub
- 对于 nktp 条目，到 `open-telemetry.github.io`、`prometheus-community.github.io` 和 `kubernetes-sigs.github.io`（chart 依赖来源）以及 `docker.io`、`ghcr.io`、`quay.io` 和 `registry.k8s.io`（其镜像来源）的出站访问
- 从 NVKR 发布版本安装：添加 `--release release.json`，即 `nvkr release export --reference <repo@digest>` 写出的 `nvkr.dev/release-description/v1` JSON。导出描述发布版本；它不做任何验证。先对同一引用运行 `nvkr release verify`。发布版本携带的 chart 按摘要拉取；镜像按摘要固定；发布版本缺少的 chart 仍用其公共仓库。发布版本决定其携带 chart 的版本，当条目固定了另一版本时给出警告；固定版本仅对发布版本缺少的 chart 生效。每个条目的 `release.yaml` 命名其构建产物：`chart.artifact`（默认：条目名）、`chart.imageArtifact`、`chart.profilesArtifact`，或清单条目的 `artifact`。构建产物按集群节点报告的架构选择，执行器读取该架构并作为 `--arch` 传给渲染。镜像构建产物缺失时，渲染拒绝并列出每个问题。使用 `--nvkr-run-dir .nvkr/runs/<attempt>` 时，chart 和镜像来自 Run 的 OCI 布局，镜像被导入节点的 containerd（`CTR` 指定导入命令）。

## 运行前

`deploy/nvke-deploy.sh` 只安装下列条目，不安装其他。运行前有五件事必须为真，运行后有一件，全部在部署路径之外：

1. **控制平面未被污点标记。** 引导集群全是控制平面，节点上存在 `NoSchedule` 污点时不会有任何调度；`clusterctl init` 会在 cert-manager 上超时且无任何提示。`nvkectl bootstrap site` 会移除该污点，`deploy/check-site.sh` 会报告仍带污点的节点及清除命令。
2. **`clusterctl init` 已运行**，带来 cert-manager、CAPI core 和 kubeadm 提供者。cert-manager 先运行，因为 CAPI 的清单内嵌了 Issuer 和 Certificate。
3. **CWE 和 profiles 镜像已在节点的 containerd 中。** 控制器 overlay 设置 `imagePullPolicy: IfNotPresent`，本路径不为它们提供拉取密钥——这是有意的，因为节点的 containerd **就是**集群的 containerd，导入的镜像无需密钥即可解析。针对发布版本渲染的部署路径按摘要从发布版本的 `cwe` 和 `controller-profiles` 构建产物设置这两个镜像，CAPNICo 镜像来自 `capnico-controller`；节点随后从发布版本指定的镜像仓库拉取，因此这些仓库需从节点可达，需要时配置 containerd 仓库配置或拉取密钥。同时使用 `--nvkr-run-dir` 从已完成的 NVKR Run（`.nvkr/runs/<attempt>`，其 `packages/` 为每个构建产物保存一个 OCI 布局）渲染时，不拉取任何内容：chart 来自布局，渲染路径中的 `images.txt` 命名每个镜像的布局，本脚本在应用任何内容前将其按引用名称导入节点的 containerd，因此引用的主机仅是一个名称。`CTR` 是导入命令，可带参数：安装不以 root 运行时用 `CTR="sudo ctr"`，集群为 kind 节点时用 `CTR="docker exec -i kind-control-plane ctr"`。没有渲染时，安装前导入镜像：

   ```
   docker save --platform linux/amd64 <image> | sudo ctr -n k8s.io images import -
   ```

   命名空间是 `k8s.io`，不是 `default`：导入错误 containerd 命名空间的镜像对 kubelet 不可见，该运行遇到的正是"401 Unauthorized pulling the controller image"。也要构建单清单（`--provenance=false --output type=docker`）——带证明清单的 OCI index 会让 nvkectl 尝试按摘要拉取。

4. **存在默认 StorageClass。** 裸站点不提供任何 StorageClass，所有 PVC 都处于 Pending。参考站点使用 rancher local-path，因为其实例类型只有一块 NVMe 且根在其上；有真实存储的站点应改用该存储。
5. **集群没有从本路径早期修订版安装的残留。** 本路径将控制器安装为发布版本 `nvke-controller`，数据库 `nvke-controller`，Temporal 命名空间 `nvke-controller`。上一修订版三者均用 `nke-controller`，更早的修订版用发布版本 `controller`、数据库 `cloud-workflow` 和 chart 的 Temporal 名称。Helm 无法接管旧发布版本的对象，Postgres 仅在空卷上运行其 `initdb` 脚本。没有东西迁移旧数据库：在全新集群上安装。
6. **站点的发布版本资源随后发布。** CWE 从 `MachineImage`、`ClusterRelease` 和 `NodePoolRelease` 资源读取其 OS 镜像、用户初始化脚本等。chart 安装 CRD；在 `nvkectl config set-site` 之后，用 `nvkectl create machineimage|clusterrelease|nodepoolrelease -f <file>` 发布资源，如 docs/get-started/local-site.md 所示。

## 运行后

**将 SPIRE exec-auth 插件注入 CAPI 控制器**，使用 `controller/hack/apply-capi-spiffe-exec-auth.sh`。

`controller/deploy/k8s/capi-spiffe-exec-auth/` 用 `csi.spiffe.io` 卷和 SPIFFE JWT exec-auth 插件修补 CAPI 控制器 Deployment，使 Cluster API 用 SPIFFE JWT 而非存储的管理员 kubeconfig 访问租户集群。两个 e2e 通道都运行它——`backend-e2e.sh` 在部署路径之后立即运行，NICo 通道和 CAPD 均如此——`spiffe.enabled=true` 的站点出于同样原因也需要它。

它不是条目，也不能是：它修补 `clusterctl init` 拥有的 Deployment，本部署路径不安装这些。它应归入 `clusterctl init`，只是它在之后而非之前运行——脚本会跳过其 rollout 等待直到 `csi.spiffe.io` 注册，而该 CSIDriver 随 spire 条目到达。在安装前运行，它修补后继续，rollout 未经验证；在之后运行，则会被检查。

overlay 包含 `capd-controller-manager.yaml`，NICo 站点没有其命名空间。脚本跳过不存在的 Deployment，因此无妨。

该运行的安装脚本不做这些，因为它有 `spiffe.enabled=false`，且没有东西需要工作负载 API。

## 每站点值

每个条目提交的值与站点无关。安装将变化的部分从站点 spec 写入每个条目的 `site-values.yaml`；e2e 通道在 `controller/hack/ci/deploy-stack.sh` 中推导相同的值。`--values-for` 仍应用于渲染路径之上，在 `site-values.yaml` 之后。

| 值 | 安装取自 | e2e 推导 |
|---|---|---|
| 信任域 | `spire.trustDomain`，否则 `nvkectl-bootstrap.nke.local` 并警告 | `${KIND_CLUSTER_NAME}.nvke.local` |
| 管理 VIP | `cluster.controlPlaneVIP`，或唯一节点的 IP | kind 控制平面 IP |
| SPIRE 管理员路径 | 常量，`/nvke-service/nvke-controller` | 相同 |
| TPM 证明器**服务端**校验和 | 按节点架构固定，`--arch` | 在 `resolve_tpm_checksum()` 中按架构固定 |
| OIDC 域 / IP SAN | overlay 的主机名加 VIP | VIP 加 kind 网关 |
| 发现 URL | `https://<VIP>:30443/.well-known/openid-configuration` | 相同形式 |
| Postgres / Temporal 密码 | `NVKECTL_DATABASE_PASSWORD`，由渲染添加的 `site-credentials` 条目写入 postgres、temporal 和 nvke-controller 命名空间的 `nvke-database` Secret；每个 chart 从那里读取，没有 values 文件携带它 | 每次运行生成 |
| SPIRE 上游 CA | `NVKECTL_SPIRE_CA_CERT_FILE` 和 `NVKECTL_SPIRE_CA_KEY_FILE`，当设置 `spire.upstreamCA` 时由同一条目写入 spire-server 中的 `spire-upstream-ca` Secret | 自签名 |
| 控制器镜像 | 发布版本，或 stack config 的 `images.controller` | 每次运行构建 |
| profiles 镜像 | 发布版本，或 stack config 的 `images.profiles` | 每次运行构建 |
| 遥测后端 | `--values-for nktp=<file>`，否则 chart 捆绑的开发后端 | 不安装 |
| 遥测 `k8s.cluster.name` | `clusterName`，通过 `nvke-base-config` 模板 | 不安装 |

TPM 校验和容易混淆：`spire/values.yaml` 中的值（`spire-server.nodeAttestor.tpmDirect.checksum`）是**服务端**插件，与节点上 `/etc/spire-agent/spire-agent.env` 中的 `TPM_PLUGIN_CHECKSUM` 无关，后者是**代理**插件。两者均按架构区分。

## SPIRE

`spiffe.enabled=true`，五个 SPIRE 条目与 `capnico-e2e` 每次运行安装并验证的相同。

节点侧无需构建：发布版本将 `/etc/spire-agent/agent.conf` 和 `initial-trust-bundle.pem` 写入 cloud-init，这是 `os-image/SPEC.md` 允许的两条互斥路径之一——另一条是 `/etc/nke/spire.env` 加镜像的渲染器。采用第一条路径的 profile 在缺少第二条的镜像上也能工作。

首次运行前值得知道两个顺序事实，均无通道在 NICo 上验证过：

- CWE 从其自身的 SPIFFE 工作负载套接字填充 `initial-trust-bundle.pem`，因此 **CWE 必须先持有 SVID 才能引导节点**。
- 租户节点必须能访问 profile 渲染的地址上的 SPIRE 服务器。e2e 中是 kind NodePort；这里是 VIP，由 `spire-server-nodeport` 条目发布。

管理集群自身的节点是另一回事，保持明文：它在 CWE 之外配置，其引导时唯一存活的 SPIRE 是临时 kind 集群中的那个，而该集群已被删除。之后没有东西访问该节点。

## 遥测

`nktp` 条目从本检出（`nktp/deploy/charts/nktp`）安装 NKTP chart，或从针对携带 `nktp` chart 构建产物的发布版本渲染时放在 `charts/` 下的归档安装，值在 `nktp/values.yaml`。它紧接 `capnico` 之后，因为它安装监控 CRD，后续三个条目创建的监控需要它们：

| 条目 | 添加内容 | 位置 |
|---|---|---|
| `temporal` | 每个服务端角色的 ServiceMonitor | `temporal/overlay.yaml` |
| `spire` | 服务端和代理的 PodMonitor | `spire/overlay.yaml` |
| `controller` | 其 ServiceMonitor、允许 `nktp` 的指标 NetworkPolicy，以及到 NKTP 节点 Collector 的追踪 | `controller/overlay.yaml` |

chart 自身的控制平面监控覆盖 API server、etcd 和 CoreDNS；`nktp/values.yaml` 关闭调度器、控制器管理器和 kube-proxy 的监控，NVKE 集群无法提供这些。本路径不为其他组件创建监控：每个组件自带，不自带的不会被采集。

NKTP 导出的所有内容都带 `k8s.cluster.name`，即站点名。集群中没有任何东西知道它——NVKE 的 kubeadm 配置将每个集群命名为 `kubernetes`——因此 `nvke-base-config` 将其渲染进 `nktp-cluster` ConfigMap，watchdog 和三个网关（即导出到后端者）从那里读取。使用 `--sitespec` 时，渲染从 spec 的 `clusterName` 取值。没有站点 spec 的普通安装从 `--config` 传入的 BootstrapConfig 渲染模板，其 `ClusterName` 必须设置；空名称会在该模板处停止安装。

默认所有内容导出到 chart 捆绑的开发后端。它将数据保存在内存中且无认证，因此仅供评估。有真实后端的站点向安装传 `--values-for nktp=<file>`，无论有无站点 spec，它都在条目自身值之后应用。chart README 的"The backend"节列出该文件设置的键。

尚未采集：Postgres 条目不运行指标导出器；CAPI core 和 kubeadm 提供者仅向带 bearer token 的调用方提供指标，节点 Target Allocator 不发送；调度器和控制器管理器仅监听 127.0.0.1；cert-manager、NodeLocal DNSCache、OVN-Kubernetes 和 CAPNICo 安装时无监控。其 Pod 状态仍通过 kube-state-metrics 到达。