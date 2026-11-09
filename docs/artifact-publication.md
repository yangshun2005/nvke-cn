# 构建产物发布参考

本文档介绍 NVKE 现有的构建产物构建与发布工作流。仅源码的 OSS 发布流程及其经验证的 SBOM 要求见 [RELEASE.md](../RELEASE.md)。构建产物发布不属于该源码发布交付物的要求。下文路径和 shell 命令均相对于仓库根目录。本文描述的工作流行为不因仅源码文档范围而禁用。

## 版本管理

NVKE 使用[语义化版本](https://semver.org/)：

```
v<MAJOR>.<MINOR>.<PATCH>
```

| 递增位 | 适用场景 |
|---|---|
| `PATCH` | 向后兼容的缺陷修复 |
| `MINOR` | 新增向后兼容的功能 |
| `MAJOR` | 破坏性 API 或行为变更 |

何为破坏性变更因组件而异，因此各组件自行定义其范围。对 `nvkectl` 而言即 [CLI 兼容性策略](../cli/docs/compatibility.md)，其中列出了这些递增位适用的命令、标志、退出码和输出格式，以及移除前必须经历的弃用流程。

## 发布节奏

没有固定发布计划。当 `main` 上积累了足够多的功能或修复，或安全补丁需要立即带外发布时，才会发布。

## 创建发布版本

### 1. 打标签

裸标签发布整个仓库：

```bash
git tag -s v0.2.0 -m "Release v0.2.0"
git push upstream v0.2.0
```

单个组件以其自身目录作为前缀打标签：

```bash
git tag -s cli/v0.2.0 -m "Release cli/v0.2.0"
git push upstream cli/v0.2.0
```

前缀是模块目录，因为 Go 要求如此：模块 `github.com/dsx-ai-factory/nvke/cli` 只能由名为 `cli/vX.Y.Z` 的标签解析。

没有工作流会创建 GitHub Release。标签通过以下工作流发布构建产物：

| 标签 | 工作流 | 发布内容 |
|---|---|---|
| `agent/vX.Y.Z` | `agent-deb-publish.yml`（+ `agent-artifact.yml`） | `.deb` 到 apt 入口、GHCR 二进制和 `.deb`、NVCR 二进制 + `.deb`、S3/CloudFront `.deb` |
| `os-image/vX.Y.Z` | `os-image-artifact.yml` | qcow2 到 NVCR；qcow2 及其 SBOM 和校验和，以及 `nvke-os-image-check`，到 S3/CloudFront；仅当 `NVKE_GITHUB_PUBLISH` 包含 `os-image` 时，版本标签的 qcow2 到 GHCR |
| `nktp/vX.Y.Z` | `nktp-chart-publish.yml` | Helm chart 到 GHCR |
| `cli/vX.Y.Z` | `tag-publish.yml`（+ `cli-artifact.yml`、`cli-artifact-nvcr.yml`、`cli-s3-publish.yml`） | 二进制到 GHCR、NVCR、S3/CloudFront |
| `controller/vX.Y.Z` | `image-nvcr.yml`、`chart-publish.yml`、`chart-publish-nvcr.yml` | 镜像和 chart；见下文 |

裸 `vX.Y.Z` 标签不发布任何内容；`tag-publish.yml` 仅对其运行许可证检查。

### 关闭 GHCR 发布

Actions 变量 `NVKE_GITHUB_PUBLISH` 列出其发布标签构建产物发往 GHCR 的组件。未设置时表示 `root,controller,cli,agent,cluster-api-provider-provisioned,nktp`，即除 `os-image` 外的所有组件，`os-image` 的 qcow2 仅发往 NVCR 和 S3。设为 `none` 可停止所有 GHCR 发布版本发布，或列出所需组件。NVCR、S3 和 CloudFront 不受影响，GHCR 上的 CI 交接也不受影响：`agent` 和 `cli` 的 `sha-` 和 `edge` 别名，以及 `os-image` 的 `sha-` 和 `main-sha-` 标签 qcow2 继续流转，因为 `agent/Makefile`、NVCR 和 S3 发布器以及 NICo VM 通道会读取它们。仅版本标签别名停止。该变量必须是仓库或组织变量；环境变量对作业条件不可见。

在仓库的 Settings、Actions、Variables 下设置，或使用 `gh variable set NVKE_GITHUB_PUBLISH --body 'controller,cli'`。空值视为未设置，使用默认列表；仅 `none` 表示什么都不发布。名称以逗号分隔，无空格：

| 名称 | 标签 | 列出该名称时 GHCR 获得的内容 |
|---|---|---|
| `root` | 裸 `vX.Y.Z` | 无；接受该名称以使现有值继续有效 |
| `controller` | `controller/v*` | `nvke-controller` 镜像和 `charts/nvke-controller` chart |
| `cli` | `cli/v*` | `cli` 二进制 |
| `agent` | `agent/v*` | `agent` 和 `agent-deb` |
| `cluster-api-provider-provisioned` | `cluster-api-provider-provisioned/v*` | 提供者镜像 |
| `nktp` | `nktp/v*` | chart |
| `os-image` | `os-image/v*` | 版本标签的 qcow2 |

`os-image/` 标签若 Packer 或启动冒烟测试失败，则永不发布 OCI 构建产物。

`nktp/` 标签发布 NVKE 遥测流水线（NKTP）Helm chart。chart 版本即标签版本：`nktp/vX.Y.Z` 将 chart 打包为 `X.Y.Z`，`nktp/vX.Y.Z-rc.1` 打包为 `X.Y.Z-rc.1`，与 NVKE controller chart 从 `controller/v*` 取版本的方式相同。`nktp/deploy/charts/nktp/Chart.yaml` 中的 `version:` 是占位符，打标签前无需编辑。工作流推送 `oci://ghcr.io/dsx-ai-factory/nvke/charts/nktp:<version>`。

`controller/vX.Y.Z` 标签发布 controller 镜像和 Helm chart：

| 工作流 | 发布内容 |
|---|---|
| `image-nvcr.yml` | 多架构镜像到 `nvcr.io/<org>/nvke-controller`，将该已发布镜像提升到 DSX 为 `nvcr.io/<dsx-org-team>/nvke-controller`，并将发布标签提升到 `ghcr.io/dsx-ai-factory/nvke/nvke-controller` |
| `chart-publish.yml` | chart 到 `ghcr.io/dsx-ai-factory/nvke/charts/nvke-controller` |
| `chart-publish-nvcr.yml` | chart 到 `nvcr.io/<org>/nvke-controller-chart` 和 `nvcr.io/<dsx-org-team>/nvke-controller-chart` |
| `image-validate-nvcr.yml` | 多架构站点验证镜像（`controller/test/e2e/Dockerfile`）直接推送到 DSX 为 `nvcr.io/<dsx-org-team>/nvke-validate`，使用 controller 镜像的版本 |

每个 `main` 提交都已将镜像和 chart 发布到 NVCR。GHCR 仅接收发布版本，因此从标签构建的 chart 可安全指向 `ghcr.io/dsx-ai-factory/nvke/nvke-controller:<version>`，而从 `main` 构建的 chart 则不能。2026-09-12 仓库迁移到 `dsx-ai-factory` 之前的发布版本被提升到 `ghcr.io/nvidia/nke/controller`：`controller/v0.1.1-rc.1` 到 `rc.3` 在那里，`controller/v0.1.0` 早于提升作业，`rc.4` 提升失败，因此该镜像不存在于任何地方。之后的发布版本落在 `ghcr.io/dsx-ai-factory/nvke` 下。

### 镜像仓库和拉取密钥名称

controller 镜像在所有镜像仓库中均为 `nvke-controller`。chart 在 GHCR 上为 `nvke-controller`，在 NVCR 上为 `nvke-controller-chart`。此次重命名之前发布的 GHCR 包已不存在：镜像 `nke-controller` 和 `controller`，以及 chart `charts/cloud-workflow-engine`、`charts/controller` 和 `charts/nke-controller`。没有以新名称重新发布任何内容。GHCR 从 `controller/v0.3.0-rc.1` 起持有 controller，因此固定到更早标签或 `nke-controller` 的引用在那里无法解析，硬件通道无法从 GHCR 验证更早的发布版本。

拉取密钥 `svc-nke-ngc-imagepull` 保留其名称。它是在此仓库之外配置的 NGC 服务账号，因此重命名意味着要在每个站点与 chart 引用同步重新配置。`chart-publish-nvcr.yml` 仅将其添加到 NVCR 发布的 chart；GHCR chart 默认无拉取密钥。

新的 GHCR 包创建为 `internal`。在首次发布该包的发布版本后，在包设置中将其设为公开。

DSX 从 `image-nvcr.yml` 接收 controller 镜像，在 NVKE 多架构清单存在之后。它从 `chart-publish-nvcr.yml` 接收 controller chart，从 `image-validate-nvcr.yml` 接收测试该 controller 的验证镜像，使用相同版本标签，以便固定 controller 的站点可运行从同一提交构建的测试。三者均使用来自同一受保护 GitHub 环境的独立凭据：

| 名称 | 类型 | 用途 |
|---|---|---|
| `NVKE_NGC_ORG` | 变量 | `nvcr.io/<org>/nvke-controller` 的源 NGC 组织 |
| `NGC_REGISTRY_TOKEN_NVKE` | 密钥 | 源 NVKE NGC 组织的读写令牌 |
| `DSX_NGC_ORG_TEAM` | 变量 | 目标 NGC 组织/团队，例如 `<org>/components-dev` |
| `NGC_REGISTRY_TOKEN_DSX` | 密钥 | DSX NGC 组织/团队的写令牌 |

源和目标都在 `nvcr.io` 上，但可能需要不同的 NGC 令牌。Docker 认证每个镜像仓库主机存储一个凭据，因此 DSX 提升使用 NVKE 令牌将源镜像拉取到本地 OCI 布局，然后使用 DSX 令牌登录，并将同一索引推送到 `nvcr.io/<dsx-org-team>/nvke-controller`。不可变标签（`sha-<commit>` 和 git 描述版本）仅在现有目标摘要与源摘要匹配时才视为幂等；不匹配则运行失败。`edge` 标签可变，仅在 `main` 上移动。

DSX chart 从 controller chart 打包为 `nvke-controller-chart`，与 NVKE NVCR chart 使用的 chart 构建产物名称相同，并推送到 `oci://nvcr.io/<dsx-org-team>`。其 chart 版本是 controller git 描述字符串去掉前导 `v`，而 `appVersion` 保留前导 `v`，以便渲染的 Deployment 拉取 `image-nvcr.yml` 提升的同一镜像标签。

将标签推送到 `upstream`。推送到 Fork 的标签仍会在那里启动 `tag-publish.yml`，运行许可证检查（对于 `cli/` 标签，还会构建 nvkectl），但不发布任何内容：每个发布器都以其发布作业基于本仓库 ID 进行保护。

### 2. 验证构建产物

标签推送后，确认任何 CI 发布流水线成功完成（节点 OS 镜像烘焙、chart 发布等）。对于 `os-image/v*`，即 **OS Image Artifact** 检查：Packer、启动冒烟测试，然后是 NVCR 和 S3 发布器（仅当 `NVKE_GITHUB_PUBLISH` 包含 `os-image` 时 GHCR）。对于 `controller/v*`，确认 **Image NVCR** 检查，以及当 `NVKE_GITHUB_PUBLISH` 包含 `controller` 时，确认 **Chart Publish** 检查和 `ghcr.io/dsx-ai-factory/nvke/nvke-controller:<version>` 上提升的镜像。对于 `nktp/v*`，确认 **NKTP Chart Publish** 检查。

## 验证已发布镜像

容器镜像携带基于其索引摘要的[构建来源证明](https://docs.github.com/actions/security-guides/using-artifact-attestations-to-establish-provenance-for-builds)。它记录了构建该镜像的工作流、ref 和提交。

```bash
gh attestation verify \
  oci://ghcr.io/dsx-ai-factory/nvke/nvke-controller@sha256:<digest> \
  --repo dsx-ai-factory/nvke \
  --bundle-from-oci
```

`--bundle-from-oci` 读取附加到 GHCR 镜像的证明。不带该参数时，命令从 GitHub 获取证明。

## 软件物料清单

以下列出的构建产物有 SBOM；Helm chart 没有自己的 SBOM。格式为 **SPDX 2.3，JSON 格式**，由 [syft](https://github.com/anchore/syft) 在 `make/tool-versions.mk` 中固定的版本生成——`make print-SYFT_VERSION`。一个文档描述一个构建产物，每个文档都从已构建的构建产物生成，而非从此源码树生成，因此它记录的是实际构建的内容，而非此提交的构建应当选择的内容。

| 构建产物 | 文档 | 主体粒度 | 位置 |
|---|---|---|---|
| `nvke-node-{fde,non-fde}-ubuntu-<rel>-<arch>.qcow2` | `nvke-node-…-<arch>.spdx.json` | dpkg 包，来自镜像自身的包数据库 | S3，磁盘旁 |
| `nvke-controller` 镜像 | `controller-sbom-linux-<arch>.spdx.json` | 镜像内容，从已推送镜像扫描 | **Image NVCR** 运行的工作流构建产物 |
| `cluster-api-provider-provisioned` 镜像 | `cluster-api-provider-provisioned-sbom-linux-<arch>.spdx.json` | 镜像内容，从已推送镜像扫描 | **Image Provisioned NVCR** 运行的工作流构建产物 |

Helm chart——`nvke-controller` 和 `nktp`——没有自己的 SBOM。chart 不携带组件；它携带对镜像的引用，该镜像有 SBOM，即上表对应行。chart 归档的 SBOM 只会列出归档本身。

粒度因构建产物而异，因为有用的问题不同。对于节点镜像，问题是"此镜像是否包含 <组件>"，所有进入镜像的内容——归档包和我们自己的 `.deb`——都是 dpkg 条目，因此文档即镜像的包集合。对于容器镜像，则是镜像的内容。

`nvkectl` 和 agent `.deb` 没有已发布的 SBOM。

### 获取单个 SBOM

节点镜像 SBOM 位于 S3 上其磁盘旁，名称与磁盘相同，`.qcow2` 替换为 `.spdx.json`。

回答"我们的哪个构建产物包含 `<组件>`"即对下载的文档进行查询，例如使用 `jq`：

```bash
jq -r --arg c openssl \
  '.packages[] | select(.name | test($c)) | "\(.name) \(.versionInfo)"' \
  *.spdx.json
```

两个容器镜像由其发布工作流扫描，因此其文档是运行上的构建产物——`gh run download <run-id> --name controller-sbom-linux-amd64.spdx.json`。

**按摘要而非标签验证镜像**。标签是可变的指针；证明基于摘要，因此移动的标签不再指向被证明的对象。`crane digest <image>:<tag>` 解析当前摘要。

`--repo` 不是可选装饰。它将证明绑定到*本*仓库；没有身份约束，你只能确定构建产物是由某处的某个 GitHub 工作流构建的。使用 `--signer-workflow` 可进一步收紧。

### 被证明的内容

| 构建产物 | 被证明主体 | 证明所在位置 |
|---|---|---|
| GHCR 上的 `nvke-controller`、`cluster-api-provider-provisioned` | 多架构索引摘要 | 作为 OCI 引用附加到 GHCR 中的镜像 |
| NVCR 上的同两个镜像 | 同一索引摘要 | 仅 GitHub 的证明存储 |

证明在构建产物落地处生成。`crane copy` 保留镜像摘要但不携带其引用，因此每次 GHCR 提升都对其刚推送的镜像进行证明，而非依赖源。

### 提升前验证

提升是候选版本成为权威的时刻，因此提升作业进行验证而非假设。`.github/actions/verify-attestation` 封装 `gh attestation verify`，在证据缺失、无法验证或不匹配时使作业失败；`image-nvcr.yml` 和 `image-provisioned-nvcr.yml` 在三处调用它：

1. **GHCR 复制前**和**DSX 提升前**，基于 `manifest` 作业发布并证明的 NVCR 索引摘要。通过前不复制任何内容。
2. **GHCR 复制后**，使用 `--bundle-from-oci`，使检查读取 GHCR 提供的引用——与消费者相同的路径——而非 GitHub API，后者会从同一摘要的 NVCR 证明作答，无法证明实际落地的内容。

每次调用都固定仓库、签名工作流、源提交和源 ref，并从已验证的声明中重新读取被证明主体摘要。可变别名——GHCR 上的 `sha-<commit>`、DSX 上的 `edge`——仅在其上所有门禁通过后才移动，因此别名永不指向运行尚未完成担保的镜像。两条路径都不重建其提升的构建产物：`crane copy` 和 `crane push` 移动 `manifest`