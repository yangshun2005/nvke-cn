# NVKE Agent — 设计文档

> 状态：活文档 / 初稿。本文描述的是**当前实际实现**的 agent，而非理想化设计。项目经过数年有机演进；本文试图为现有代码补写一套连贯的设计叙述，以便推理、评审和有意演进。

---

## 1. 概述

**NVKE agent**（`nvke-agent`）是一个 Go gRPC 服务器，作为 systemd 服务运行在 NVKE（NVIDIA Kubernetes Engine）集群的每个节点上。它是集群编排的**节点内执行面**：NVKE 控制器（本 monorepo 中 `controller/` 下的统一编排器）通过向每个节点上的 agent 发起 gRPC 调用来驱动集群生命周期，而非 SSH 登录后临时执行命令。

一句话概括：**agent 将特权、节点本地的命令式操作（kubeadm、apt、磁盘加密、静态 Pod 操作、脚本执行）转化为经过认证、可审计、在关键处幂等的 gRPC API。**

### 主要职责

- **通过 kubeadm 管理 Kubernetes 生命周期** — init、join、reset、preflight、获取 join 字符串和 kubeconfig。
- **CNI 安装** — 将打包的 OVN / Calico chart 或 manifest 部署到集群（当前路径），另保留一条旧版 init 内 CNI 路径供旧编排器使用。
- **软件/包管理** — 安装 NVKE "bundle"（tgz）、apt 包、apt 仓库配置，以及二者的 "collection"。
- **Kubernetes API/etcd 静态加密** — 初始化、提升 provider、密钥轮换、对象重写，均跨控制平面节点协调。
- **控制平面优雅关闭 / kube-vip 撤离与恢复** — 停止本节点提供控制平面 VIP，同时不破坏集群可用性。
- **节点内省与支持** — 状态、节点信息、支持包收集、任意（经审计的）脚本执行、文件上传。

### 它*不是*什么

- 它**不是**控制器/协调器。它不 watch API server，也不在循环中收敛期望状态。它执行离散的、大多为命令式的 RPC 并报告状态。（少数操作在"重复调用安全"的意义上是"收敛/幂等"的——见 §7——但没有后台控制循环。）
- 它**不是**编排器。跨节点排序（例如"先 init 一个 CP，再 join 其余节点，然后在一个节点上安装 CNI"）属于 NVKE 控制器，而非 agent。
- 它**不**特定于云/基础设施。它面向基于 Ubuntu 的 NVKE OS 镜像，除此之外与基础设施无关。

---

## 2. 背景与演进

agent 最初只是一个"通过 gRPC 运行 kubeadm"的薄服务，随着 NVKE 需求增长而不断累积能力。这段历史在代码和 API 中可见：

- 已弃用的独立 RPC（`InstallKubeadm`、`UpgradeAgent`、`ApplyNodeProfile`、`InstallCNIPlugin`、`RewriteEncryptedObjects`、`UpdateKubernetesAPIEncryptionKey`、`GetNodeInfo`、`BootstrapNode`）已作为 GitHub monorepo 迁移的一部分**从 proto 服务中移除**。
- Bundle 安装已从定制的 tgz "bundle" 和 "node profile" 迁移到 **apt 包 + 基于 S3/OCI 的 apt 仓库**（见同级 `apt-publisher` 项目）。
- CNI 安装从 **`kubeadm init` 内**安装迁移到由 Debian 打包 chart 驱动的**独立 `InstallCNI` RPC**。当 `no_cni_install` 为 false 时，仍保留一条旧版**内部** init 内路径供旧编排使用。
- 传输默认从明文 gRPC 迁移到 **SPIFFE mTLS**。
- 较新的子系统（优雅关闭、API server manifest 修补、etcd 加密重写）采用了更规范的模式——异步操作框架、双通道结果契约、WAL 式可恢复性——这些早于旧 RPC。

设计含义：**所有新工作优先采用较新模式（§7）**。

---

## 3. 目标与非目标

### 目标

