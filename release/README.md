# release/

本目录存放发布与资格验证工具：编排流程接收一组已构建的候选构建产物，并确定一次资格验证运行将覆盖哪些内容。

> `qualify plan` 从仓库解析出真实计划并写入其 `plan.json`，`qualify verify` 检查计划中列出的所有内容均已发布。随后每个 stream 的 leg 用 `qualify result` 记录结果，并用 `qualify publish` 将其发送到 TestGrid。[`qualify-e2e` 工作流](../.github/workflows/qualify-e2e.yml) 运行全部四个命令，并为每个 stream 资格验证一个 job。

## 用法

`qualify plan` 解析要资格验证的内容并以 JSON 写出。从仓库根目录运行。

`registry.yaml` 将构建产物仓库命名为 `oci://nvcr.io/${QUALIFY_E2E_NGC_ORG}`，因此每个 `plan` 都需要该变量。它在加载文件时读取，未设置时会在任何解析开始前使命令失败。

```console
$ export QUALIFY_E2E_NGC_ORG=<the NGC org the artifacts are published under>

# The highest version of nvke, over every stream
$ go run ./release/cmd/qualify plan --subject nvke

# A specific tag, narrowed to the golden-path scope
$ go run ./release/cmd/qualify plan --subject nvke --version v0.22.0-beta.1 --scope golden-path

# Use a version you choose in place of what resolves
$ go run ./release/cmd/qualify plan --subject nvke --version v0.22.0-beta.1 --overrides controller=v0.1.1-rc.11,kube=1.36.3

# Write the plan to a file instead of stdout
$ go run ./release/cmd/qualify plan --subject nvke --output plan.json

# Qualify a beta: describe every component at the commit the tag marks
$ go run ./release/cmd/qualify plan --subject nvke --version v0.22.0-beta.1 --resolution commitDescribe

# Check that everything the plan names is published, waiting for any still building
$ NGC_API_KEY=<an NGC key that can read that org> \
    go run ./release/cmd/qualify verify --plan plan.json --wait 90m
```

`verify` 默认匿名读取镜像仓库，除非提供凭据；而两个仓库都包含私有仓库，因此没有凭据时每个 OCI 引用都会返回 401。使用 `--wait` 时，它会在整个预算时间内对每个引用重试，之后才失败。

| 变量 | 用途 |
|---|---|
| `NGC_API_KEY` | `nvcr.io`，用户名为字面量 `$oauthtoken` |
| `GITHUB_ACTOR`、`GITHUB_TOKEN` | `ghcr.io` |

| 标志 | 默认值 | 含义 |
|---|---|---|
| `--subject` | 必填 | 要资格验证的对象。`nvke` 表示整个 NVKE |
| `--version` | 最高版本 | 对象的版本，必须是已存在的标签 |
| `--overrides` | 无 | 替代解析结果的值，格式为逗号分隔的 `name=value` 对 |
| `--scope` | `full` | 覆盖矩阵的范围，取自 `policy.yaml` 中的 scope |
| `--output` | `-` | 计划写入位置。`-` 表示 stdout |
| `--resolution` | `commitDescribe` | 如何解析组件，以负责解析的策略命名。`commitDescribe` 在对象版本指向的提交处描述每个组件，这是 beta 的资格验证方式。`gitTag` 从各自的发布标签获取，发布资格验证必须选择它：保持默认时，发布会验证 describe 命名的构建产物，这些产物存在，因此会通过，而实际资格验证的却是非发布标签的构建 |
| `--check-exists` | 关闭 | 对每个解析出的位置发起请求并要求其已发布。关闭时将该问题留给 `qualify verify` |
| `--root`、`--policy`、`--registry` | 所在检出目录及 `release/` 下的文件 | 从何处读取仓库及这两个文件 |

不带 `--version` 时，对象的解析方式与每个组件相同，取任意类型的最高版本。传入 `--version` 以指定一个已存在的标签。

