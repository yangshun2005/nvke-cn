# 架构概览

NVKE（NVIDIA Kubernetes Engine）在 NVIDIA 基础设施上编排 Kubernetes 集群的生命周期。该仓库是一个包含十三个模块的 Go 工作区。其中三个模块在生产环境运行：NVKE 控制器（以 Helm chart 部署）、节点代理（以 Debian 包安装在每个节点上，由 systemd 运行），以及 Cluster API 基础设施提供者（与控制器部署在同一集群中）。其余模块包括 CLI、生成的 API 模块、API 认证库、管理引擎的 CRD 类型，以及测试和开发工具。

```
              ┌──────────────────────────────────────────────┐
              │   site controller cluster                    │
              │                                              │
   client ───▶│  NVKE controller (controller/)                │
  REST/HTTP   │                                              │
              │   REST API ──▶ Postgres ──▶ outbox processor │
              │                  ▲                │          │
              │   reconcilers ───┘                │          │
              │   (controller-runtime)            ▼          │
              └───────────────────────────────┬──────────────┘
                          │                   │
                          │ gRPC, dialed by   │ starts workflows
                          │ the controller    ▼
                          │            ┌─────────────┐
                          │            │  Temporal   │
                          │            └──────┬──────┘
                          │                   │ Forge API
                          ▼                   ▼
              ┌──────────────────────┐  ┌─────────────┐
              │  nvke-agent (agent/)  │  │    Forge    │
              │  gRPC server on each │  │ (bare-metal │
              │  node, systemd unit  │  │  lifecycle) │
              └──────────────────────┘  └─────────────┘
```

## 十三个模块

| 模块 | 二进制 | 说明 |
|---|---|---|
| `controller/` | `controller` | NVKE 控制器：REST API、Temporal 工作流、Postgres、reconciler。源码位于 `cmd/worker/` |
| `agent/` | `agent` | 节点代理，运行在每个受管节点上的 gRPC 服务器 |
| `cluster-api-provider-provisioned/` | `manager` | Cluster API 基础设施提供者，管理 NVKE 在 CAPI 引导前已预配的主机 |
| `cli/` | `nvkectl` | 命令行客户端 |
| `controller/api/` | 无 | OpenAPI 规范及生成的服务端和客户端类型 |
| `controller/deploy/charts/` | 无 | 控制器和 postgresql 的 Helm chart，内嵌以便 `nvkectl` 安装其自身提交对应的 chart |
| `auth/` | 无 | NVKE API 认证库 `auth/authn`，供控制器使用 |
| `controller/nico-sim/` | `nico-sim`、`nico-simctl` | 兼容 Forge 的虚拟机和 Docker 容器模拟器，用于测试和开发 |
| `controller/test/e2e/` | 无 | 端到端测试套件 |
| `controller/hack/capd-reboot-controller/` | `capd-reboot-controller` | CAPD 重启行为的测试辅助工具 |
| `management-engine/` | 无 | 管理引擎，包括 NVKECluster 和 NodePool |
| `os-image/` | `nvke-os-image-check` | OS 镜像检查 |
| `release/` | `qualify` | 发布资格认证：认证什么、运行了什么、什么已认证。不构建任何发布产物 |

`go.work` 是权威列表。工作区如何组合以及公共依赖如何解析，参见 [go-workspace.md](go-workspace.md)。

## controller

| 组件 | 描述 |
|---|---|
| `api/` | OpenAPI 3.0 规范及 oapi-codegen 服务端和客户端 |
| `internal/server/` | HTTP 处理器、认证中间件 |
| `internal/temporal/workflows/` | 集群预配、拆除和重启工作流 |
| `internal/temporal/activities/` | 工作流调用的各个步骤 |
| `internal/temporal/outbox_processor.go` | 轮询 outbox 表并启动工作流 |
| `internal/controllers/` | controller-runtime reconciler |
| `internal/db/` | Bun ORM 及 PostgreSQL 迁移 |
| `profiles/` | OS 镜像和 Kubernetes 配置 profile，YAML 格式 |

### 集群请求如何变为工作

API 不直接调用 Temporal。它写入一行记录，由其他组件拾取。这种间接是有意为之的，也是大多数人初次阅读处理器代码时容易搞错的地方。

1. 客户端调用 `POST /api/clusters`。
2. 处理器开启一个数据库事务，通过 `internal/server/clusters.go` 中的 `CreateWorkflowOutboxEntry` 同时写入集群记录和 `workflow_outbox` 表中的一行。两者要么都成功，要么都不成功，因此已提交的集群一定有其工作排队。
3. `OutboxProcessor` 轮询该表，启动匹配的 Temporal 工作流，并将该行标记为已处理（`internal/temporal/outbox_processor.go`）。
4. 工作流创建 Cluster API 资源。配置的基础设施提供者预配或接管节点，然后控制器用 kubeadm 引导 Kubernetes。
5. 控制器通过 gRPC 拨号每个节点上的 `nvke-agent`，驱动引导的节点侧操作（`internal/agent/client.go`）。
6. 集群达到 `Ready` 状态，kubeconfig 可用。

