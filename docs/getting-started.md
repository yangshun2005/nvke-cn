# 入门指南

本指南带你搭建 NVKE 的本地开发环境，并验证各模块能正常构建和测试。仓库是一个包含多个模块的 Go 工作区，模块列在 `go.work` 中。各模块的用途见仓库 README。

## 前置条件

| 工具 | 版本 | 安装 |
|---|---|---|
| Go | 1.26+ | https://go.dev/dl/ |
| Docker | 24+ | https://docs.docker.com/get-docker/ |
| Flox（可选） | 最新 | https://flox.dev/ — 在整个仓库范围内固定完整工具链 |
| GNU Make | 4.0+ | macOS 自带 3.81；`brew install make` 会提供 `gmake` |

Go 版本高于 1.26 也可以。`make` 会把 `GOTOOLCHAIN` 固定为 `go.work` 声明的版本，必要时自动下载，因此构建不依赖 `PATH` 上是哪个 Go。若绕过 `make` 直接用 `go` 构建，则会依赖 `PATH` 上的 Go——而且工具链比声明版本新时，启动会因 `strcase.UnicodeVersion` 不匹配而 panic。

## 克隆仓库

```bash
git clone https://github.com/dsx-ai-factory/nvke.git
cd nvke
```

## controller

> **`controller/` 无需 NVIDIA 凭据即可构建。** 其依赖全部公开。冷构建需要
> 能访问公共 Go module proxy；在仓库根目录执行
> `make build` 会编译全部十三个模块。详见
> [AGENTS.md](../AGENTS.md)。

仓库用 [Flox](https://flox.dev/) 固定工具链，根目录下有一个环境，服务于所有模块。在仓库任意位置激活：

```bash
flox activate
```

这是可选的。不激活时，每个目标都用 `PATH` 上的工具运行；若缺少所需工具，目标会提示是哪个工具以及如何安装。

### 构建

```bash
make -C controller build          # → controller/bin/controller
```

### Lint

```bash
make -C controller lint           # golangci-lint（20 分钟超时）
make -C controller fmt            # gofumpt + goimports
```

### 测试

```bash
make -C controller test           # 带覆盖率的单元测试
```

### 代码生成

若修改了 `controller/api/openapi.yaml`，需重新生成 Go 绑定：

```bash
make -C controller generate
```

切勿手动编辑 `*.gen.go` 文件——它们会被覆盖。

### 本地运行

```bash
kind create cluster                       # NVKE controller 需要监听一个集群
make -C controller run-local              # 先启动 Postgres、Temporal，再启动 controller
```

Swagger 位于 [http://localhost:8080/api/docs/](http://localhost:8080/api/docs/)。基础设施凭据配置在 Cluster API provider 上，而非 controller 上。

[first-cluster.md](first-cluster.md#api-only) 完整介绍了这条路径，并进一步讲到租户集群。

## agent (nvke-agent)

源码位于 `agent/`。安装到节点上的二进制名为 `nvke-agent`。

```bash
cd agent
make build             # 交叉编译为 linux；在 macOS 上直接 go build
                       # 产出的二进制任何节点都无法运行
go test ./...          # 或：make test
```

NVKE controller 要与某个受管节点通信，该节点上必须已存在 `nvke-agent` 和 `spire-agent`。这两个二进制通常预置在 OS 镜像中，或通过 cloud-init 添加——controller 不会安装它们。完整的构建、proto 和测试命令见 `agent/AGENTS.md`。

## 下一步

- 阅读 `docs/architecture.md` 了解系统概览
- 阅读 `controller/AGENTS.md` 获取 AI 辅助开发技巧
- 查看 `CONTRIBUTING.md` 了解 PR 流程和代码规范