# Agent 安全上下文

本 monorepo 的漏洞报告流程见仓库根目录的 [SECURITY.md](../../SECURITY.md)。

## 安全架构与上下文

### 用途与软件类别

NVKE agent（`nvke-agent`）是一个特权 Go gRPC 服务，以 systemd 单元形式安装在 NVIDIA Kubernetes Engine 节点上。NVKE 控制器调用它执行节点本地的 Kubernetes 生命周期与管理任务，这些任务原本需要 root shell 访问权限。

该服务：

- 通过 `kubeadm` 初始化、加入、重置和检查 Kubernetes 节点；
- 安装 NVKE 包、Debian 包、apt 仓库和 CNI 组件；
- 配置 Kubernetes API/etcd 加密并轮换加密密钥；
- 修改控制平面静态 Pod 清单和 kube-vip 状态；
- 上传文件、执行脚本、收集支持包。

已实现的 API 定义在 `pkg/agent/proto/node_agent.proto`。服务入口点和传输策略在 `cmd/agent/main.go`；特权 RPC 实现在 `internal/agent/` 下。

### 受保护资产

- 每个 NVKE 节点的 root 级控制权及其系统配置。
- Kubernetes 管理员 kubeconfig 和 kubeadm 加入凭据。
- RPC 请求中传递的 Kubernetes API 加密密钥和 LUKS 主密钥。
- SPIFFE X.509 SVID 和 SPIRE 工作负载 API 信任根。
- `/var/lib/nvke/state.json`，包括生命周期和操作状态。
- Kubernetes 静态 Pod 清单和集群 RBAC 资源。
- NVKE 包、包集合、签名密钥、索引和 OCI 凭据。
- 审计日志、脚本执行日志和支持包，可能包含敏感运维信息。

### 信任边界与数据流

1. **控制器到 agent：** NVKE 控制器连接 TCP 端口 `15005`。生产连接使用 SPIFFE mTLS。`cmd/agent/main.go` 要求客户端 SVID 与 agent 处于同一信任域，且路径以 `/nke-service/` 开头。
2. **SPIRE 到 agent：** agent 从 `--spiffe-socket-path` 配置的 SPIRE 工作负载 API socket 获取 X.509 身份和信任包。
3. **agent 到主机：** systemd 服务以 root 运行，调用 `kubeadm`、`apt-get`、`helm`、`kubectl`、`cryptsetup`、`clevis`、`crictl` 等工具，写入安全敏感的主机和 Kubernetes 配置。
4. **agent 到 Kubernetes：** 在控制平面节点上，agent 使用管理员 kubeconfig 执行集群变更。本地 API-server 健康检查目标为 `https://127.0.0.1:6443`；该仅回环端点有意跳过 TLS 验证。
5. **agent 到制品服务：** 包和收集操作从配置的 HTTPS/S3 或 OCI 源获取内容。apt 仓库使用调用方提供的签名密钥和 `signed-by` keyring。
6. **agent 到运维人员：** 一元 RPC 元数据写入 `/var/log/nvke-agent-audit.log` 并镜像到服务日志。脚本活动写入 `/var/log/nvke-execute-scripts.log`。`GetSupportBundle` 向已认证调用方返回选定状态和日志。

### 授权模型

SPIFFE mTLS 对调用方 NVKE 服务进行认证和粗粒度授权。持有授权的 `/nke-service/` 身份即可访问完整 gRPC 服务；agent 不实现按调用方或按 RPC 的角色。各 RPC 在适用处增加校验，如控制平面角色检查、文件路径和大小限制。

这种宽泛权限是有意为之，因为 NVKE 控制器是受信任的集群编排器。因此，授权控制器身份被攻破必须视为该身份可达的每个节点可能被 root 攻破。

## 威胁模型

### 1. 特权 gRPC 操作被未授权使用

攻击者若到达端口 `15005` 并获得授权的 `/nke-service/` SPIFFE 身份，即可以 root 调用 `ExecuteScript`、`StreamExecuteScript`、`UploadFileBundle`、kubeadm 操作、加密操作或包安装。

