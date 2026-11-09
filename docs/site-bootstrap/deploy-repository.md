# 部署路径

部署路径是一个目录，按安装顺序列出 NVKE 服务。它包含一个 `nvke-deploy.yaml`，以及每个条目对应一个目录。条目可以是带 values 的 Helm chart、一组普通清单，或用于渲染站点密钥的模板。

NVKE 自带一个部署路径 [`deploy/nico`](https://github.com/dsx-ai-factory/nvke/tree/main/deploy/nico)。`deploy/nvke-deploy.sh install` 会按站点规格和 NVKR 发布版本渲染并应用它，作为安装的一个步骤。条目列表及其输入见[在单台机器上端到端运行 NVKE](../get-started/local-site.md)。

部署路径是接口。脚本是应用它的参考方式。运维人员可以用自己的工具或通过 GitOps 应用同样的条目。`deploy/nvke-deploy.sh install --dry-run` 会按顺序打印脚本将执行的每个 kubectl 和 helm 调用。

## 布局

```text
deploy/nico/
├── nvke-deploy.yaml              # 条目，按安装顺序
├── nvke-base-config/             # 模板：命名空间、SPIRE CA 链、两个 Secret
│   ├── 00-namespaces.yaml
│   ├── 20-nico-credentials.yaml.tmpl
│   ├── 35-dgxc-tenant-credentials.yaml.tmpl
│   ├── 40-oidc-upstream-ca.yaml
│   ├── 41-oidc-upstream-ca-certificate.yaml   # create-if-absent：仅首次安装
│   └── 45-nktp-cluster.yaml.tmpl
├── spire-server-nodeport/        # 清单：一个 Service
│   └── spire-server-nodeport.yaml
├── capnico/                      # 清单：CAPNICo provider，绑定到其镜像
│   ├── capnico.yaml
│   └── release.yaml
├── nktp/                         # helm：来自本检出的 chart
│   ├── release.yaml
│   └── values.yaml
├── temporal/                     # helm：来自公共仓库的 chart
│   ├── release.yaml
│   ├── values.yaml
│   └── overlay.yaml
└── ...
```

只读取 `nvke-deploy.yaml` 中列出的目录。根目录下的其他文件会被忽略。

## `nvke-deploy.yaml` 文件

```yaml
items:
  - {name: nvke-base-config, type: template}
  - {name: spire-tpm-pubhashes, type: manifest}
  - {name: spire-server-nodeport, type: manifest}
  - {name: spire-oidc-provider-nodeport, type: manifest}
  - {name: capnico, type: manifest}
  - {name: nktp, type: helm}
  - {name: postgresql, type: helm}
  - {name: spire-crds, type: helm}
  - {name: temporal, type: helm}
  - {name: spire, type: helm}
  - {name: controller, type: helm}
```

| 字段 | 必填 | 描述 |
|-------|----------|-------------|
| `name` | 是 | 条目在部署路径下的目录名。对于 `helm` 条目，也是其 chart 的默认 NVKR 构建产物名称。允许字母、数字、`-`、`_` 和内部点号。 |
| `type` | 是 | `template`、`manifest` 或 `helm`。 |

列表顺序即安装顺序。条目逐个应用，某个条目失败会停止运行。不做任何自动检测：未列出的条目不会被应用，`nvkectl` 和执行器自身的解析器都会在应用任何内容之前拒绝未知类型。

脚本通过 `deploy/lib/nvke-deploy-items.py` 读取条目列表和条目的 `release.yaml`；`nvkectl render deploy-path` 通过 `cli/internal/deploypath` 读取它们。两者都接受此处展示的语法。

## 条目类型

### `template`：站点密钥

模板条目包含 `.yaml.tmpl` 文件，旁边也可以有普通 `.yaml` 文件。每个 `.yaml.tmpl` 用 Go 的 [`text/template`](https://pkg.go.dev/text/template) 渲染，普通文件被复制，结果作为清单条目应用。部署路径借此创建堆栈所需的 Secret，而无需提交凭据。

模板上下文由站点规格、堆栈配置和安装环境提供：

| 表达式 | 来源 |
|------------|------------|
| `.ClusterName` | `clusterName` |
| `.Nico.Endpoint`、`.Nico.OrgID`、`.Nico.APIName` | 堆栈配置的 `nico.endpoint`、`.orgID`、`.apiName` |
| `.Nico.Token` | `NVKECTL_NICO_TOKEN` |
| `.Tenant.User` | `NVKECTL_NODE_USER` |
| `.Tenant.Password` | `NVKECTL_NODE_PASSWORD` |

引用其他字段的模板会被拒绝，空值也会被拒绝：否则会渲染出空凭据。包含引号、反斜杠或控制字符的值同样会被拒绝，因为每个值都落在带引号的 YAML 标量内。

`deploy/nico/nvke-base-config/20-nico-credentials.yaml.tmpl` 渲染 CAPNICo 读取的 Secret：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: nico-credentials
  namespace: capnico-system
type: Opaque
stringData:
  endpoint: "{{ .Nico.Endpoint }}"
  orgID: "{{ .Nico.OrgID }}"
  token: "{{ .Nico.Token }}"
  apiName: "{{ .Nico.APIName }}"
```

文件按名称顺序应用。使用数字前缀（`00-`、`20-`、`40-`），确保命名空间在任何内容落入之前已存在。

没有站点规格时，`install --config <file>` 从具有相同字段名的 JSON 文档渲染模板。e2e 通道使用这种方式；站点安装不需要。

### `manifest`：普通 Kubernetes 清单

清单条目包含 `.yaml` 文件。每个文件按名称顺序用 `kubectl apply --server-side` 应用。

清单条目可以携带 `release.yaml`，将其 YAML 运行的镜像绑定到 NVKR 构建产物。`deploy/nico/capnico/release.yaml`：

```yaml
image: ghcr.io/dsx-ai-factory/cluster-api-provider-nico/controller:v0.1.0
artifact: capnico-controller
```

`image` 是已提交 YAML 中指定的仓库和标签。`artifact` 是发布它的 NVKR 输出。针对发布版本渲染时会写入一个 `kustomization.yaml`，将镜像重新指向构建产物的仓库和摘要，脚本应用该 kustomization 而非文件本身。`release.yaml` 本身永远不会被应用。

#### `create-if-absent`：仅在首次安装时应用的文件

`.yaml` 文件可以通过文件任意位置的注释声明仅在它创建的对象尚不在集群中时才应用：

```yaml
# nvke-deploy: create-if-absent <kind>:<name>:<namespace>
```

应用该文件之前，脚本会读取该对象。如果存在，则跳过该文件并输出一行说明；如果不存在，则照常应用。探测是读取操作，因此在 `--dry-run` 下也会运行，试运行能描述将要发生的安装。除 `NotFound` 之外的任何原因导致的探测失败都会停止安装，而不是把无响应的 API server 当作"不存在"——在 `--dry-run` 下可能没有集群，此时会报告该情况。

这适用于少数站点需要持久保留且无法重新推导的对象，不适用于普通漂移，后者仍属于 apply 的范畴。目前只有一个：`41-oidc-upstream-ca-certificate.yaml` 保存 OIDC 上游 CA，站点的每个租户都已将其证书作为信任锚嵌入。对其 spec 的任何更改都会使 cert-manager 用新密钥重新签发 CA，而它为 OIDC 发现 provider 签发的叶子证书将不再链接到租户信任的证书。因此站点保留安装时使用的 CA——发布重命名将其主体改为 NVKE，只有在此之后安装的站点才会获得新的 CA。

**门控属于执行器，而非清单。** 对该文件执行 `kubectl apply -f`、同步该目录的 GitOps 控制器，或任何不读取该指令的其他工具，都会应用 Certificate 并重新签发 CA。`nvke-deploy.sh` 之外的执行器必须自行实施门控——参见[以其他方式应用](#以其他方式应用)。

携带该指令的文件必须是创建该对象的唯一位置，且命名的对象应是持有状态的对象，而非声明它的对象：CA 的门控在其 `Secret` 上，后者持有密钥材料，而非 `Certificate`。将此类文件与其链的其余部分分开，使不携带状态的部分——`40-oidc-upstream-ca.yaml` 中的两个 Issuer——在每次安装时继续被应用。

针对发布版本渲染的条目通过 `kustomization.yaml` 应用，这是单个流的单次 apply，不遵守该指令。只有将镜像绑定到发布版本的条目才会生成它，而这些条目都不携带该指令。

### `helm`：一个 chart 作为一个发布版本

helm 条目将一个 chart 安装为一个 Helm 发布版本。其 `release.yaml` 指定 chart、发布版本和 values：

```yaml
chart:
  name: temporal            # chart 的名称
  version: "0.74.0"         # 仓库 chart 必填；路径 chart 不指定
  repository: https://go.temporal.io/helm-charts   # 或 path: ../../nktp/deploy/charts/nktp
  artifact: temporal        # chart 的 NVKR 构建产物；默认：条目名称
  imageArtifact: cwe        # chart 运行的镜像的 NVKR 构建产物（如有）
  profilesArtifact: controller-profiles             # 及其 profiles 内容镜像
  setFile:                  # <helm key>=<绝对路径>，作为 --set-file 传递
    - some.chart.key=/etc/nvke/some-file
release:
  name: temporal
  namespace: temporal
values:
  - values.yaml             # 按顺序应用；后面的文件优先
  - overlay.yaml
requires:
  - {kind: certificate, name: oidc-provider-upstream-ca, namespace: spire-server, condition: Ready}
```

| 字段 | 描述 |
|-------|-------------|
| `chart.repository` 或 `chart.path` | chart 的来源。二者之一。仓库是 `https://` 索引或 `oci://` 引用。路径相对于部署路径，因此从 `deploy/nico` 可以以 `../../controller/deploy/charts/controller` 访问本检出中的 chart。 |
| `chart.version` | 在仓库中选择 chart。路径 chart 不指定版本：路径处的 chart 自带版本。 |
| `chart.artifact`、`chart.imageArtifact`、`chart.profilesArtifact` | 通过 NVKR 的输出名称将条目绑定到 NVKR 发布版本：chart、chart 运行的镜像及其 profiles 内容镜像。 |
| `chart.setFile` | 运维人员持有的文件，作为 `--set-file` 传递给 helm。部署路径携带路径，绝不携带内容。 |
| `release.name`、`release.namespace` | Helm 发布版本。命名空间必须已存在；`nvke-base-config` 创建 `deploy/nico` 使用的命名空间。 |
| `values` | 条目目录中的 values 文件，按顺序应用。渲染会追加 `site-values.yaml`，当发布版本携带条目的镜像时还会追加 `release-images.yaml`。`install --values-for <item>=<file>` 在这些之后添加文件。 |
| `requires` | 在 chart 安装之前必须存在并达到某条件的对象。`helm --wait` 只等待工作负载，因此由先前条目签发的 Certificate 在此声明，并用 `kubectl wait` 等待。 |

安装会为每个条目运行 `helm upgrade --install --wait`，使用条目的命名空间和 values，并对因服务器错误而失败的 chart 下载进行重试。有发布版本时，发布版本决定其携带的每个 chart 的版本，固定了其他版本的 `release.yaml` 会收到警告。固定版本仅对发布版本缺少的 chart 生效。

## 为站点和发布版本渲染

已提交的部署路径与站点无关，不指定任何摘要。`nvkectl render deploy-path` 写入针对站点规格、发布版本或两者解析后的副本。已提交的文件被复制，绝不编辑。副本包含：

- 每个接受站点值的条目中的 `site-values.yaml`，追加到其 `values`：信任域、管理 VIP、节点架构的 TPM 插件校验和、controller 和 profiles 镜像，以及每个 chart 读取凭据的 Secret 名称。任何 values 文件都不携带凭据。[`deploy/nico/README.md`](https://github.com/dsx-ai-factory/nvke/blob/main/deploy/nico/README.md#per-site-values) 列出了它们。
- `site-credentials/secrets.yaml`，渲染在第一个条目之后、命名空间存在后添加的清单条目：postgresql、temporal 和 controller 条目命名空间中的 `nvke-database` Secret，包含数据库密码和控制器的连接字符串；当站点使用 CA 时，spire 条目命名空间中的 `spire-upstream-ca`。
- 每个模板条目被渲染为清单条目。
- `charts/<item>/` 保存按摘要拉取的 chart 归档，适用于发布版本携带的每个 chart；条目的 `chart.path` 指向它。发布版本缺少的 chart 保持已提交状态。
- 每个其镜像构建产物由发布版本携带的 helm 条目中的 `release-images.yaml`；每个绑定到镜像构建产物的清单条目中的 `kustomization.yaml`。
- 根目录下：`images.txt`，结果运行的镜像；`nvkr-release.json`，发布版本的副本；以及默认 `--credentials rendered` 下忽略该树的 `.gitignore`。

默认 `--credentials rendered` 下，副本持有站点凭据。它仅对所有者可读，渲染会拒绝共享临时目录下的输出。`nvke-deploy.sh` 渲染到 `NVKE_STATE_DIR`（默认 `$XDG_STATE_HOME/nvke`）下的临时目录，并在退出时删除，因此不会留下渲染的凭据。要读取渲染结果，请自行运行命令：

```bash
nvkectl render deploy-path --deploy-path deploy/nico --sitespec sitespec.yaml --stack-config stack-config.json \
  --arch linux/arm64 --release release.json --out ~/.local/state/nvke/renders/my-site
```

对于位于仓库中的树，在该命令中添加 `--credentials external`。渲染随后不读取任何凭据变量和 CA 文件，不写入 Secret 和 `.gitignore`，也不写入本机的任何路径：发布版本不携带的 chart 保留其已提交的 `chart.path`，该路径从树安装的位置解析，因此使用 `--release` 渲染以将每个 chart 放入树中。它最后列出集群必须持有的 Secret：命名空间、名称、键，以及当类型不是 `Opaque` 时的类型。Secret 的命名空间由第一个条目 `nvke-base-config` 创建，因此你的密钥存储在该条目之后、读取它的条目之前交付每个 Secret，而非在安装开始之前。两种方式下 chart 命名相同的 Secret。`nvke-deploy.sh` 没有此选项：它始终渲染凭据并删除树。

`nvke-deploy.sh install --values-for ITEM=FILE` 在条目的 values 之上叠加文件。有一个键不能这样设置：controller 从 `nvke-database` Secret 读取其数据库连接字符串，该 Secret 由渲染根据部署路径的 `appDb` values 构建，因此设置 `appDb` 的文件会被忽略。执行器在应用任何内容之前拒绝它。在部署路径中修改 `appDb`，即 `deploy/nico/controller/overlay.yaml`，然后重新运行安装。

## 以其他方式应用

每个条目都是带 values 的 Helm chart 或普通 YAML，因此其他工具可以交付相同的条目。脚本在 `helm` 和 `kubectl apply` 之外做的事情很少，列在此处，以便其他执行器可以照做：

1. 按列表顺序应用条目；在第一个失败处停止。
2. 先渲染模板条目。在 GitOps 下，它们和 `site-credentials` 条目产生的 Secret 改由你的密钥存储提供；`--credentials external` 列出它们。每个 Secret 必须在 `nvke-base-config` 创建其命名空间之后、读取它的第一个条目之前存在于该命名空间中：在 sync-wave 设置中，位于两者之间的一个 wave。
3. 在每个 chart 之前等待其 `requires` 条目。在 GitOps 中，这是 sync-wave 边界或对象上的健康检查。
4. 使用本地 NVKR Run（`--nvkr-run-dir`）时，在第一个条目之前将 `images.txt` 中列出的镜像导入节点的 containerd。
5. 遵守 `create-if-absent`。携带该指令的文件仅在它命名的对象不在集群中时才应用；将其应用到已有该对象的站点会重新创建该对象。对于 `41-oidc-upstream-ca-certificate.yaml`，这意味着用新密钥重新签发 OIDC 上游 CA，使站点每个租户已嵌入的信任锚失效。在 GitOps 下，将该文件从现有站点的同步中排除，或使其同步以 `spire-server` 中不存在 `secret/oidc-provider-upstream-ca` 为条件；无法表达任一条件的工具不得将该文件交付给已有 CA 的站点。不要让集群读取失败代替"不存在"。

GitOps 工具将这些映射到 sync wave 和密钥存储；`nvkectl render deploy-path` 写入它交付的每站点 values 和渲染后的模板条目。部署路径周围的步骤，如 `clusterctl init` 和 SPIRE exec-auth 补丁，在同一指南中，保持命令式。

## 添加条目

1. 在部署路径下创建一个以条目命名的目录。
2. 添加其文件：helm 条目的 `release.yaml` 和 values，清单条目的 `.yaml` 文件，模板条目的 `.yaml.tmpl