工作流活动也通过 `CreateWorkflowOutboxEntry` 以同样方式排队后续工作，而非直接启动工作流。

### reconciler

除工作流引擎外，控制器还在 `internal/controllers/` 下运行 controller-runtime reconciler。它们涵盖节点池、控制平面和节点调度、证书续期、租户访问和密钥、CAPI 机器的 TPM 和 SPIFFE 身份，以及 Teleport 令牌。

这一点值得明确说明，因为这种组合并不常见。一个进程同时运行基于 Postgres 的 REST API、Temporal worker 和一组 Kubernetes reconciler。只看工作流或只看 reconciler，都无法完整了解什么会对变更做出反应。

### 配置

控制器在启动时从 `nvke-controller-config` 一次性读取设置。该 ConfigMap 由 Helm chart 根据其 values 渲染，并挂载到 `CONTROLLER_CONFIG_PATH` 指定的路径。`internal/controllerconfig` 负责加载和校验；文件无效时控制器拒绝启动，而非回退到默认值。文件不包含的设置（如数据库连接字符串和 API 认证文件）来自环境变量（`internal/config/config.go`）。chart 拒绝设置已被文件替代的变量的 `env` 条目；chart 中的 `files/controller-config-env.yaml` 列出了每个变量及其替代键。

`oidc` 段配置工作负载集群 API 服务器的认证方式：管理平面使用的共享提供者（`oidc.managementOIDCAuth`）以及最多五个集群级提供者（`oidc.providers`）。创建处理器将集群级列表复制到集群记录（`clusters.oidc_providers`），`kubeadm init`、后续控制平面加入以及 `enableOidc` 滚动更新均从该副本渲染 API 服务器的认证配置，绝不使用实时文件。[为工作负载集群配置 OIDC](workload-clusters/oidc.md) 描述了这两者，以及 API 服务器拒绝来自集群级提供者的身份。

## nvke-agent

代理在每个受管节点上作为 systemd 服务运行。它是一个 **gRPC 服务器**：它监听，由控制器拨号（`agent/cmd/agent/main.go`、`controller/internal/agent/client.go`）。它不主动回连，控制器也没有供其调用的节点注册端点。

职责：

- 在节点上运行 `kubeadm init` 和 `kubeadm join`，并安装 CNI
- 引导 kubelet
- 通过 SPIRE 进行身份证明，使用 TPM 或加入令牌
- 处理节点生命周期操作，如 drain、重启和拆除

`agent/` 中的 `make build` 为 Linux amd64 和 arm64 交叉编译，因为代理运行在这些平台上。

## 部署

控制器以 Helm chart（`controller/deploy/charts/controller`）部署到 *site controller* 集群——一个专用 Kubernetes 集群，管理某站点的 NVKE 租户集群。

代理没有 chart。它以 Debian 包形式发布，安装在每个节点上并由 systemd 运行。

Cluster API 基础设施提供者是第三个部署组件。它是一个 `manager` 二进制，拥有自己的容器镜像和 `cluster-api-provider-provisioned/config/` 下的 kustomize 配置，以及使其可通过 `clusterctl` 安装的 `metadata.yaml`。它拥有 `ProvisionedCluster` 和 `ProvisionedHostClaim` 类型，并 reconcile NVKE 已预配的主机，以便 Cluster API 可以引导它们。

### 机器代理连接

创建或重新应用 Machine 时，NVKE 在 `nvke.nvidia.com/agent-connection` 中发布节点持久化的传输模式：`plaintext`、`spiffe`、`spiffe-jwt`、`external-mtls` 或 `external-mtls-jwt`。主机 UUID 单独发布在 `nvke.nvidia.com/host-id` 中。节点固定的 SPIRE 证明发布在 `nvke.nvidia.com/spire-attestation-method`（`tpm`、`joinToken` 或 `dsxImds`）和 `nvke.nvidia.com/spiffe-enabled`（`true` 或 `false`）中。SPIFFE 控制器读取这些注解而非数据库。操作者提供的 Machine 元数据不能覆盖其中任何一个。

`machine-annotations` 控制器将这些注解从节点记录复制到 NVKE 发布它们之前创建的 Machine 上。它只修改这些键，且仅在 `host-id` 指向该节点的 Machine 上修改。

## 延伸阅读

- `docs/go-workspace.md` — 工作区和依赖管理工作流
- `controller/docs/authenticating-to-the-api.md` — API 认证
- `docs/design/byo-os.md` — 提案：自带 OS 镜像
- `controller/AGENTS.md` — 控制器的 AI 代理开发指南
- `agent/docs/DESIGN.md` — 节点代理架构和 API 接口
- `agent/docs/SECURITY.md` — 节点代理安全上下文
- `agent/AGENTS.md` — 节点代理的 AI 代理开发指南