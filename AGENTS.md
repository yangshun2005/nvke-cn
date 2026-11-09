# Agent 指南 — NVIDIA Kubernetes Engine (NVKE)

面向在本仓库中工作的 AI 编码代理（Claude Code、Codex、Copilot Workspace、Cursor）的指南。人类读者请阅读 [CONTRIBUTING.md](CONTRIBUTING.md)；本文件补充代理在开始编辑前需要遵守的约束。

`AGENTS.md` 是权威文件。每个 `AGENTS.md` 旁边的 `CLAUDE.md` 是一行 `@AGENTS.md` 导入垫片，供 Claude Code 使用——它不直接读取 `AGENTS.md`。请保持垫片无内容，以免不同工具看到不同的指令。

## 必读

- [README.md](README.md) — NVKE 是什么，各部分如何协同
- [CONTRIBUTING.md](CONTRIBUTING.md) — 分支命名、提交格式、评审流程
- **你正在编辑的模块的 `AGENTS.md`。** 这是 monorepo，每个模块有自己的工具链：
  - [controller/AGENTS.md](controller/AGENTS.md) — Flox、代码生成、数据库、Temporal
  - [agent/AGENTS.md](agent/AGENTS.md) — proto 生成、打包、gRPC 约定
  - [cli/AGENTS.md](cli/AGENTS.md) — nvkectl、验证、文档注释约定
  - [os-image/AGENTS.md](os-image/AGENTS.md) — 参考节点 OS 镜像规格、Packer、连通性检查器
  - [nktp/AGENTS.md](nktp/AGENTS.md) — NKTP Helm chart：版本管理、依赖固定
  - [controller/nico-sim/AGENTS.md](controller/nico-sim/AGENTS.md) — 本地 NICo 模拟器，支持 VM 和容器后端
  - [controller/deploy/dashboards/AGENTS.md](controller/deploy/dashboards/AGENTS.md) — Grafana 参考 JSON（仅导入，不属于 nke-stack）；运维人员请阅读：[README.md](controller/deploy/dashboards/README.md)

## 按需阅读

- [docs/architecture.md](docs/architecture.md) — 模块、数据流、部署模型
- [docs/getting-started.md](docs/getting-started.md) — 本地开发环境搭建
- [SECURITY.md](SECURITY.md) · [GOVERNANCE.md](GOVERNANCE.md) · [RELEASE.md](RELEASE.md) · [MAINTAINERS.md](MAINTAINERS.md)

## 项目简介

NVKE 是裸金属 GPU Kubernetes 的管理平面。种子集群搭建高可用控制平面，控制平面再按需创建 GPU 租户集群。主要组件：

- **`controller/`** — NVKE 控制器（原 CWE）。通过 Temporal 工作流编排集群生命周期，后端为 PostgreSQL，提供 OpenAPI REST 接口。
- **`agent/`** — 节点代理。以 systemd 服务形式在每个节点上运行的 gRPC 服务器。驱动 `kubeadm`、安装包和 bundle，通过 SPIFFE/SPIRE mTLS 认证。
- **`cluster-api-provider-provisioned/`** — Cluster API 基础设施提供者，用于 NVKE 在 CAPI 引导之前预置的主机。

## NVKE 控制器的命名

`controller/` 是一个名为 **NVKE controller**（`nvke-controller`）的服务。单独的 "controller" 是 reconciler 的通用词：management-engine 的 NVKECluster reconciler、控制器自身的 `controller/internal/controllers`，以及每个 `controller-manager` Deployment。其他组件引用该服务时，使用全名，以免与上述混淆：

| 位置 | 写法 |
|---|---|
| 正文 | NVKE controller；同一页面后续可简称 "the controller" |
| 标志、Kubernetes 对象名、文件、Tilt 资源 | `nvke-controller` |
| 环境变量设置 | `NVKE_CONTROLLER_*` |
| Go 包 | `nvkecontroller` |
| Go 标识符 | `NVKEController…` |

`controller/` 内部的代码引用自身，因此在那里用 "controller" 即可。`controller/` 目录及其 Go 模块路径保留原名：它们是代码布局，不是其他组件引用的名称。`NKE_CONTROLLER_*` 设置和 nvkectl 的 controller 标志等现有名称在其各自的 Pull Request 中变更，并附带各自的兼容步骤；不要在其他工作中顺带重命名它们。