控制措施包括 `cmd/agent/main.go` 中的 SPIFFE mTLS、同信任域和路径前缀检查、网络分段、一元 RPC 审计日志以及操作特定校验。因此 SPIRE 控制平面和授权控制器身份是关键安全依赖。

### 2. 开发环境之外启用明文 gRPC

以 `--disable-mtls` 启动 `cmd/agent/main.go` 会移除传输认证和加密，同时保留完整特权 API 和 gRPC 反射。任何能到达该端口的网络主体都可控制节点。

该标志和明文 `grpcurl` 示例仅用于隔离开发环境。生产部署配置绝不能设置 `--disable-mtls`，节点网络必须限制对端口 `15005` 的访问。

### 3. 恶意脚本、归档或暂存文件

`internal/agent/agent.go` 及相关上传和执行路径有意支持远程文件放置和任意脚本执行。构造的归档路径、不安全目标、超大文件、shell 内容或授权调用方的误用可能修改主机或泄露密钥。

暂存的 kubeadm 文件要求规范绝对路径、有效的所有权和权限值以及单文件大小限制。上传和执行活动会被审计。这些检查减少意外误用，但不会对来自授权编排器的内容做沙箱隔离。

### 4. 制品或仓库供应链被攻破

`InstallBundle`、`InstallCollection`、`ConfigureAptRepo` 和 `InstallPackages` 可安装从配置的 S3、HTTPS、apt 或 OCI 位置获取的 root 执行内容。被攻破的发布者、镜像仓库、仓库签名密钥、索引、凭据或配置可能引入恶意节点软件。

生产部署必须使用 NVKE 控制的制品位置，保护 `NGC_API_KEY` 等 OCI 凭据，使用专用 `signed-by` keyring 验证 apt 元数据，并保留发布方访问控制和发布来源。开发 S3 位置和 `file://` 源不是生产信任锚。

### 5. 集群和磁盘加密密钥泄露

Kubeadm 加入令牌、管理员 kubeconfig、Kubernetes API 加密密钥和 LUKS 口令跨越 gRPC 边界。调试日志、命令行构造、错误消息、状态持久化、审计输出或支持包可能暴露这些值。

加密和 LUKS 密钥通过内存或进程标准输入传递，不得记录日志。调用方必须保护 `GetKubeconfig`、`GetKubeadmJoinString` 和 `GetSupportBundle` 的响应，脱敏日志，并立即轮换已泄露凭据。

### 6. 不安全或不一致的控制平面变更

影响 `encryption-config.yaml`、kube-apiserver 静态 Pod 清单、kube-vip 或 etcd 对象重写的中断或顺序错误操作可能降低可用性，或使数据仍可用旧密钥读取。向控制平面节点提供不同加密密钥可能导致集群数据不可用。

较新实现使用操作 ID、持久化状态、串行化变更、预写状态、就绪检查、快照、回滚和显式终态。NVKE 控制器仍负责跨节点排序，并对每个控制平面节点应用相同加密配置。

### 7. 支持制品或状态返回过多数据

`GetSupportBundle` 打包服务日志、审计日志、脚本日志、包清单、状态和调用方选定文件。状态 RPC 和支持包响应也可能泄露拓扑或生命周期细节。授权但被攻破的调用方可利用这些接口进行侦察或数据收集。

仅受信任的编排和支持身份可调用这些 RPC。运维人员必须只请求必要的附加文件，将返回的归档视为敏感数据，并在本仓库之外实施保留和访问控制。

## 关键安全假设