- 为特权节点操作提供**单一、经过认证、可审计的入口**，使编排无需交互式 shell 访问。
- 对预期会重试的操作（kubeadm 状态轮询、加密重载等待、按 ID 键控的异步操作）**可安全重复调用**。
- **在 agent 重启后存活**而不损坏节点状态（原子状态文件、可恢复/重启时显式报错的操作语义）。
- 在破坏性节点操作期间**保持集群可用性**（优雅关闭、apiserver manifest 修补失败时自恢复）。
- 无需向每个客户端分发 `.proto` 即可**被发现**（gRPC reflection），使编排器能探测 RPC 特性。

### 非目标

- 跨节点协调 / 集群级状态（由 NVKE 控制器负责）。
- 通用远程 shell（脚本执行存在但经审计且受限）。
- 单节点多租户——一个 agent 独占一个节点。

---

## 4. 系统上下文

                        ┌─────────────────────────────┐
                        │      NVKE controller         │
                        │        (orchestrator)        │
                        └───────────────┬──────────────┘
                                        │ gRPC (SPIFFE mTLS), per-node
             ┌──────────────────────────┼──────────────────────────┐
             ▼                          ▼                          ▼
   ┌───────────────────┐      ┌───────────────────┐      ┌───────────────────┐
   │   node (CP)       │      │   node (CP)       │      │   node (worker)   │
   │  ┌─────────────┐  │      │  ┌─────────────┐  │      │  ┌─────────────┐  │
   │  │  nvke-agent  │  │      │  │  nvke-agent  │  │      │  │  nvke-agent  │  │
   │  └──────┬──────┘  │      │  └──────┬──────┘  │      │  └──────┬──────┘  │
   │  kubeadm/apt/     │      │  kubeadm/apt/     │      │  kubeadm/apt/     │
   │  crictl/cryptsetup│      │  crictl/cryptsetup│      │  ...              │
   │  local apiserver  │      │  local apiserver  │      │                   │
   └───────────────────┘      └───────────────────┘      └───────────────────┘
             ▲
             │ X.509 SVIDs
   ┌───────────────────┐
   │  SPIRE agent      │  (workload API socket)
   └───────────────────┘

外部来源：
  - Bundle/apt 构建产物：S3（https://…s3…/nke_bundles, nke-apt-*）或 OCI（nvcr.io）
  - CNI chart/manifest + 镜像：NVKE Debian 包（nvke-cni-ovn-*, nvke-cni-calico-*）
```

- **调用方：** NVKE 控制器是主要调用方。`grpcurl` 用于开发/调试，借助 reflection 成为一等公民。
- **身份：** SPIRE 签发 X.509 SVID；agent 按信任域和 `/nke-service/` SPIFFE 路径前缀授权客户端。
- **构建产物来源：** bundle/包来自 S3 或 OCI 镜像仓库；CNI 内容来自 NVKE 发布的 Debian 包。
- **本地 apiserver：** 在控制平面节点上，agent 与*本地* apiserver（`https://127.0.0.1:6443`，跳过 TLS）通信以进行重载/就绪等待，并通过 admin kubeconfig 与集群 apiserver 通信以进行集群变更。

---

## 5. 进程模型与生命周期

入口：`cmd/agent/main.go`。

1. **解析 flag/配置** → `AgentConfig`（监听端口、bundle 源 URL、索引 URL、CNI 镜像仓库、mTLS 开关、SPIFFE socket、审计/脚本日志路径、mlx5 重扫描开关）。
2. **日志设置** — 三个 JSON `slog` logger：默认（stdout）、**审计**（`/var/log/nvke-agent-audit.log` 及 stdout 供 journald）、**脚本执行**（`/var/log/nvke-execute-scripts.log`）。
3. **`RunNodeStartup`** → `NewNodeAgentServer`：
   - 确保 `/var/lib/nvke` 和 `state.json` 存在（绝不截断已有状态）。
   - 可选解析 bundle 源 URI（HTTP/S3/`file://`/`oci://`）。未设置或无效时记录警告并继续——优先 apt。
   - 设置 bundle 源时解析 bundle 索引文件（Ubuntu 22 vs 24），除非被 `--index-file-url` 覆盖。
   - 从 `state.json` 加载持久化的 `NodeStatus`；规范化旧值（`UpdateState`）；若节点已完成 init/join，在 kubelet 配置中重新断言 `registerNode: false`（防止僵尸节点）。
   - 检测架构，运行 **mlx5 重扫描**（DPU NIC 启动），除非禁用。
   - `swapoff -a`。
