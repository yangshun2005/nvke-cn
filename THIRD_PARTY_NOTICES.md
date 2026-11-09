# 第三方声明

NVKE 是一个由独立构建、独立发布的组件组成的 monorepo。每个组件链接的第三方 Go 模块各不相同，因此声明按**组件**分别生成，而非合并为单一文件。合并会把每个模块的署名附加到每个构建产物上，从而错误地描述每个产物实际包含的内容。

本文件是索引，记录哪些组件已生成声明、声明所在位置，以及哪些尚未生成——以及原因。

## 已覆盖的组件

| 组件 | 声明 | 生成来源 | 强制机制 |
|---|---|---|---|
| `agent/` — 节点代理 | [agent/THIRD_PARTY_NOTICES.md](agent/THIRD_PARTY_NOTICES.md) | `linux/amd64` 和 `linux/arm64` 下 `./cmd/...` 的导入闭包 | `Third-Party Notices` 工作流，每次推送时运行 |
| `controller/` — 云工作流引擎 | [controller/THIRD_PARTY_NOTICES.md](controller/THIRD_PARTY_NOTICES.md) — **151 个模块** | `linux/amd64` 下 `./cmd/worker` 和 provider 的 `./cmd/manager` 的导入闭包 | `Third-Party Notices` 工作流，每次推送时运行 |
| `cli/` — `nvkectl` CLI | [cli/THIRD_PARTY_NOTICES.md](cli/THIRD_PARTY_NOTICES.md) — **24 个模块** | `linux/amd64` 下 `./cmd/nvkectl` 的导入闭包 | `Third-Party Notices` 工作流，每次推送时运行 |
| `os-image/` — `nvke-os-image-check` 二进制 | [os-image/THIRD_PARTY_NOTICES.md](os-image/THIRD_PARTY_NOTICES.md) — **9 个模块** | `linux/amd64` 下 `./cmd/nvke-os-image-check` 的导入闭包 | `Third-Party Notices` 工作流，每次推送时运行 |

重新生成：

```bash
cd agent      && make notices
cd controller && make notices
cd cli        && make notices
cd os-image   && make notices
```

⚠️ **controller、nvkectl 和 os-image 的生成器需要开启 workspace**（不要设置 `GOWORK=off`），因为 `os-image` 依赖 `github.com/dsx-ai-factory/nvke/agent`，而后者只能通过 workspace 的 `use` 指令解析；关闭 workspace 时，`go-licenses` 会在收集任何内容之前就以 `go.mod file not found` 失败。

`make notices-check` 会重新生成文件，若结果与已提交内容不同则失败。CI 在每次推送时运行它，因此文件无法在构建不失败的情况下偏离依赖树。

## 尚未覆盖

agent、controller、CLI 和 os-image 二进制均带有生成的署名，CI 在每次推送时重新生成并比对全部四个。

由 `image-provisioned-nvcr.yml` 发布的 `cluster-api-provider-provisioned` 镜像附带 `controller/THIRD_PARTY_NOTICES.md`。该文件由 CWE worker 和 provider 的 manager（`./cmd/manager`）共同生成，因此覆盖 provider 的整个依赖闭包，controller 的声明检查会在其偏离时失败。provider 的闭包为 67 个模块，`linux/amd64` 和 `linux/arm64` 为同一集合，且全部包含在文件中。生成器仅分析 `linux/amd64`；要检查 `linux/arm64`，请将以下命令的输出

```bash
cd cluster-api-provider-provisioned
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go list -deps \
  -f '{{with .Module}}{{if not .Main}}{{.Path}}@{{.Version}}{{end}}{{end}}' ./cmd/manager | sort -u
```

与 `controller/THIRD_PARTY_NOTICES.md` 中索引的 Module 列进行比对。

`management-engine` manager 镜像和 chart 不属于初始开源发布的一部分。本仓库中没有任何工作流构建或发布它们，`Dockerfile` 和 `make docker-build` / `docker-push` 目标用于开发，镜像名由调用方提供。该模块的 API 类型被编译进 controller，其 CRD 随 controller chart 发布，因此这些内容带有 controller 的署名。若 manager 镜像被发布，它需要 `LICENSE`、`NOTICE` 和生成的依赖清单，并在上表中增加一行。

`capd-reboot-controller` 由 `make build` 构建，但不随任何内容发布——无镜像、无构建产物、无包。若开始发布，它需要在上表中增加一行，并拥有自己的 `notices` 目标。

