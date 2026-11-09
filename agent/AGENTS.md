# AGENTS.md

## 项目概览

**NVKE agent** 位于 [dsx-ai-factory/nvke](https://github.com/dsx-ai-factory/nvke) 单体仓库的 `agent/` 目录。
它是一个基于 Go 的 gRPC 服务器（systemd 服务），运行在 Kubernetes 节点上，为 Kubeadm 及软件包/软件包集安装提供远程管理。

**主要用例：**
- 通过 Kubeadm 创建和管理 Kubernetes 集群（init、join、reset）
- 节点引导及软件包/软件包集安装
- Kubernetes API/etcd 密钥加密
- 通过 SPIFFE（SPIRE workload API）实现 mTLS

**模块路径：** `github.com/dsx-ai-factory/nvke/agent`

## 使用方配置

复制 `config.env.example` → `config.env`（已被 gitignore），将构建和 agent 指向**你自己的**构建产物主机。参见该文件中的 `NKE_BUNDLE_SERVER_URL`、apt 接入和容器镜像仓库配置项。运行时仍接受与 CLI 标志相同的值（`--bundle-server-url` 等）；环境变量用于填充标志默认值。

## 版本管理

Agent 发布使用**组件作用域的 git 标签**（Kubernetes/Go 多模块单体仓库风格）：

```bash
git tag -a agent/v0.1.30 -m "agent v0.1.30"
git push origin agent/v0.1.30
```

推送标签会发布：

- OCI 二进制到 GHCR / NVCR（`.github/workflows/agent-artifact*.yml`）
- `.deb` 到 apt-publisher S3 接入点，以及以
  `ghcr.io/dsx-ai-factory/nvke/agent-deb` 形式发布到 GHCR
  （`.github/workflows/agent-deb-publish.yml`）

`make` / `./build/agent.sh` 从以下命令推导 `VERSION`：

```bash
git describe --match 'agent/v*' --tags --always --dirty
```

`agent/` 前缀会被剥离，用于二进制 ldflag 和 `.deb` 版本
（`nvke-agent_0.1.30_amd64.deb`）。需要时可用 `VERSION=v0.1.30` 覆盖。

Go 模块使用方固定相同形状的标签：

```bash
go get github.com/dsx-ai-factory/nvke/agent@agent/v0.1.30
```

## 构建命令

从单体仓库根目录：

```bash
./build/agent.sh           # 构建所有架构
./build/agent.sh test
./build/agent.sh package
./build/agent.sh protos
```

或在 `agent/` 目录下：

```bash
# 构建所有架构（amd64 + arm64）
make build

# 构建指定架构
make build-amd64
make build-arm64

# 打包发布版本
make package

# 清理构建产物
make clean
```

## 测试命令

```bash
# 运行所有测试
make test

# 直接运行测试
go test ./...

# 运行指定测试
go test -v ./internal/agent -run TestExecuteKubeadmInit
```

## 代码质量

```bash
make fmt
make vet
make check-protos
make lint
make notices-check
```

`make lint` 是 CI 的 lint 关卡。`make notices-check` 校验
`THIRD_PARTY_NOTICES.md` 与随附的依赖集一致。

## Proto 生成

本项目使用 Protocol Buffers 定义 gRPC。Proto 文件位于 `pkg/*/proto/`。

```bash
# 生成 proto 文件（本地 buf）
make protos

# 格式化 proto 文件
make proto-fmt

# 校验 proto 文件是否为最新
make check-protos

# 清理生成的 proto 文件
make clean-proto
```

**重要：** 修改 `.proto` 文件后，务必运行 `make protos`，并将更新后的 `*.pb.go` 文件与 proto 变更一起提交。

## Git Hooks

```bash
# 将 pre-commit hook 安装到单体仓库的 .git/hooks（自动校验 proto）
make setup-hooks
```

pre-commit hook 会自动：
1. 校验 proto 文件是否为最新
2. 若已过期则重新生成并暂存

## 项目结构

```text
cmd/
└── agent/          # 主 nvke-agent 二进制

internal/
├── agent/          # 核心 agent 服务器实现
├── grpcserver/     # gRPC 注册 / 反射
└── k8s/            # Kubernetes 命令工具

pkg/
├── agent/proto/    # NodeAgent gRPC proto 定义
├── auth/           # gRPC 认证的令牌拦截器
├── deb/            # Debian 包辅助工具
└── utils/          # 共享工具（bundles、cmd-runner 等）

scripts/            # 随发布版本打包的安装脚本
samples/            # 示例配置和客户端
docs/               # 设计与安全背景文档
```

## 代码约定

### Go 风格
- 使用 `slog` 进行结构化日志记录（不要用 `log` 或 `fmt.Printf`）
- 错误消息应小写且不以标点结尾
- 使用表驱动测试，测试用例名称应具描述性
- 优先使用接口以便测试（参见 `CommandRunner`、`FileWriter`、`NodeStatusPersister`）

### 命名
- gRPC 方法使用 PascalCase，与 proto 定义一致
- 内部状态常量为小写字符串：`"ready"`、`"pending"`、`"error"`
- 节点角色：`"controlplane"`、`"worker"`（小写）

### 错误处理
- RPC 方法返回 gRPC 状态错误：`status.Errorf(codes.Internal, "...")`
- 使用 `fmt.Errorf` 配合 `%w` 进行错误包装
- 返回前用 `slog.Error` 记录错误

### 测试
- 使用 `github.com/stretchr/testify/assert` 进行断言
- 为单元测试创建接口的 mock 实现
- 测试文件与源文件同目录（`*_test.go`）

## 关键文件

| 文件 | 用途 |
|------|---------|
| `internal/agent/agent.go` | 主 NodeAgent gRPC 服务器实现 |
| `internal/agent/status.go` | 节点状态管理与持久化 |
| `pkg/agent/proto/node_agent.proto` | gRPC 服务定义 |
| `pkg/utils/bundles.go` | 软件包集获取与安装逻辑 |
| `pkg/utils/cmd-runner.go` | 命令执行抽象 |

## 依赖

- **Go** — 下限由根 `go.work` 中的 `go` 指令决定，而非本模块的 `go.mod`。`agent/go.mod` 中的版本更低，但所有构建都通过工作区解析。用
  `awk '/^go /{print $2; exit}' ../go.work` 读取当前值，不要轻信此处写的数字。

  在默认的 `GOTOOLCHAIN=auto` 下，较旧的本地 Go 会下载符合要求的工具链，构建照常进行，因此通常无感知。当该开关被阻止时才会出问题——`GOTOOLCHAIN=local`、显式固定旧版本，或无法联网下载——此时所有感知工作区的命令（包括 `go list`）都会失败并报 `go.work requires go >= <version>`。编辑器和语言服务器最先遇到此问题，因为它们直接调用 `go list` 而非通过 `make`，而 `make` 会为你将 `GOTOOLCHAIN` 固定为 `go.work` 中的值。
- **buf**（用于 proto 生成；参见 `make/tool-versions.mk` 中的 `BUF_VERSION`）
- **protoc** 插件，通过 buf 远程插件（`protoc-gen-go`、`protoc-gen-go-grpc`）

## 状态管理

Agent 将状态持久化到 `/var/lib/nvke/state.json`。关键状态包括：
- 节点角色与阶段（controlplane/worker、join/init/reset）
- Kubeadm 状态与 join 字符串
- 软件包集安装状态
- 磁盘加密状态

## 常见模式

### 新增 gRPC 方法

1. 在 `pkg/agent/proto/node_agent.proto` 中添加方法定义
2. 运行 `make protos`
3. 在 `internal/agent/agent.go`（或相应领域文件）中实现该方法
4. 在同目录的 `*_test.go` 中添加单元测试

### 后台操作

长时间运行的操作使用 goroutine 并更新 `NodeStatus`：
```go
go s.processKubeadmInit(args, req) // 异步处理
defer s.flushStatus(&ns) // 完成时持久化
```

### 使用 mock 测试

```go
server := &NodeAgentServer{
  Runner: &MockRunner{Output: []byte("..."), Err: nil},
  FileWriter: &MockFileWriter{},
  Persister: &MockPersister{},
  NodeStatus: &NodeStatus{...},
}
```

## gRPC 服务器反射

Agent 在 `proto.NodeAgent` 的同一端口上注册 **gRPC 服务器反射**（v1 和 v1alpha）
（参见 `internal/grpcserver.Register`）。`grpcurl` 等工具无需本地 `node_agent.proto` 副本即可列出服务并调用 RPC：

```bash
# 明文（开发）
grpcurl -plaintext localhost:15005 list
grpcurl -plaintext localhost:15005 list proto.NodeAgent

# 就绪检查（由 scripts/grpcurl-lib.sh 使用）
grpcurl -plaintext localhost:15005 list
```

编排器（NVKE controller）在调用 apt 或 InstallCNI 工作流前，通过反射探测 RPC 可用性。`GetKubeadmStatus` 中的 semver 仅用于日志记录。

## 安全说明

- 漏洞报告：参见单体仓库根目录的 `SECURITY.md`。节点 agent 架构说明：`docs/SECURITY.md`。
- 启用 mTLS（默认）时，agent 使用 workload API 的 SPIFFE 身份（`--spiffe-socket-path`）。仅在本地开发时使用 `--disable-mtls`（明文 gRPC）。
- 反射在同一受 mTLS 保护的服务器上启用；生产环境中客户端需要有效的 SPIFFE 凭据才能使用。
- 加密提供方：secretbox（XSalsa20-Poly1305）用于 Kubernetes API 密钥