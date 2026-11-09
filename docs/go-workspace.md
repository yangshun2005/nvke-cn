# Go 工作区

本仓库包含十三个 Go 模块。根目录的 `go.work` 将它们关联在一起，因此一次 `make build` 即可覆盖整棵目录树，且统一使用同一个 Go 工具链版本。`go.work`、`go.work.sum` 和模块清单均已提交；生成的 `vendor/` 目录树不提交。所有依赖（包括 `github.com/dsx-ai-factory/cluster-api-provider-nico`）均为公开依赖。Go 在首次使用时将它们下载到模块缓存，因此冷构建需要网络访问，但不需要 NVIDIA 凭据。

| 模块 | 在 `go.work` 中 | 角色 |
|---|---|---|
| `agent` | 是 | 节点代理 |
| `auth` | 是 | API 认证库 |
| `cli` | 是 | 命令行工具 |
| `cluster-api-provider-provisioned` | 是 | provisioned-host 提供者 |
| `controller` | 是 | 云工作流引擎 |
| `controller/api` | 是 | 生成的 API 模块 |
| `controller/deploy/charts` | 是 | 内嵌 Helm chart |
| `controller/hack/capd-reboot-controller` | 是 | CAPD 测试辅助工具 |
| `controller/nico-sim` | 是 | 本地 NICo 模拟器 |
| `controller/test/e2e` | 是 | 端到端测试 |
| `management-engine` | 是 | 管理服务 |
| `os-image` | 是 | 节点镜像工具 |
| `release` | 是 | 发布版本资格工具 |

## 日常使用

```bash
make                # 列出目标
make build          # 将发布的命令构建到 bin/
make work-check     # 校验工作区和模块清单
```

`make build` 生成 `bin/agent`、`bin/capd-reboot-controller`、`bin/controller`、`bin/management-engine`、`bin/nvke-os-image-check`、`bin/nvkectl`、`bin/operator-api`、`bin/tenant-api` 和 `bin/updater`。

## 更新依赖

修改某个模块的 `go.mod` 或 `go.sum` 后，运行 `make work-sync`。它会执行 `go work sync`，在禁用工作区模式下整理每个成员模块，并重建 `go.work.sum`。将 `go.work`、`go.work.sum` 以及所有变更的 `go.mod` 和 `go.sum` 一并提交。工作区模式为每个依赖选定一个版本；sync 将这些选定版本记录到每个成员模块的清单中，使独立模块构建与工作区构建保持一致。

`make work-check` 执行同样的刷新，若受跟踪的清单发生变动则失败。失败时它可能已编辑这些文件；请审查 diff 并运行 `make work-sync` 以应用预期更新。冷缓存下它需要可访问的公共 Go 模块代理，CI 不会静默跳过新鲜度检查。

该检查还会验证 `go.work` 和 `go.work.sum` 已被跟踪、`go.work` 列出的模块与根 `Makefile` 完全一致、不存在工作区级 `replace`，且每个 `go.mod` 都属于已列出的模块。

## Go 版本

**由 `go.work` 决定。** 其 `go` 指令是本仓库构建和测试所用 Go 工具链的唯一权威声明，其他所有地方要么从中派生，要么与之校验。

| 使用方 | 获取版本的方式 |
|---|---|
| CI | `actions/setup-go` 上的 `go-version-file: go.work` |
| 根目录和 `controller/` 的 Makefile | 从 `go.work` 计算出的 `GOTOOLCHAIN` |
| `govulncheck` 门禁 | `make/tool-versions.mk` 中的 `GOVULNCHECK_GO_VERSION`，派生而来 |
| 每个成员 `go.mod` 的 `go` 指令 | 手工维护、受校验，可通过 `make go-version-sync` 重写 |
| Dockerfile 中的 `golang` 基础镜像 | 手工维护、受校验 |
| Flox 清单 | 手工维护于 `.flox/env/manifest.toml`，受校验——静态清单无法派生 |

支持的版本是**恰好一个补丁版本**，而非一个范围。每个成员模块的 `go` 指令都等于 `go.work` 的，因此模块绝不会声称用本仓库未测试的工具链构建。这比 Go 的要求更严格——指令是最低要求，更低的版本仍可编译——但此处曾累积的三个值——1.26.0、1.26.5 和 1.26.6，同处一个工作区——并非经过考量的兼容性声明，而是无人有理由注意到的漂移。

`make go-version-check` 断言整张表。它只读文件，不需要工具链，并在 **Go Build** 工作流中运行，因此覆盖 Fork 和 Dependabot 的 Pull Request。

它从 `Makefile` 的 `WORK_MODULES` 和 `PRIVATE_MODULES` 获取模块列表，而非重新发现，因此不存在第二份会漂移的列表——这正是该检查要防止的故障。受跟踪的 `go.mod` 若不在任一列表中即为错误：新模块不能被添加而不受检查。`PRIVATE_MODULES` 被列出但不受检查，因为 `go.work` 不对工作区外的模块作任何声明。请通过 `make` 运行它，由 `make` 提供这些列表；脚本拒绝裸调用而非猜测。

### 升级版本

1. 编辑 `go.work` 中的 `go` 指令。
2. `make go-version-sync`——重写每个成员 `go.mod` 的指令。
3. 更新 Dockerfile 中的 `golang` 标签和摘要，以及 `.flox/env/manifest.toml` 中的 `go` 条目。两者都无法派生，因此 `go-version-sync` 会报告它们并以非零状态退出，而非假装完成。
4. `make go-version-check` 确认，然后 `make work-sync`。

补丁升级仅此而已。**次版本**升级则额外是一个决策：它改变每个成员编译所用的语言版本，以及 `controller/api` 的使用者所需的最低 Go 版本，因此应有意识地做出，而非通过合并 Dependabot 的 Pull Request。`.github/dependabot.yml` 因此忽略 `golang` 镜像的次版本和主版本更新。

## 添加模块

1. 创建其 `go.mod`，并将其路径添加到根 `Makefile` 的 `WORK_MODULES`。
2. 若它有由根 `build` 目标构建的命令，将其添加到 `CMD_MODULES`。
3. 运行 `make work-sync` 并提交生成的工作区和模块文件。

## 故障排查

若 Go 报告缺少模块或校验和，请确认公共模块代理可访问；若依赖有变更，运行 `make work-sync`。若链接的 git 工作树无法写入 VCS 元数据，使用 `GOFLAGS=-buildvcs=false make build`。nico-sim 远程测试主机也需要访问公共模块代理。在仓库根目录，根本身不是模块：请通过 `Makefile` 构建，或指定模块路径，如 `go build ./agent/...`。