通过 `--overrides` 给出的值会替换该组件或依赖原本解析出的结果，并按解析结果的方式检查：标签必须存在，Kubernetes 或 OVN 版本必须是镜像锁定的版本。镜像或引导加载程序（如 `os-image-amd64`）是依赖，其覆盖值为 URL，因此也按名称覆盖。被覆盖的名称列在计划的 `overrides` 下。

源的 base 可以命名 `${VARIABLE}`，在读取文件时从环境变量替换。构建产物所在的镜像仓库命名的是某个组织的账户，而非 NVKE 的任何信息，因此文件声明其形式，由运行时提供值。未设置或为空的变量会在请求它的文件上失败，并一次性列出所有缺失的名称，而不是生成一个在无组织下命名构建产物的计划。

规划只访问 git。每个构建产物是否已发布由 `qualify verify` 对计划中列出的所有内容统一询问一次。`--check-exists` 则在解析期间询问，这需要能访问 bucket 的网络：该 bucket 是私有的，因此缺失的构建产物和不允许的网络都会返回 403，该检查只能确认构建产物存在，不能确认其不存在。

发布使用 `--resolution gitTag` 进行资格验证，从各自的发布标签获取每个组件。Beta 使用 `--resolution commitDescribe` 进行资格验证：beta 标签指向一个提交且不携带组件标签，因此标签会返回上一个发布版本而非该处构建的内容。提交从被资格验证的版本推导而来，而非直接给出。计划如下所示：