- 生产环境以启用 SPIFFE mTLS 的方式启动 agent，不使用 `--disable-mtls`。
- SPIRE 安全工作负载证明，保护其签名密钥和工作负载 API socket，轮换 SVID，并仅向受信任的 NVKE 服务签发 `/nke-service/` 身份。
- 网络策略、主机防火墙和站点路由将 TCP 端口 `15005` 限制在授权编排路径。
- NVKE 控制器认证其运维人员，授权请求的工作流，校验租户和节点所有权，保护敏感 RPC 响应，并安全排序多节点操作。
- 节点操作系统、systemd 单元、根文件系统、SPIRE agent 以及 agent 调用的可执行文件受信任，且防止非特权修改。
- 生产制品仓库、OCI 镜像仓库、apt 签名密钥和包索引由 NVKE 发布基础设施控制，不可被不受信任主体写入。
- Kubernetes 管理员 kubeconfig、加密配置和静态 Pod 清单具有仅 root 可写的主机权限。
- 回环端点 `127.0.0.1:6443` 不能被重定向到不受信任的服务；这是本地 API-server 客户端跳过 TLS 验证的依据。
- 审计和脚本日志由部署基础设施收集、访问控制、轮换和监控。应用不提供远程日志完整性或保留保证。
- agent 为一个 NVKE 集群拥有一个节点。它不设计用于在同一主机上强制隔离互不信任的租户。

## 部署假设与加固

- 仅因其主机管理职责需要，才以 root 运行打包的 systemd 服务；不要授予额外 Linux 能力或向无关工作负载挂载访问。
- 若 SPIRE 工作负载 API 不可用，部署应失败而非回退到明文。
- 限制并监控端口 `15005`；不要通过公共负载均衡器暴露。
- 以仅 root 可写保护 `/etc/default/nvke-agent`、SPIRE socket、`/var/lib/nvke`、`/etc/kubernetes`、apt keyring 和 `/var/log/nvke-*`。
- 使用生产控制的 HTTPS 或 OCI 包源和专用 apt 签名密钥。不要使用任意调用方控制的仓库。
- 将每个授权的 `/nke-service/` SVID 视为特权管理身份，并在平台允许范围内尽量缩小其作用域和生命周期。
- 监控对脚本执行、文件上传、kubeconfig 获取、加密、RBAC、仓库配置和包安装 RPC 的调用。
- 安全删除临时敏感制品，避免将含密钥的文件加入支持包。

## 范围之外

- Kubernetes、kubeadm、SPIRE、容器运行时、主机操作系统或第三方包中非 agent 集成导致的漏洞，仍归各自上游项目负责。
- 人类运维人员的认证和授权由 NVKE 控制器及周边 NVKE 平台实现，而非 agent。
- 跨节点编排、租户所有权决策、备份、灾难恢复、网络分段、日志保留和制品发布基础设施属于外部控制。
- 隔离开发机上的明文操作支持用于本地测试，但不是受支持的生产安全态势。

## 已接受风险

当前没有 Argus 发现被记录为已接受风险。未来的已接受风险必须在此处指明具体暴露、范围、负责人、过期或复审条件以及补偿控制，并在 `.security-triage.yaml` 中有对应条目。

## 常见误报模式

这些模式需结合所报告路径和执行上下文验证；它们不是一概抑制：

- `README.md`、脚本或测试中的 `grpcurl -plaintext` 和 `--disable-mtls` 描述本地开发。生产中的明文操作是有效漏洞。
- 文档和测试夹具中的示例 kubeadm 令牌、加密密钥、密码、证书或 kubeconfig 仅在来源确认无法认证到活动系统时才是非生产值。
- `internal/k8s/registry.go` 中的 `Insecure: true` 或跳过 TLS 验证仅限于回环 API-server 端点 `127.0.0.1:6443`。对非回环或调用方控制端点采取相同行为是有效问题。
- 用于重试抖动或非安全标识符的 `math/rand` 不是密码学用途。用于凭据、密钥、令牌或授权决策的随机数仍属安全敏感。
- `internal/agent/` 下的 root 命令执行和文件写入是核心产品能力，但围绕它们的缺失认证、输入边界校验、密钥处理或审计控制仍是有效发现。

## 报告受理门槛

报告应指明从攻击者控制输入或不受信任主体到受支持部署中安全影响的具體路径。依赖仅开发明文模式、仅示例凭据或已是 root 攻击者的报告，必须解释该条件为何能在生产中出现。

此门槛不排除 SPIFFE 授权绕过、跨信任域访问、归档或路径遍历、命令注入、密钥泄露、制品验证失败、不安全控制平面变更，或从较低信任 NVKE 身份提权。