# OSS alpha 凭据操作

使用本指南选择受支持的续期或轮换流程，然后遵循其完整操作手册。**T1** 是承载 NVKE 控制器、Temporal 和 PostgreSQL 的站点管理 Kubernetes 集群。**T2** 是控制器管理的租户 Kubernetes 集群。
共享 kubeadm 布局并不意味着它们的维护可以互换：T1 重启可能中断编排 T2 维护的服务。

受支持的 alpha 配置使用 kubeadm 静态控制平面 Pod、堆叠本地 etcd、默认的 `/etc/kubernetes` 和 `/var/lib/kubelet/pki` 路径，以及现有的本地 kubeadm CA。这些名称描述的是实现要求，不是配置文件名。Fleet 配置由部署者提供，因此选择流程前请核实实际节点布局。

## 凭据与实现清单

| 凭据 | 平面与所有者 | 支持的操作与证据 |
| --- | --- | --- |
| API server、API 到 kubelet、front-proxy 客户端及 API 到 etcd 的叶子证书 | T1 操作员；T2 控制器工作流与节点代理 | T1 手动或 T2 计划/强制 kubeadm 叶子续期。比较到期时间与公开指纹；重启并验证使用方。 |
| 堆叠 etcd 的 server、peer 和 healthcheck-client 叶子证书 | T1 操作员；T2 控制器工作流与节点代理 | 在同一 kubeadm 叶子证书包中续期。继续前验证实际 etcd 成员和每个端点。 |
| `admin.conf`、`controller-manager.conf`、`scheduler.conf`；存在时的 `super-admin.conf` | T1 操作员；T2 控制器工作流与节点代理 | 随证书包续期内嵌客户端证书。`super-admin.conf` 在加入的控制平面上是可选的。导出的 admin 副本需要单独安全刷新。 |
| Kubernetes、front-proxy 和 etcd CA 证书及私钥 | T1/T2 引导与站点 PKI 所有者 | 叶子续期期间现有自签名 CA 保持不变。受管节点上观察公开 CA 到期时间；CA 轮换暂缓。 |
| Kubelet 服务证书 | T2 kubelet、控制器租户 CSR 审批器、Kubernetes 签名器 | Kubelet 请求续期；控制器在批准前验证受管节点身份；Kubernetes 签名；kubelet 必须安装结果。分别检查每个阶段。 |
| Kubelet 客户端证书 | Kubeadm/kubelet 与 Kubernetes 客户端签名器 | 在加入的 T2 节点上检查固定路径的公开到期时间。叶子证书包工作流不会续期它；需单独排查 kubelet 客户端/引导配置。 |
| Etcd API 对象数据加密密钥 | T2 操作员发起的控制器工作流与节点代理 | 现有 secretbox 密钥轮换：分发、提升、重写旧对象、移除旧密钥、完成。证明既有对象已重写且所有节点已收敛。 |
| T1 etcd 数据加密密钥 | 站点备份/恢复所有者 | 将匹配密钥与受保护的恢复备份一起保存。NVKE 不提供 T1 密钥轮换工作流。 |
| SPIFFE SVID、SPIRE CA 材料和代理传输身份 | 站点身份部署 | 由 SPIRE/传输设置拥有；不由 kubeadm 证书维护续期。参见[代理安全](https://github.com/dsx-ai-factory/nvke/blob/main/agent/docs/SECURITY.md)和[传输配置](https://github.com/dsx-ai-factory/nvke/blob/main/controller/docs/configuring-node-agent-tls-jwt.md)。 |
| 服务账号签名密钥、引导令牌、磁盘加密密钥、OIDC 签发者密钥、应用 TLS 和外部 PKI | 各自的组件/站点所有者 | 不在这些轮换流程范围内。它们的存在不意味着有 alpha 轮换 API。 |

关于确切的公开检查目录和不可用/可选行为，参见[凭据监控](../../controller/docs/credential-monitoring.md)。私钥、加密密钥和任意文件路径绝不构成指标标签。

## 选择流程

| 任务 | 操作手册 | 完成要求 |
| --- | --- | --- |
| 续期 T1 kubeadm 叶子证书 | [站点管理证书续期操作手册](../site-operations/certificate-renewal.md)和[附加证据要求](#manual-t1-evidence-and-qualification) | 一次一个控制平面节点；四个静态组件全部恢复、CA 不变、站点服务健康。 |
| 计划或强制 T2 kubeadm 续期 | [租户证书续期](../tenant-clusters/certificate-renewal.md) | 所属工作流和集群状态最终成功、节点健康、叶子证书已续期或记录为无操作、CA 不变，并在每个节点上验证。 |
| 诊断 T2 kubelet 服务证书续期 | [Kubelet 服务证书](../tenant-clusters/kubelet-serving-certificates.md) | 身份有效的 CSR 已批准、证书已签发、已安装的公开证书在节点上验证。 |
| 轮换现有 T2 etcd 加密密钥 | [Etcd 密钥轮换](../../controller/docs/etcd-encryption-key-rotation.md) | 需要 Kubernetes 1.34+ 和与控制器同一合格构建的代理；使用前参见其加载配置/进程指标前提条件。既有 Secret 已重写/可读、每个 API server 使用新主密钥、待处理状态和所有权已清除。 |
| 安装或响应生命周期告警 | [凭据监控](../../controller/docs/credential-monitoring.md) | 最新的公开观察、选定的抓取/规则、已评估的告警和已送达的通知。 |
| 记录、导出或恢复审计证据 | [凭据审计](../../controller/docs/credential-audit.md) | 已验证的手动 JSONL 或已提交的 T2 账本记录、受访问控制的导出和已声明的证据缺口。 |

一次只执行一个集群变更。T2 续期/轮换期间保持控制平面成员固定；不要与升级、回收、节点替换或其他凭据操作重叠。T1 维护前，暂停 Fleet 变更并完成或明确恢复未完成的 T2 工作。维护窗口必须包含控制台访问、恢复所有者、受保护的备份和商定的中断/错误预算。

## 拓扑与可用性

具有正常工作的 VIP/负载均衡故障转移的健康三控制平面集群是 HA 合格基线。仅三个 Ready 的 Kubernetes 节点既不能证明 etcd 法定人数，也不能证明客户端故障转移。测试客户端实际使用的端点，并检查 etcd 投票者和每个端点。T2 叶子续期拒绝不稳定的成员关系，且从不续期两控制平面集群；在此类集群的证书到期前，将其扩展到三个或更多控制平面节点。仅当 `controlPlaneConfig.desiredCount` 为 3 或更多时续期才会恢复。对于其他多节点拓扑，在宣传可用性保证前保留合格证据。

单控制平面没有冗余。其叶子续期会重启唯一的 etcd 和 API server，导致预期的 API 中断；请规划中断。Etcd 密钥轮换热重载配置，是另一种操作，但单节点站点仍无故障容忍。外部 etcd、外部 CA 签名和自定义 PKI 布局需要单独的流程。

`RenewK8sCertsV2` 是规范的 T2 叶子工作流。注册的 `RenewK8sCerts` 类型保留旧历史重放。感知回执的内嵌操作手册是 V2 选择的执行器兼容路径；它不是单独的操作员维护 API。旧版替代 T2 部署路径不被声称为合格 alpha 配置。不要通过调用旧工作流或手动编辑清单来绕过健康或所有权门禁。

## 自签名 CA 行为与暂缓能力

当配置未提供现有 PKI 时，kubeadm 引导会创建 Kubernetes、front-proxy 和堆叠 etcd CA。叶子续期使用这些当前 CA，不延长其有效期或替换其信任根。在每次叶子流程前后记录公开 CA 指纹；任何差异都是需要调查的意外变更。已续期的叶子证书无法弥补已过期或不受信任的 CA。

接近 CA 到期需要单独审查的 PKI 迁移或集群替换计划。此 alpha 功能不提供 CA 轮换、吊销、客户管理 PKI、对已过期 PKI 的通用紧急恢复，或完整的自动生命周期管理。删除本地凭据副本或续期它都不会吊销其他副本。不要基于成功的叶子续期宣传这些能力。

## 维护期间的监控与审计

在依赖到期时间前确认检查新鲜度。续期到期日是检查计划，不是到期估计。监控覆盖数据库管理的节点，即记录在控制器数据库中的节点；非受管 T1 节点需要手动操作手册的公开检查。缺失/过时的观察和失败的抓取必须保持可见，而不是被解释为健康的凭据。

[监控指南](../../controller/docs/credential-monitoring.md#configuration)是配置默认值/边界和随附告警名称的来源。仅在选定兼容 Prometheus Operator 的评估器和真实 Alertmanager 接收器时安装其可选 `PrometheusRule`；仅 NKTP 抓取本身不会送达告警。保留站点的到期前警告、已送达通知和续期后解决样本。

使用[审计指南](../../controller/docs/credential-audit.md#manual-t1-evidence)在 T1 维护前准备 `credential-audit`。在实际步骤边界记录 `check`、`approve`、`renew` 和 `verify` 观察，使用稳定的运行 ID 和不同的步骤 ID。审计工具的操作员主体是自报的；将其与站点经过认证的变更记录关联。当[轮换演示](https://github.com/dsx-ai-factory/nvke/blob/main/hack/rotation-demo/README.md)观察维护时，将确切的 `kube-system` 命名空间 UID 作为 `--cluster` 传入，将配置的节点名作为 `--node`（不是 `--node-id`），将其配置的 `manual_operation`（可选带每步后缀 `/<action>-<n>`，如 `/renew-2`）作为 `--operation` 传入，其中 `<action>` 为 `check`、`approve`、`renew`、`rotate` 或 `verify`，`<n>` 为 1 到 99 且无前导零；其他集群或操作的记录会以 `manual_audit_target_or_run_mismatch` 失败。使用 `--node-id` 或不同节点名记录的记录不计入该节点，运行会以 `per_node_audit_coverage_incomplete` 失败。T1 输出是本地 JSONL，不会自动插入 PostgreSQL。对于自动 T2 观察，使用只读数据库角色导出控制器账本，并按 `event_id` 去重。

T2 写入在变更后是有界尽力而为，因此数据库中断或崩溃可能留下缺口。直接启动 Temporal 不提供经过认证的人类操作者。审计指南记录了归属和保留限制。监控 `nvke_controller_credential_audit_writes_total` 失败，用经审查的工作流证据核对缺失事件，不要为制造审计记录而重复变更。

## 手动 T1 证据与合格性

手动续期流程是单独的[站点管理证书续期操作手册](../site-operations/certificate-renewal.md)；本节增加证据要求，不是替代的续期/重启或恢复流程。遵循该完整操作手册的逐节点顺序、备份、法定人数、健康和失败即停门禁。T1 维护可能中断审计数据库本身。

在窗口前准备审计工具。按照[手动审计契约](../../controller/docs/credential-audit.md#manual-t1-evidence)，在维护运行下将实际预检、批准、每个节点的续期和后检记录为不同的观察。检查其退出状态，仅将已验证的 JSONL 追加到受保护的证据文件。这些记录是自报的，不是自动 T1 编排或 PostgreSQL 送达。保留原始记录并声明缺口；绝不仅为制造证据而重复维护。

在**每个选定节点**上，将公开 CA 校验和及叶子到期时间/指纹添加到操作手册的预检记录中。这些命令需要 `sha256sum` 和 `openssl`；它们不读取私钥：

```bash
CA_FINGERPRINTS_BEFORE=$(sudo sha256sum /etc/kubernetes/pki/ca.crt \
  /etc/kubernetes/pki/front-proxy-ca.crt /etc/kubernetes/pki/etcd/ca.crt) &&
  printf '%s\n' "$CA_FINGERPRINTS_BEFORE" &&
  sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt \
    -noout -dates -fingerprint -sha256 ||
  { echo 'Preflight failed; stop and investigate.' >&2; false; }
```

在该节点续期和使用方验证后，与该节点保留的基线比较。如果会话丢失，恢复记录的续期前校验和；绝不用续期后基线替代：

```bash
(
  set -eu
  : "${CA_FINGERPRINTS_BEFORE:?Restore the recorded pre-renewal CA checksums for this node.}"
  CA_FINGERPRINTS_AFTER=$(sudo sha256sum /etc/kubernetes/pki/ca.crt \
    /etc/kubernetes/pki/front-proxy-ca.crt /etc/kubernetes/pki/etcd/ca.crt)
  [[ "$CA_FINGERPRINTS_AFTER" == "$CA_FINGERPRINTS_BEFORE" ]] || {
    echo 'Unexpected CA change; stop maintenance and investigate.' >&2
    exit 1
  }
  sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt \
    -noout -dates -fingerprint -sha256
)
```

要求 CA 文件校验和相同，且已续期叶子指纹改变、NotAfter 值更晚。使用 `sudo kubeadm certs check-expiration` 检查所有已续期条目，而不仅是此 API server 样本。此处的 CA 文件校验和是公开文件比较；审计工具单独记录公开 DER 指纹。两者都不延长 CA 有效期或证明可用性。

保留源/工具修订、Kubernetes 版本、布局/拓扑、UTC 时间、每节点预检/续期/重启/后检结果、不变的 CA 校验和、叶子到期时间/指纹、API 中断摘要、受保护备份引用和审计引用。明确记录失败和未执行的检查。将实时合格声明绑定到其原始证据，而不是本地语法或文档检查。

## 证据与密钥处理

使用[凭据合格性演示](https://github.com/dsx-ai-factory/nvke/blob/main/hack/rotation-demo/README.md)处理明确选定的 T2 续期/密钥轮换场景、到期告警送达和单独的 T1 操作手册观察。它在经过清理的发布证据中记录必需的失败和缺失场景；离线测试不能使站点合格。

保留经过清理的维护记录，包含源修订、控制器/代理和 Kubernetes 版本、配置/布局、集群和节点身份、拓扑、UTC 开始/结束时间、工作流/运行 ID 或手动步骤 ID、公开到期时间和指纹、健康门禁、API 流量结果、告警和审计引用。说明哪些检查失败或被跳过；工作流启动确认不是完成，离线模拟测试不是实时合格。

将私有备份产物分开保存：kubeconfig、CA 密钥、节点 PKI 归档、etcd 快照及其匹配的加密密钥应放在 Git 之外的加密、受访问控制的存储中。不要将 Secret、密钥字节、证书 PEM、kubeconfig 内容、任意命令输出或完整 Temporal 历史粘贴到 Jira/PR/演示证据中。公开 DER 证书 SHA-256 指纹足以进行证书比较。加密密钥证明必须使用轮换指南中描述的受信任验证器；绝不导出密钥字节。

在宣布发布验收前，针对选定站点验证每次运行。这些指南和离线检查提供可执行流程，不声称某环境已通过它们。使用[发布合格性记录](https://github.com/dsx-ai-factory/nvke/blob/main/docs/qualification/overview.md)保留实时结果和已知缺口。