```console
$ go run ./release/cmd/qualify plan --subject nvke --version v0.21.0 --scope full
{
  "subject": "nvke",
  "version": "v0.21.0",
  "scope": "full",
  "combinations": [
    {
      "id": "capi-provisioned-v0.1.0_cli-v0.1.0_controller-v0.1.1_os-image-v0.1.0",
      "components": [
        { "name": "controller",       "version": "v0.1.1", "url": "oci://nvcr.io/<org>/nvke-controller:v0.1.1", "versionStrategy": "gitTag", "locationStrategy": "ociArtifact" },
        { "name": "cli",              "version": "v0.1.0", "versionStrategy": "gitTag" },
        { "name": "os-image",         "version": "v0.1.0", "versionStrategy": "gitTag" },
        { "name": "capi-provisioned", "version": "v0.1.0", "url": "oci://nvcr.io/<org>/cluster-api-provider-provisioned:v0.1.0", "versionStrategy": "gitTag", "locationStrategy": "ociArtifact" }
      ],
      "dependencies": [
        { "name": "cni-ovn",               "version": "2.4.2", "versionStrategy": "lockedPackage" },
        { "name": "os-image-bootloader-amd64", "url": "https://cdn.example/nke-isos/qcow/efi/x86_64/qcow-imager.efi", "locationStrategy": "url" },
        { "name": "os-image-bootloader-arm64", "url": "https://cdn.example/nke-isos/qcow/efi/aarch64/qcow-imager.efi", "locationStrategy": "url" },
        { "name": "cli-amd64",                 "version": "v0.1.0", "url": "oci://nvcr.io/<org>/nvkectl:v0.1.0-linux-amd64", "locationStrategy": "ociArtifact" },
        { "name": "cli-arm64",                 "version": "v0.1.0", "url": "oci://nvcr.io/<org>/nvkectl:v0.1.0-linux-arm64", "locationStrategy": "ociArtifact" },
        { "name": "kind",                      "version": "v0.32.0", "versionStrategy": "makeVar" },
        { "name": "clusterctl",                "version": "v1.12.3", "versionStrategy": "makeVar" },
        { "name": "nerdctl",                   "version": "2.3.5", "versionStrategy": "makeVar" }
      ],
      "streams": [
        {
          "name": "forge-gb200-amd64-ubuntu-24.04-kube-1.36",
          "capabilities": { "arch": "amd64", "gpu": "gb200", "kube": "1.36", "provider": "forge", "ubuntu": "24.04" },
          "dependencies": [
            { "name": "kube",           "version": "1.36.3", "versionStrategy": "lockedPackage" },
            { "name": "os-image-amd64", "version": "v0.1.0", "url": "https://cdn.example/nvke-packages/os-image/releases/v0.1.0/amd64/nvke-node-non-fde-ubuntu-24.04.5-amd64.qcow2", "locationStrategy": "osImage" },
            { "name": "os-image-arm64", "version": "v0.1.0", "url": "https://cdn.example/nvke-packages/os-image/releases/v0.1.0/arm64/nvke-node-non-fde-ubuntu-24.04.5-arm64.qcow2", "locationStrategy": "osImage" }
          ]
        },
        {
          "name": "forge-gb200-arm64-ubuntu-24.04-kube-1.36",
          "capabilities": { "arch": "arm64", "gpu": "gb200", "kube": "1.36", "provider": "forge", "ubuntu": "24.04" },
          "dependencies": [
            { "name": "kube",           "version": "1.36.3", "versionStrategy": "lockedPackage" },
            { "name": "os-image-amd64", "version": "v0.1.0", "url": "https://cdn.example/nvke-packages/os-image/releases/v0.1.0/amd64/nvke-node-non-fde-ubuntu-24.04.5-amd64.qcow2", "locationStrategy": "osImage" },
            { "name": "os-image-arm64", "version": "v0.1.0", "url": "https://cdn.example/nvke-packages/os-image/releases/v0.1.0/arm64/nvke-node-non-fde-ubuntu-24.04.5-arm64.qcow2", "locationStrategy": "osImage" }
          ]
        },
        {
          "name": "forge-gb200-amd64-ubuntu-26.04-kube-1.36",
          "capabilities": { "arch": "amd64", "gpu": "gb200", "kube": "1.36", "provider": "forge", "ubuntu": "26.04" },
          "dependencies": [
            { "name": "kube",           "version": "1.36.3", "versionStrategy": "lockedPackage" },
            { "name": "os-image-amd64", "version": "v0.1.0", "url": "https://cdn.example/nvke-packages/os-image/releases/v0.1.0/amd64/nvke-node-non-fde-ubuntu-26.04.1-amd64.qcow2", "locationStrategy": "osImage" },
            { "name": "os-image-arm64", "version": "v0.1.0", "url": "https://cdn.example/nvke-packages/os-image/releases/v0.1.0/arm64/nvke-node-non-fde-ubuntu-26.04.1-arm64.qcow2", "locationStrategy": "osImage" }
          ]
        },
        {
          "name": "forge-gb200-arm64-ubuntu-26.04-kube-1.36",
          "capabilities": { "arch": "arm64", "gpu": "gb200", "kube": "1.36", "provider": "forge", "ubuntu": "26.04" },
          "dependencies": [
            { "name": "kube",           "version": "1.36.3", "versionStrategy": "lockedPackage" },
            { "name": "os-image-amd64", "version": "v0.1.0", "url": "https://cdn.example/nvke-packages/os-image/releases/v0.1.0/amd64/nvke-node-non-fde-ubuntu-26.04.1-amd64.qcow2", "locationStrategy": "osImage" },
            { "name": "os-image-arm64", "version": "v0.1.0", "url": "https://cdn.example/nvke-packages/os-image/releases/v0.1.0/arm64/nvke-node-non-fde-ubuntu-26.04.1-arm64.qcow2", "locationStrategy": "osImage" }
          ]
        }
      ]
    }
  ]
}
```

## 计划如何使用

`plan.json` 是资格验证运行的输入。在 [`qualify-e2e` 工作流](../.github/workflows/qualify-e2e.yml) 中：