## 构建前必读

**每个模块无需 NVIDIA 凭据即可构建。** 从干净检出开始，只要能访问公共 Go 模块：

```bash
make build
```

即可编译所有模块并链接二进制文件。Go 在首次下载后缓存公共模块。

这一变化来自 #76，该 PR 用公共的 `github.com/NVIDIA/ncx-infra-controller-rest` 替换了内部 carbide SDK（`nvmetal/cloud-api-sdk-go`，通过 Forge 提供者访问）。在此之前，`controller/` 只需要用公共依赖构建的 `agent/`。

**不再有内部依赖。** `github.com/dsx-ai-factory/cluster-api-provider-nico` 是最后一个。它现在是公共仓库，由 `proxy.golang.org` 提供，因此 `make work-check` 无需仓库特定凭据即可验证完整模块图。

如果任务要求你构建或测试 `controller/` 而模块无法解析，**说明情况并停止**。不要添加指向本地检出的 `replace`，也不要为了通过构建而 stub 依赖。

## 目录布局

| 路径 | 内容 | 可编辑？ |
|---|---|---|
| `agent/cmd/`、`agent/internal/`、`agent/pkg/` | 节点代理实现 | 是 |
| `cli/` | nvkectl CLI 实现和 e2e 测试夹具 | 是 |
| `auth/authn/` | NVKE API 认证库，供控制器使用 | 是 |
| `auth/authz/` | NVKE API 授权库：客户端、后端接口、隔离检查和租户 ID 规则 | 是 |
| `cluster-api-provider-provisioned/` | Provisioned Cluster API 基础设施提供者 | 是 |
| `controller/internal/` | 控制器服务器、控制器、Temporal 工作流和活动、数据库层 | 是 |
| `controller/api/openapi.yaml` | OpenAPI 3.0 规范 — **REST API 的事实来源** | 是 |
| `controller/profiles/` | 配置 profile 及其 hook 引用的 runbook | 先询问 — 正在迁出本仓库 |
| `build/` | 构建脚本（`agent.sh`） | 是 |
| `docs/` | 仓库级文档 | 是 |
| `os-image/` | 参考节点 OS 镜像规格、Packer 配方、`nvke-os-image-check` | 是 |
| `controller/api/*.gen.go` | oapi-codegen 输出 | **否 — 重新生成** |
| `cluster-api-provider-provisioned/api/v1alpha1/zz_generated.deepcopy.go` | controller-gen 输出 | **否 — 重新生成** |
| `cluster-api-provider-provisioned/config/crd/bases/*.yaml` | controller-gen 输出 | **否 — 重新生成** |
| `cluster-api-provider-provisioned/config/rbac/role.yaml` | controller-gen 输出 | **否 — 重新生成** |
| `controller/deploy/charts/controller/files/crds.yaml` | `make chart-crds` 输出（对 `management-engine` CRD 运行 kustomize） | **否 — 重新生成** |
| `agent/pkg/agent/proto/*.pb.go` | buf/protoc 输出 | **否 — 重新生成** |
| `agent/THIRD_PARTY_NOTICES.md` | go-licenses 输出 | **否 — 重新生成** |
| `controller/deploy/charts/` | Helm chart | 是。保留版本占位符；见下文。 |
| `nktp/deploy/charts/` | NKTP 遥测 Helm chart | 是，但需提升 chart 版本。阅读 `nktp/AGENTS.md` |
| `controller/deploy/dashboards/` | Grafana 参考仪表盘（JSON 导入） | 是 — 阅读该目录的 `AGENTS.md`。仅修改仪表盘时不要提升 Helm chart 版本 |

保持控制器 chart 的 `version`、`appVersion` 和 README 版本徽章为 `0.0.0`。

## 代理可以做什么