4. **`startSecureServer`**（goroutine）：
   - 构建 gRPC server 选项：**审计日志一元拦截器**始终启用；**SPIFFE mTLS 凭据**除非 `--disable-mtls`。
   - `grpcserver.Register` 安装 `NodeAgent` 服务**及 gRPC reflection**（v1 + v1alpha）。
   - 监听 `:15005`（默认）；持久化 `listenStatus`。
5. **`main` 阻塞于 `ctx.Done()`**（SIGINT/SIGTERM）后退出。

agent 是**每节点单个长生命周期进程**。长操作在 goroutine 中运行，通过持久化状态报告进度，而非阻塞 RPC。

---

## 6. 组件架构

### 6.1 `NodeAgentServer`（`internal/agent`）

核心类型。它嵌入生成的 `UnimplementedNodeAgentServer` 并持有：

- **可测试性行为接缝（接口）：**
  - `Runner nvkeutils.CommandRunner` — 抽象 `exec`（`Run`、`RunSplitOutput`、stdin/stdout/stderr 选项）。测试中 mock。
  - `FileWriter nvkeutils.FileWriter` — 抽象文件写入。
  - `Persister NodeStatusPersister` — 加载/持久化 `NodeStatus`。
  - `clientRegistry kubernetesClientRegistry` — 惰性 K8s 客户端。
  - `kubeadmFileOps fileOps` — 暂存文件的所有权解析。
- **状态：** `NodeStatus *NodeStatus`，由 `mu sync.Mutex` 保护。
- **并发保护：** `encryptionMu`（串行化 encryption-config 变更）、`operationStartMu`（串行化异步操作启动）、`nodeAgentCfgMu`（串行化 `config.yaml` 写入），以及 `activeOperations`/`rewriteActive` 存活标志。
- **配置：** bundle 源、索引文件、CNI 镜像仓库。
- **测试钩子：** `RootPath`（文件操作的类 chroot 前缀）、轮询间隔和超时覆盖、`apiServerReadyMaxAttempts/PollInterval`。

RPC 面按能力域拆分到多个文件：`agent.go`（kubeadm/bundle/脚本/上传核心）、`cni.go`、`etcd_encryption_*.go`、`control_plane_graceful_shutdown.go`、`patch_apiserver_manifest.go`、`preflight.go`、`operations.go`（异步框架）、`agent_deb.go`（apt）。

### 6.2 状态与持久化（`internal/agent/status.go`）

- `NodeStatus` 是单个 JSON 文档，捕获一切：名称、角色、阶段、状态、消息、已完成阶段、join 信息、CNI 状态、磁盘加密状态、K8s API 加密状态/阶段、异步操作 map、已安装包、bundle/脚本历史等。
- `JSONFilePersister` **原子**写入：临时文件 → `fsync` → `rename`。
- **快照纪律：** 变更时获取 `mu`，通过 `cloneNodeStatus` 深拷贝，释放锁，然后持久化克隆。`flushStatus` 将快照应用到实时状态并持久化。这避免在磁盘 I/O 期间持锁，也避免 slice/map 别名 bug。
- 有界历史：脚本状态裁剪至 `ScriptStatusLimit`（3）。

`state.json` 是**跨重启的事实来源**，有意容忍旧 schema（加载时规范化）。

### 6.3 Kubernetes 客户端（`internal/k8s/registry.go`）

`ClientRegistry` 从 admin kubeconfig 惰性构建并缓存两个客户端：

- `Client()` — 类型化 clientset（集群）。
- `LocalClient()` — 固定到 `https://127.0.0.1:6443` 且禁用 TLS 验证的 clientset，使重载/就绪等待针对*本*节点的 apiserver 而非 VIP。

Kubeconfig 读取和解析失败会带上下文返回。

### 6.4 工具（`pkg/`）