- `plan` job 运行 `qualify plan --subject nvke`。指定版本的 dispatch 输入和 `QUALIFY_E2E_*` 仓库变量会作为 `--overrides` 传入，因此计划会记录它们，job 无需猜测拿到的是哪一个。`--resolution` 默认为 `commitDescribe`，除非通过输入或 `QUALIFY_E2E_RESOLUTION` 选择 `gitTag`；beta 标签推送无论变量如何都采用 `commitDescribe`。计划作为构建产物上传。
- `qualify-run` 为每个组合的每个 stream 运行一次，从第一步起就受 GitHub environment 门控。它在占用任何机器之前对该计划运行 `qualify verify`，因此从未发布的构建产物会在此处失败，而不是在运行一小时后才失败——当计划在某个提交处解析时则等待，因为发布者可能仍在运行——然后解析版本和能力并构建站点输入。其版本、架构和 GPU 型号来自计划中该 stream 的条目，因此每个 stream 资格验证的正是计划解析出的内容。每个 job 以其 stream 命名。
- 套件运行后（或运行失败后），`qualify result` 从该 stream 的计划条目及其 Ginkgo 报告写出 `qualify-result.json`，在每个非 dry-run 的 leg 上执行——包括 PR leg 和基础设施故障，因此即使没有发布任何内容，记录也存在。随后 `qualify publish` 将其连同 JUnit 报告发送到 TestGrid——但仅在受信任的 ref（`main` 或发布/beta 标签）上，因为这是 testgrid 仓库的 WIF provider 唯一会交换令牌的身份。带标签的 PR leg 会获得结果记录作为工作流构建产物，TestGrid 上则没有内容。

计划携带 job 所需的内容：要测试的版本以及要运行的 stream 的能力。e2e 套件要求 Forge 运行使用的每个版本（`controller`、`capi-provisioned`、`kube`、`cni-ovn`、`kind`、`clusterctl`、`nerdctl` 以及镜像和引导加载程序 URL），缺少任何一个都会在机器存在之前失败。它不携带这些的默认值，因此计划是版本的唯一来源。

## 这不是什么

`build/` 为一个组件生成构建产物。`release/` 从不构建任何东西。它消费已存在的构建产物，本目录下的任何内容将源码编译为交付构建产物都是错误。这种分离正是资格验证有意义的原因：发布推广的是经过测试的确切字节，而能够重新构建的工具就是能够替换的工具。

按组件的发布保持原样。`agent/v*`、`os-image/v*`、`controller/v*` 和 `cli/v*` 标签继续完全按照 [`RELEASE.md`](../RELEASE.md) 所述驱动各自的工作流。`release/` 在这些发布之上进行编排；它不替代它们。

## 布局

| 路径 | 内容 |
|---|---|
| [`registry.yaml`](registry.yaml) | 每个组件如何解析到版本，以及构建产物发布所在的命名位置。它持有查找版本的规则，从不持有版本本身 |
| [`policy.yaml`](policy.yaml) | 存在的 stream 及其上的 scope |
| [`qualification.yaml`](qualification.yaml) | 当 NVKE 作为整体进行资格验证时验证什么：一起测试的组件，以及运行所需的其他版本 |
| [`cmd/qualify/`](cmd/qualify/) | 二进制文件。`plan` 解析要资格验证的内容，`verify` 检查其已发布 |
| [`internal/qualification/`](internal/qualification/) | 读取和检查这三个文件，并将其转换为计划 |
| [`internal/resolve/`](internal/resolve/) | 解析策略。对本仓库只取 `internal/artifact` |
| [`internal/artifact/`](internal/artifact/) | 通过 HTTPS 或镜像仓库询问某个位置是否包含内容，并等待仍在发布的内容 |
| [`hack/next-tag.sh`](hack/next-tag.sh) | 下一个 beta 或候选发布标签获得的版本，基于最新 GA 标签以来的提交。[`test_next_tag.py`](hack/test_next_tag.py) 针对 fixture 仓库测试它 |

每个文件只关注一件事。单独进行资格验证的组件会在其代码旁携带自己的 `<component>/qualification.yaml`，使用相同的 Schema，并共享 registry 和 policy。

## 对象、组件和依赖

`qualification.yaml` 说明一次资格验证覆盖什么。它为**对象**（被资格验证的东西