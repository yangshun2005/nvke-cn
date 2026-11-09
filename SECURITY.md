# 安全策略

NVIDIA 致力于保障软件产品与服务的安全与可信，包括本组织管理的所有源代码仓库。

## 报告漏洞

**请向 NVIDIA PSIRT 报告安全漏洞。** 不要通过 GitHub Issue、Pull Request 或 GitHub Security Advisory 报告漏洞。NVIDIA 产品安全策略要求所有外部报告统一经由 PSIRT 处理，以便在所有受影响的 NVIDIA 产品间协调披露，而不限于本仓库。

- **网页：** [安全漏洞提交表单](https://www.nvidia.com/object/submit-security-vulnerability.html)
- **邮箱：** psirt@nvidia.com — 请使用 [NVIDIA 公开 PGP 密钥](https://www.nvidia.com/en-us/security/pgp-key)加密

请尽量提供以下信息：

- 受影响的版本、标签或提交 SHA
- 受影响的组件 — `controller/`（Cloud Workflow Engine）或 `agent/`（节点代理）
- 漏洞类型：远程代码执行、权限提升、凭据泄露、认证绕过、拒绝服务等
- 复现步骤，如有概念验证请一并提供
- 影响范围，以及攻击者如何触达漏洞代码

若安全问题是误公开报告的，维护者将限制公开讨论串的讨论，并引导报告者联系 PSIRT。

## 响应与确认

PSIRT 在调查过程中与报告者协调，并适时提供进展更新。公开的协调披露与修复政策见 [NVIDIA PSIRT 政策](https://www.nvidia.com/en-us/security/psirt-policies/)。响应与解决时间取决于具体案例；本仓库不另行设定公开截止期限。

如需跟进报告，请联系 psirt@nvidia.com，不要新开公开 Issue。

## 禁运与协调披露

已确认的漏洞按禁运流程处理。PSIRT 管理时间线：修复准备期间问题保持保密，任何 NVIDIA 安全公告或 CVE 信息仅在 PSIRT 授权披露后发布。NVIDIA 是 CVE 编号机构，会为需要用户采取行动的已解决问题分配 CVE 编号。

维护者配合此流程：私下开发修复，并在 PSIRT 解除禁运前暂停公开讨论——包括 Issue、Pull Request、提交信息和发布说明。在 PSIRT 授权披露前，请对你报告的任何内容保密。

开发修复时，维护者可仅使用**未发布的草稿** GitHub Security Advisory 创建临时私有 Fork。它不是报告渠道，且不得发布。

## 支持的版本

各组件按各自的标签线发布（`agent/vX.Y.Z`、`cli/vX.Y.Z`、`controller/vX.Y.Z`、`os-image/vX.Y.Z`、`nktp/vX.Y.Z`），产品发布 `vX.Y.Z` 将它们整合在一起；首批稳定版本于 2026-09-30 发布。`controller/api` Go 模块发布 `v0.0.x`，供使用者固定版本。没有长期维护的发布分支：安全修复合入 `main`，随受影响组件的下一个版本发布，或按 [RELEASE.md](RELEASE.md) 所述从修复后的标签切出热修复版本。

| 版本 | 支持状态 |
|---|---|
| `main` | ✅ 接收安全修复 |
| 各组件线的最新发布版本 | ✅ 在其下一个版本或热修复版本中修复 |
| 旧版本、预发布版本和 Fork | ❌ 不支持 |

在项目达到 1.0 之前，策略是仅支持各组件的最近一个次版本发布。

## 报告者致谢

NVIDIA 可与 PSIRT 协调，在发布的安全公告或相关发布说明中致谢报告者。提交报告时请告知 PSIRT 你是否希望匿名，或希望使用的姓名或账号。

NVIDIA 不设漏洞赏金计划。我们确实会致谢按协调漏洞披露政策解决的外部报告问题。

## 构建产物完整性

**发布到 GHCR 的容器镜像按摘要进行认证。请验证认证，未通过认证的镜像视为未验证。**

这些镜像带有 [GitHub 构建来源认证](https://docs.github.com/actions/security-guides/using-artifact-attestations-to-establish-provenance-for-builds)：无密钥 Sigstore，基于 GitHub Actions OIDC 令牌签名，记录生成镜像的工作流、ref 和提交，并记录在公开透明日志中。本项目不持有签名密钥。

[验证已发布镜像](docs/artifact-publication.md#verifying-a-published-image)给出了验证命令。务必传入 `--repo dsx-ai-factory/nvke` —— 这才能将认证绑定到本仓库，而非任意位置的 GitHub 工作流。

已发布镜像的 NVCR 和 DSX 副本尚未认证，因为这些镜像仓库不提供 OCI referrers API。未声明任何 SLSA 等级。

第三方依赖许可证按组件记录在 [agent/THIRD_PARTY_NOTICES.md](agent/THIRD_PARTY_NOTICES.md)、[cli/THIRD_PARTY_NOTICES.md](cli/THIRD_PARTY_NOTICES.md)、[controller/THIRD_PARTY_NOTICES.md](controller/THIRD_PARTY_NOTICES.md)（同时涵盖所配置的 provider）和 [os-image/THIRD_PARTY_NOTICES.md](os-image/THIRD_PARTY_NOTICES.md) 中，由 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) 建立索引；四份文件均在每次推送时于 CI 中重新生成并验证。

## 不在范围内

以下情况通常不构成 NVKE 的漏洞：

- 上游组件的漏洞 —— Kubernetes、kubeadm、containerd、Temporal、PostgreSQL、SPIRE、Cluster API。请向所属项目报告。若 NVKE 对其中某一组件的使用使上游问题可被以原本不可行的方式利用，则**属于**范围。
- 需要已具备受管节点 root 权限或管理集群 cluster-admin 权限才能利用的发现。
- 属于运维方责任的加固缺失：RBAC、网络策略、密钥管理、磁盘加密设置、入口暴露。
- 需要物理接触硬件的拒绝服务。
- 未证明实际可利用路径的扫描器输出。
- 针对他人运营的部署的报告 —— 请联系该运营方。

如有疑问，请报告。PSIRT 宁愿排查一个范围外的问题，也不愿漏掉真实问题。

## 运维方指南

如果你部署或运营 nke：

- **密钥。** 切勿提交凭据、API 密钥、令牌或 kubeconfig。通过环境或密钥管理器提供。数据库连接字符串正是因此从环境读取。
- **节点身份。** 代理默认使用 SPIFFE/SPIRE mTLS 认证。`--disable-mtls` 仅用于本地开发 —— 切勿在他人可访问的网络上启用。
- **网络暴露。** 将控制器的 API 置于认证之后，并限制在可信网络。代理的 gRPC 端口供控制平面使用，不用于一般访问。
- **更新。** 及时应用安全版本，并阅读发布说明了解当时生效的支持版本政策。
- **依赖。** 审查 Dependabot 告警，保持固定版本为最新。
- **Pod 安全准入。** NVKE 将自己的系统命名空间标记为 `privileged` —— kube-proxy、CNI 和 node-local-dns 都需要主机访问权限，这是这些命名空间的常规做法。对你自己的命名空间应用 Pod 安全准入是你的责任。

## NVIDIA 产品安全

其他所有安全问题，请参阅 NVIDIA [产品安全门户](https://www.nvidia.com/en-us/security)。