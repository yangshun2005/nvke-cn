# agent

以 systemd 服务形式运行在每个 NVKE 节点上的 gRPC agent。编排器调用它来驱动 kubeadm 生命周期、包/CNI 安装、API 加密及相关节点本地操作——无需交互式 SSH。

| 文档 | 用途 |
|-----|---------|
| [AGENTS.md](AGENTS.md) | 构建、测试、proto、版本管理和贡献者约定 |
| [docs/DESIGN.md](docs/DESIGN.md) | 架构与 API 接口 |
| [docs/SECURITY.md](docs/SECURITY.md) | 安全上下文（报告：仓库根目录 `SECURITY.md`） |
| [pkg/agent/proto/node_agent.proto](pkg/agent/proto/node_agent.proto) | 规范 RPC 定义 |
| [config.env.example](config.env.example) | 构建/运行时产物端点配置项 |

**模块：** `github.com/dsx-ai-factory/nvke/agent`

## 快速开始

```bash
# 从 monorepo 根目录
./build/agent.sh test
./build/agent.sh build

# 或从本目录
make test
make build
```

在节点上安装发布 tarball（以 root 身份）：

1. 在主机上解压包
2. 运行 `scripts/agent-installer.sh`（或发布包中自带的安装程序）
3. 通过 `/etc/default/nvke-agent` 配置参数（见下文）

## 配置

复制 `config.env.example` → `config.env`（已 gitignore）用于本地构建/发布覆盖。在节点上，通过 `/etc/default/nvke-agent` 设置参数。

重要参数（也可通过环境变量设置；见 `AGENT_ARGS`）：

| 参数 | 说明 |
|------|--------|
| `--bundle-server-url` | 可选。InstallBundle 的包源（`https://`、`http://`、`file://` 或 `oci://`）。也可用 `NKE_BUNDLE_SERVER_URL`。未设置时优先使用 apt。 |
| `--index-file-url` | 可选，绝对索引 URL（`NKE_BUNDLE_INDEX_URL`）；否则由包服务器 + OS 推导 |
| `--cni-image-registry` | 可选，CNI helm `global.registry` 的 OCI 前缀（`NKE_CNI_IMAGE_REGISTRY`） |
| `--disable-mtls` | 明文 gRPC（**仅限开发**） |
| `--spiffe-socket-path` | SPIFFE workload API socket。除非设置 `--disable-mtls`，否则用于 gRPC 服务器的 mTLS；SPIRE agent RPC 始终使用 |
| `--secure-server-port` | gRPC 监听端口（默认 `15005`） |
| `--version` | 打印构建版本并退出 |

示例：

```bash
# 优先 apt（无需包服务器）
AGENT_ARGS="--disable-mtls"

# 可选：启用旧版 HTTP/OCI 包安装
AGENT_ARGS="--bundle-server-url https://artifacts.example.com/ --disable-mtls"
# OCI 包：
# AGENT_ARGS="--bundle-server-url oci://registry.example.com/nvke/bundles --disable-mtls"
```

没有内置的产物主机。使用包时，使用者必须将 agent 指向自己的 S3/HTTP 或 OCI 布局。索引条目使用相对于包服务器 URL 的路径（例如 `nke_bundles/Ubuntu_24.04/kubeadm_v1.30.0.tgz`）。

gRPC 反射在同一端口启用（配合 `grpcurl` 使用）。

### 日志

| 路径 | 用途 |
|------|---------|
| `/var/log/nvke-agent-audit.log` | 审计日志（`--audit-log-file`）；记录同时镜像到服务 journal |
| `/var/log/nvke-execute-scripts.log` | `ExecuteScript` / `StreamExecuteScript` 结果 |
| `journalctl -u nvke-agent` | 服务 journal |

`GetSupportBundle` 返回包含状态、journal 和相关诊断信息的 tarball。

## 核心能力

- **Kubeadm：** init、join、reset、preflight、join 字符串、kubeconfig
- **CNI：** 包就绪后执行 `InstallCNI`；新编排建议在 init 时设置 `noCniInstall: true`（`noCniInstall` 为 false 时保留旧版 init 内 CNI）
- **包 / 集合：** apt 包、apt 仓库、包、集合
- **加密：** Kubernetes API/etcd 静态加密
- **控制平面操作：** 优雅关闭 / kube-vip 恢复、apiserver manifest 补丁
- **工具：** 脚本执行、文件上传、支持包

完整 RPC 列表：`pkg/agent/proto/node_agent.proto` 和 [docs/DESIGN.md](docs/DESIGN.md)。

## 示例：创建小型集群（grpcurl）

以下示例使用**明显伪造**的 token、哈希和密钥，以免扫描器将其视为真实凭据。请替换为你环境中的值。

