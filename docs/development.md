# 开发

本页是构建、测试和检查工作的参考，供你在提交 Pull Request 前使用。`CONTRIBUTING.md` 涵盖变更流程：Issue、分支、提交格式和签署。

仓库根目录的 `make help` 按分组列出所有目标，每个模块也有自己的目标列表。此处不重复这两份列表，因为目标列表的副本会过时，而生成的那份不会。

```bash
make help                 # 根目录目标
make -C controller help   # 任意模块的目标
make show-versions        # CI 锁定的所有工具版本
```

## 获取工具

你需要 Go；在 macOS 上还需要 GNU Make 4，因为 Apple 自带的是 3.81，根 `Makefile` 拒绝在其下运行。`brew install make` 会提供 `gmake`。

其余工具都是可选的，直到某个目标需要它；需要缺失工具的目标会说明并告知获取方式。

仓库根目录还提供 [Flox](https://flox.dev/) 环境，按 CI 使用的版本锁定整套工具链，包括 GNU Make：

```bash
brew install flox        # 或参见 https://flox.dev/docs/install-flox/install
flox activate            # 在根目录或任意模块下均可
```

**Flox 是可选方案，并非必需。** 每个目标都使用你 `PATH` 中的工具运行。想省心使用锁定版本就用 Flox，想自己管理工具就跳过。

## 本地循环

```bash
make build                # 编译所有模块，二进制输出到 bin/
make test                 # 每个模块自己的测试套件
make coverage             # 同样的套件，附带每模块覆盖率表
make -C controller lint   # 只改了一个模块时，只跑该模块
```

`make test` 已为每个模块写入覆盖率文件到 `coverage/`。`make coverage` 从头重跑收集模块并报告；`make coverage-report` 跳过运行，直接基于已有文件报告。`go tool cover -html=coverage/<module>.out` 可打开单个文件。**没有阈值会阻止合并，百分比也不衡量测试好坏**——在评审中引用前请先读 [coverage-policy.md](coverage-policy.md)。

每个模块都响应同样的四个动词，因此 `make -C <module> {build,test,lint,fmt}` 对 `agent`、`cli`、`controller`、`controller/nico-sim` 和 `os-image` 都适用。

针对 NVKE controller（`controller/`），`make -C controller generate` 会从 `api/openapi.yaml` 重新生成 OpenAPI 客户端、服务端和模型。**这是最容易被遗忘的一步。** 生成的文件已提交，改了 spec 却不重新生成会留下过期文件，CI 会发现。

## 提交 Pull Request 前

```bash
make preflight
```

它会判断你改了哪些模块，然后对这些模块执行 lint 和测试；当变更涉及工作区和许可证头检查时，也会加上这些检查。运行前会打印将要执行的内容以及未覆盖的内容。

当变更涉及工作区时，preflight 会运行 `make work-check`。这不是只读操作：它会在每个模块中运行 `go work sync` 和 `go mod tidy`，可能改写 `go.mod`、`go.sum` 和 `go.work.sum`。当所需的模块或校验和数据不在本地模块缓存中时，它会访问 Go 模块代理，因此冷缓存下需要网络。提交它产生的变更，或先 stash 以保持工作树干净。参见 [go-workspace.md](go-workspace.md)。

需要时可自行限定范围：

```bash
make preflight MODULES=cli
make preflight PREFLIGHT_BASE=upstream/main
```

想跑全部时：

```bash
make build test           # 编译并测试所有模块
make work-check           # 工作区与模块清单一致
make license-header-check # 每个源文件都带 Apache-2.0 头
```

每次改动 `go.mod` 时，运行 `make work-sync` 并提交其所有变更。依赖是公开的，按需下载到 Go 模块缓存；无需访问 NVIDIA 内部仓库。

## 本地无法复现的内容

合并门禁对每个 Pull Request 运行所有成员，无论你改了什么，因为被调用的工作流会忽略自身的 `paths:` 过滤。参见[工作流指南](../.github/workflows/README.md)。因此本地运行只能缩小可能新失败的范围，不能缩小实际运行的范围。

以下是本地结果与 CI 不同的检查及原因：

| 检查 | 差异原因 |
|---|---|
| CAPD 端到端套件 | `make test-e2e-backend-ci` 除非主机是 amd64 上的 Linux，否则拒绝启动。在 Mac 上它会在到达 Docker 前停止。 |
| gofail 套件 | `make -C controller e2e-gofail` 需要 PostgreSQL、envtest API server 和 Temporal dev server。 |
| CodeQL、Scorecard、依赖审查 | 需要 GitHub 侧基础设施，没有本地形式。 |
| 版本字符串 | CI 检出是浅克隆且不带标签，因此那里的 `git describe` 返回提交，而你的完整克隆返回标签。两者都正确。 |

## 文档

`docs/` 存放较长的指南。如果工作区或模块检查失败，先读 [`go-workspace.md`](go-workspace.md)。