- `pkg/utils` — 命令运行器、文件写入器、OS/架构检测、bundle 获取/安装（HTTP + OCI）、JSON/文本获取、索引解析。
- `pkg/deb` — dpkg 查询/清单辅助。
- `pkg/auth` — bearer `TokenAuthenticator` 一元拦截器（文件后端，每 10s 热重载）。*注意：* 生产接线认证是 `main.go` 中的 SPIFFE mTLS；token 拦截器是替代/旧版认证原语。
- `pkg/agent/proto` — 生成的 gRPC stub。

---

## 7. 关键设计模式

这些是较新代码遵循、新工作应采用的约定。

### 7.1 异步操作框架（`operations.go`）

用于长运行、节点本地操作（优雅关闭、kube-vip 恢复、etcd 对象重写）：

- **调用方提供的 `operation_id`** 是幂等键。
- `StartX(operation_id)` + `GetXStatus(operation_id)` 配对。Start 在 goroutine 中运行工作并立即返回；状态通过轮询获取。
- 语义：
  - 对相同 ID/类型的活动/终态操作重新 `Start` 是**幂等**的（返回当前状态；不重跑）。
  - **同类型**的不同非终态操作被**拒绝**（`FailedPrecondition`）——同类型同时只运行一个。
  - 持久化的**非终态操作但无存活 worker**（即 agent 在操作中途重启）被**显式拒绝**而非静默恢复。*应*恢复的操作必须编写为从持久化状态恢复。
- 进度通过 `SetStep(step)` 回调报告，持久化到 `NodeStatus.NodeAgentOperations`。

### 7.2 双通道结果契约

破坏性、自愈操作在响应体中报告**节点可能留下的每种终态**，并返回 **nil gRPC 错误**；gRPC 错误码保留给"请求无法尝试、节点未变"。

典型示例 — `PatchAPIServerManifest`：`OUTCOME_APPLIED / UNCHANGED / FAILED_RESTORED / FAILED_NOT_RESTORED`。调用方可区分"已回滚到调用前状态"与"无法确认回滚，需介入"——该区分可跨线传输。gRPC `InvalidArgument`/`FailedPrecondition` 仅在未触碰任何内容时使用。

### 7.3 收敛/幂等变更

`PatchAPIServerManifest` 读取实时静态 Pod manifest，**在内存中**应用 strategic-merge patch（键控段：volumes/mounts/env）和无键 `flag_ops`（命令行上的 SetFlag/RemoveFlag），仅在结果不同时写入（先快照）。对已应用 manifest 重跑为 `OUTCOME_UNCHANGED`。写入后针对本地 apiserver 验证就绪，失败时自动从快照恢复。

### 7.4 加密的 WAL 式可恢复性

`UpdateEncryptionConfig`（密钥轮换）在**触碰 `encryption-config.yaml` 之前写入预写状态文件**，然后阻塞直到本地 apiserver 确认新配置重载。`DeadlineExceeded` 明确**可安全重试**：WAL 让恢复的调用避免重复应用变更。这使单密钥轮换步骤容忍崩溃/超时。

### 7.5 多节点加密排序（HA 安全）

要在现有集群启用加密，先用 identity 初始化每个控制平面节点，再用 secretbox。此顺序确保新写入在每个控制平面 API server 收到密钥前均可读。

在每个控制平面节点调用 `PromoteKubernetesAPIEncryptionProviders` 将 secretbox 置首。用 `StartEtcdEncryptionAtRestObjectsRewrite` 启动一次重写。轮询 `GetEtcdEncryptionAtRestObjectsRewriteStatus` 直到操作成功。然后在每个控制平面节点调用 `RemovePlainTextSecondaryEncryptionEntry` 移除明文回退。

节点标签（`nvke.nvidia.com/k8s-api-encryption-*`）门控进度。新集群在 `kubeadm init` 时直接启用加密。

### 7.6 快速失败 RPC + 异步 worker

Kubeadm init/join **同步**校验参数（在启动 goroutine 前失败），设置 `phase=…, status=pending`，持久化，然后在 goroutine 中运行 kubeadm，完成
```