```bash
# 初始化第一个控制平面（TEST-NET 地址 192.0.2.10 仅用于文档）
grpcurl -plaintext -d '{
  "podNetworkCidr": "10.244.0.0/16",
  "serviceCidr": "10.96.0.0/12",
  "noCniInstall": true
}' 192.0.2.10:15005 proto.NodeAgent.ExecuteKubeadmInit

# 轮询状态 / join 字符串
grpcurl -plaintext 192.0.2.10:15005 proto.NodeAgent.GetKubeadmStatus
grpcurl -plaintext 192.0.2.10:15005 proto.NodeAgent.GetKubeadmJoinString

# 加入另一节点（占位符——非真实 kubeadm 材料）
grpcurl -plaintext -d '{
  "kubeadmJoinString": "join 192.0.2.10:6443 --token abcdef.0123456789abcdef --discovery-token-ca-cert-hash sha256:0000000000000000000000000000000000000000000000000000000000000000 --control-plane --certificate-key 0000000000000000000000000000000000000000000000000000000000000000"
}' 192.0.2.11:15005 proto.NodeAgent.ExecuteKubeadmJoin
```

在控制平面主机上安装包（`InstallPackages`）后，安装 CNI：

```bash
grpcurl -plaintext -d '{
  "cniVariant": "CNI_VARIANT_OVN",
  "podNetworkCidr": "10.244.0.0/16",
  "serviceCidr": "10.96.0.0/12"
}' 192.0.2.10:15005 proto.NodeAgent.InstallCNI
```

更多示例客户端和 manifest：[`samples/`](samples/) 和 [`sample-clients/`](sample-clients/)。

## 加密（概要）

API 加密是多步骤流程；参见 DESIGN.md 和 proto 注释。在文档或工单中张贴示例密钥时，使用明显无效的材料，例如：

```text
encryptionKey: "EXAMPLEONLY_not-a-real-secretbox-key_000000000000="
```

切勿将生产 token、证书密钥或加密密钥粘贴到 README 中。

## mTLS

生产环境应使用 SPIFFE mTLS（默认）。`--disable-mtls` 仅用于本地/开发。客户端需要匹配的 SPIFFE 身份（路径前缀 `/nke-service/`）。

## 构建说明

```bash
make build          # linux/amd64 + linux/arm64
make package        # release/ 下的发布 tarball
make protos         # 通过 buf 重新生成 *.pb.go
make setup-hooks    # 感知 monorepo 的 proto pre-commit hook
```

版本来自 `git describe --match 'agent/v*'`（组件作用域标签；见 [AGENTS.md](AGENTS.md)）。需要时用 `VERSION=...` 覆盖。

## 已发布产物

### 二进制文件（GHCR）

`main` 上的每次提交和每个 `agent/vX.Y.Z` 标签都会将二进制文件作为 OCI 产物发布到 `ghcr.io/dsx-ai-factory/nvke/agent`，并附带多架构索引（`.github/workflows/agent-artifact.yml`）。按平台拉取：

```bash
oras pull ghcr.io/dsx-ai-factory/nvke/agent:v0.3.0 --platform linux/arm64   # 发布版本
oras pull ghcr.io/dsx-ai-factory/nvke/agent:edge   --platform linux/amd64   # main 最新
```

这些是二进制文件，**不是**容器镜像——`docker pull` 会拉取一个，但没有可运行的内容。请使用 `oras`。

⚠️ **路径变更过两次。** 发布版本 `agent/v0.2.1-rc.1` 到 `agent/v0.2.1-rc.4` 在 2026-09-12 仓库迁移到 `dsx-ai-factory` 之前发布，位于 `ghcr.io/nvidia/nke/agent`。截至 `agent/v0.2.1-rc.0`（含）的发布版本位于更早的 `ghcr.io/nvidia/nvke-agent`。两者均已冻结，仍可提供。

标签为 `edge`、`sha-<commit>` 和 `vX.Y.Z`，每个标签下都有 `-amd64` / `-arm64` 单平台变体。这些是产物，不是容器镜像——层就是可执行文件本身，因此 `docker pull` 不是正确的使用方式。

### Debian 包（apt 接收）

每个 `agent/vX.Y.Z` 标签还会构建原生 amd64 和 arm64 `.deb`，并上传到 apt-publisher 的 S3 接收端（`.github/workflows/agent-deb-publish.yml`，通过 `scripts/upload-s3-apt-intake.sh`）。main 合并和分支工作流运行不会摄入——仅 `refs/tags/agent/v*`。相同的包也会作为 OCI 产物发布到 GHCR：

```bash
oras pull ghcr.io/dsx-ai-factory/nvke/agent-deb:v0.3.0 --platform linux/amd64
```

Apt 仍是节点安装路径；上面的 OCI 二进制是同一个可执行文件，但不含打包/systemd unit。参考 OS 镜像（`os-image/`）安装从此检出构建的 `.deb`。包规范和维护者脚本位于 `os-image/packages/nvke-agent/packaging/`；`make -C agent deb-amd64` 或 `deb-arm64` 使用共享的固定 nfpm 构建。

接收目标来自 GitHub `apt-intake` 环境（`APT_INTAKE_S3_URI`、`APT_INTAKE_AWS_ROLE_ARN` 等）——此处不提交任何特定于 bucket 的内容。一旦 apt-publisher 指向该接收端，即可从之前的 GitLab 包源切换。