⚠️ **本节在 2026-08-16 之前列出了两个未解决的缺口**，描述了一个存在但未经验证的 `controller/THIRD_PARTY_NOTICES.md`，以及 `nvkectl` 缺失的声明。两者均已在下方关闭。**部分或错误的文件比没有更糟**——它宣称了自身并不具备的完整性——因此这些行一直保持开放，直到每个闭包都能端到端走通*且*有检查强制执行。

### controller 行是如何关闭的

它长期处于开放状态，原因是 *"`controller/go.mod` 依赖两个非公开模块"*。当 [#76](https://github.com/dsx-ai-factory/nvke/pull/76) 移除它们后，这一原因不再成立，只剩一个当时为内部、现已公开的模块 `cluster-api-provider-nico`。此后有两个机械性问题使其保持开放，两者都值得记录，因为都不显而易见：

**`go-licenses` 要求其 rootDir 包含 vendor 树。** 本仓库在 workspace 根目录、每个模块之上统一 vendor 一次，因此从 `controller/` 运行生成器会直接失败，报 `rootDir /…/controller should contain dir /…/vendor/k8s.io/…`。生成器现在会向上查找到拥有 `vendor/` 的目录，并以该目录为基准命名包，因此一份实现即可服务所有模块。

**`go-licenses` 必须使用与 `go list` 相同的工具链构建。** 版本不匹配时，它会把每个标准库包报告为 *"does not have module info"* 并产出空结果——这看起来像生成器损坏，而非版本偏差。`controller/Makefile` 已固定其版本。

⚠️ **文件存在不等于文件正确。** 此前曾从 GitLab 导入过一个 `controller/THIRD_PARTY_NOTICES.md`，它署名的模块在此处的任何 `go.sum` 中都不存在、也未 vendor，同时遗漏了其他模块。**错误的声明文件比缺失的更糟，因为它看起来合规。** CI 中的检查正是区别所在。

## 复制的源文件

生成的声明仅覆盖 Go 模块。`controller/internal/kubeadmconfig/thirdparty/` 下的以下五个文件复制自 [kubernetes/kubernetes](https://github.com/kubernetes/kubernetes) 的 **v1.34.0**（提交 `f28b4c9efbca5c5c0af716d9f2d5702667ee8a45`），经修改，并保留其上游版权和许可证块。Kubernetes 在该修订版未提供 `NOTICE` 文件。复制时未记录修订版；该修订版为推断所得，副本与 v1.33 和 v1.34 均匹配。

| 本地文件 | 上游文件 | 上游版权 |
|---|---|---|
| `bootstraptoken/v1/types.go` | `cmd/kubeadm/app/apis/bootstraptoken/v1/types.go`、`utils.go` | Copyright 2021 The Kubernetes Authors. |
| `kubeadmtypes/v1beta3/types.go` | `cmd/kubeadm/app/apis/kubeadm/v1beta3/types.go` | Copyright 2021 The Kubernetes Authors. |
| `kubeadmtypes/v1beta4/types.go` | `cmd/kubeadm/app/apis/kubeadm/v1beta4/types.go` | Copyright 2023 The Kubernetes Authors. |
| `kubeletconfig/v1beta1/types.go` | `staging/src/k8s.io/kubelet/config/v1beta1/types.go` | Copyright 2017 The Kubernetes Authors. |
| `kubeproxyconfig/v1alpha1/types.go` | `staging/src/k8s.io/kube-proxy/config/v1alpha1/types.go` | Copyright 2017 The Kubernetes Authors. |

许可证：Apache License, Version 2.0，条款与本仓库的 [LICENSE](LICENSE) 相同；原始文件为 [v1.34.0/LICENSE](https://github.com/kubernetes/kubernetes/blob/v1.34.0/LICENSE)。

## 内嵌的前端库

nico-sim UI 在 `controller/nico-sim/ui/vendor/` 下打包了 xterm.js（含其 fit 和 search 插件）、highlight.js 以及 Nord highlight.js 主题。其组件名称、版本、来源、版权声明和完整许可证文本见 [controller/nico-sim/ui/vendor/THIRD_PARTY_NOTICES.md](controller/nico-sim/ui/vendor/THIRD_PARTY_NOTICES.md)。

## 生成文件的范围

每个生成文件列出该组件已发布命令可触达的每个 Go 模块（覆盖其发布的所有平台），并附每个许可证的完整文本。它排除：

- Go 标准库。
- 属于该模块自身的包。
- 已发布命令无法触达的仅测试和构建时依赖，因为它们不被分发。

## 另见

- [NOTICE](NOTICE) — 本仓库的署名声明，依据 Apache License, Version 2.0 的要求
- [LICENSE](LICENSE) — 本仓库分发所依据的许可证