- 在 `agent/internal/`、`agent/pkg/`、`controller/internal/` 和 `os-image/` 下实现或修复逻辑。
- 通过编辑 `controller/api/openapi.yaml` 修改 REST API，**然后运行 `make generate`** 使生成的 Go 代码匹配。生成器版本固定在 `api/oapi-codegen.mod` 中；切勿直接调用 `oapi-codegen`。
- 通过编辑 `agent/pkg/agent/proto/node_agent.proto` 修改 gRPC API，**然后运行 `make protos`**，并在同一提交中提交重新生成的 `*.pb.go`。
- 通过编辑 Go 类型和标记修改 provisioned 提供者 API，**然后在 `cluster-api-provider-provisioned/` 中运行 `make generate manifests`**。
- 添加单元测试、表驱动用例和现有接口的 mock 实现。
- 改进 `docs/`、模块 `docs/` 目录和根目录 markdown 文件的文档。
- 在测试覆盖变更的前提下，在包内重构。
- 更新依赖并运行 `go mod tidy`，同时提交 `go.mod` 和 `go.sum`。

## 代理不得做什么

- **不得提交凭据、密钥、API key、token、kubeconfig、证书或 `.env` 文件。** 任何类似凭据的内容都不得出现在本仓库、测试夹具、示例清单或 profile 中。本仓库正准备公开，其完整历史也将一并公开。使用 `REDACTED` 或 `example-token` 等占位值。`DB_CONNECTION_STRING` 和 SPIFFE socket 路径由运行时环境提供——切勿内联真实值。如果发现已提交的真实密钥，**停止并报告**，而不是推送包含该密钥的修复。
- **不得手工编辑生成输出。** `controller/api/*.gen.go`、provisioned 提供者的 deepcopy、CRD 和 controller role 文件、`agent/pkg/agent/proto/*.pb.go` 和 `agent/THIRD_PARTY_NOTICES.md` 由其模块生成目标产出。修改源类型、标记、OpenAPI 规范、`.proto` 文件或 `go.mod`，然后重新生成。
- **不得批量重写许可证头。** 每个源文件都带有 `SPDX-License-Identifier: Apache-2.0`，与仓库的 LICENSE 一致。与你正在编辑的文件头保持一致；`make license` 会为新文件添加。因副作用而重新许可文件的 Pull Request 将被拒绝。
- **不得为了让变更通过而削弱 linter。** `.golangci.yml` 文件是项目标准。应修复代码。不要在同一行没有理由的情况下添加 `//nolint`。
- **不得在 `controller/` 中添加 `TODO:` 注释**——它们会导致 linter 失败。使用带跟踪链接的 `FIXME:`，这是既定约定。跟踪链接指本仓库的 GitHub Issue，而非内部跟踪器。
- **不得提交构建输出**——`bin/`、`agent/build/deb_*` 或 `controller/.local/`。这些都已列入 `.gitignore`；保持现状。根目录的 `release/` 不是构建输出：它是发布工具模块，`.gitignore` 特意为其否定了 `**/release/` 规则。
- **不得在无关变更中修改 `.github/workflows/` 中的发布或 CI 工作流。** 它们把关每次合并。
- 不得添加新依赖而不在 Pull Request 中说明原因。
- **不得在提交或 Pull Request 中添加代理署名。** 不得有署名代理的 `Co-Authored-By` trailer，PR 描述中不得有 "Generated with …" 或 "Created with …" 页脚。提交由人类贡献者创作并签署。AI 辅助的披露是 [CONTRIBUTING.md](CONTRIBUTING.md#ai-assisted-contributions) 要求的人类撰写的说明，而非代理页脚。这与 NVKR 和 AICR 仓库一致。已提交的 `.claude/settings.json` 关闭了 Claude Code 的默认署名；其他工具必须以相同方式配置。

## 编码约定

- **错误：** 用 `fmt.Errorf` 和 `%w` 包装。仅使用标准库——此处不使用 `github.com/pkg/errors`。
- **日志：** `log/slog`，结构化。不用 `fmt.Printf`，不用 `log`。
- **gRPC：** 从 RPC 方法返回带适当状态码的 `status.Errorf`。
- **可测试性：** 代理通过接口产生副作用——`CommandRunner`、`FileWriter`、`NodeStatusPersister`。使用它们以便测试替换 mock。
- **测试：** 表驱动，用例名具描述性，与代码同目录的 `*_test.go`，使用 `github.com/stretchr/testify`。
- **导入** 按标准库、第三方、本模块分组，用空行分隔。
- **换行：** 优先单行 Go 函数声明，以 150 字符为参考而非硬限制。保持一两个参数内联，除非结果行异常长；当多行参数块能显著提升可读性时使用之。换行时，每个参数独占一行。
- 导出的类型和函数带有以标识符名开头的文档注释。
- 在手写结构体和结构体初始化器中，将新字段追加在现有字段之后。保持 `ID` 和其他现有字段的当前顺序。

### 推荐与已弃用的模式

```go
// 推荐：使用 slog 进行结构化日志。
slog.Error("kubeadm init failed", "node", name, "err", err)

// 已弃用：非结构化输出。在聚合日志中无法搜索。
fmt.Printf("kubeadm init failed on %s: %v\n", name, err)
```

```go
// 推荐：用标准库包装，使 errors.Is 和 errors.As 可用。
if err := s.Runner.Run(ctx, "kubeadm", args...); err != nil {
    return fmt.Errorf("running kubeadm %s: %w", args[0], err)
}

// 已弃用：github.com/pkg/errors 不是本仓库的依赖。
return errors.Wrap(err, "running kubeadm")
```

```go
// 推荐：通过接口产生副作用，以便测试 mock。
type NodeAgentServer struct {
    Runner utils.CommandRunner
}
func (s *NodeAgentServer) reset(ctx context.Context) error {
    return s.Runner.Run(ctx, "kubeadm", "reset", "-f")
}

// 已弃用：直接调用 exec 使该方法在没有真实 kubeadm 的机器上无法测试。
func (s *NodeAgentServer) reset(ctx context.Context) error {
    return exec.CommandContext(ctx, "kubeadm", "reset", "-f").Run()
}
```

```go
// 推荐：带跟踪链接的 FIXME。TODO: 在 controller/ 中会导致 linter 失败。
// FIXME: https://github.com/dsx-ai-factory/nvke/issues/<N> — 提供者接口落地后移除。

// 已弃用：无归属、无工单的无界 TODO。
// TODO: clean this up later
```

```yaml
# 推荐：修改规范，然后运行 `make generate`。
# controller/api/openapi.yaml
    nodePoolName:
      type: string
      minLength: 1
```

```go
// 已弃用：手工编辑生成的模型以添加相同字段。
// controller/api/models.gen.go  <- 重新生成的输出，你的编辑将丢失
```

## 工具

每个模块固定自己的工具链并按需下载。使用 `make`，而非底层二进制，以保持版本固定。

Flox 是可选的，但可用时优先使用：如果 `flox` 在 `PATH` 中且你尚未进入环境（`FLOX_ENV` 未设置），以 `flox activate -- make <target>` 运行一次性命令，使工具版本与 CI 一致（低于固定版本的 `golangci-lint` 会静默漏掉 CI 报告的问题）。没有 Flox 时，普通 `make` 使用 `PATH` 中的任何内容。

| 工具 | 模块 | 用途 |
|---|---|---|
| Flox | 所有模块 | 可选的可复现工具链，固定在仓库根目录。 |
| `oapi-codegen` | `controller/` | 从 OpenAPI 规范生成 Go 服务器、客户端和模型 |
| `addlicense` | `controller/` | 许可证头（`make license`） |
| `buf` | `agent/` | Proto 格式化和生成 |
| `go-licenses` | `agent/` | `THIRD_PARTY_NOTICES.md`（`make notices`） |
| `golangci-lint` | 两者 | Lint，按各模块的 `.golangci.yml` |

## 验证 — 提交 Pull Request 前运行

在 `agent/` 中：

```bash
make fmt
make vet
make check-protos     # 如果 *.pb.go 过期则失败
make lint             # 这是 CI 关卡
make test
make notices-check    # 如果 THIRD_PARTY_NOTICES.md 有漂移则失败
```

在 `controller/` 中：

```bash
make generate         # 仅当你修改了 openapi.yaml
make api-compat-check # 仅当你修改了 openapi.yaml
make license
make fmt
make lint
make test
```

`api-compat-check` 是 `generate-check` 不具备的关卡：它将 `api/openapi.yaml` 与最新发布的 `controller/api/vX.Y.Z` 标签比较，对会破坏现有客户端的变更失败。删除字段能通过 `generate-check` 但会在此失败。破坏性变更在 `controller/api/openapi-compat-waivers.yaml` 中按变更豁免，需指定负责人——绝不通过禁用检查。参见 [