# 凭据轮换资格验证演示

`demo.sh` 针对一个明确选定的集群运行真实的资格检查。它使用 `kubectl`、Temporal CLI、NVKE 控制器 REST API、只读 PostgreSQL 和 Prometheus。它只写入公开的、白名单内的 JSON 证据。本仓库不包含实际运行。离线测试演练采集器故障和脱敏；它们不对部署进行资格验证。

支持的基线是 kubeadm 静态 Pod，使用堆叠 etcd，三个控制平面节点位于选定 VIP 之后（`kubeadm-ha`）。单控制平面（`kubeadm-single`）在续期期间预期会出现 API 中断；请为该配置声明明确的错误/延迟预算。外部 etcd、外部 PKI、自定义证书布局和 CA 轮换需要其他流程。

对于 `etcd-key`，请遵循[轮换前提条件](../../controller/docs/etcd-encryption-key-rotation.md#prerequisites)：Kubernetes **1.34+**、代理来自**与控制器相同的合格构建**，并且需要节点本地已加载配置/进程启动指标；仅计数器证明不受支持。请静默独立的文件系统/配置写入者以及其他控制器变更。离线演示测试不对这些前提条件或阶段感知的故障/重置恢复进行资格验证。

## 准备环境

使用 Python 3.10+、兼容 Kubernetes 的 `kubectl`、Temporal CLI 1.1.1 或更高版本（各通道使用 JSON [workflow describe](https://docs.temporal.io/cli/command-reference/workflow) 和 `workflow start --id-conflict-policy Fail`）、`psql` 和 OpenSSL。T1 采集还需要 SSH、非交互式特权访问、GNU `date`、`sha256sum`、`openssl`、`base64` 以及控制平面节点上的 `kubectl`。脚本从不打印命令诊断信息或历史记录。Temporal 连接/TLS 凭据使用其现有的 CLI 环境/配置文件；PostgreSQL 使用 `PGSERVICE` 和 `PGPASSFILE` 以及只读角色。将这些文件和 kubeconfig 保存在 Git 之外。

将 [config.example.json](config.example.json) 复制到运维人员拥有的目录。将每个示例值替换为已批准站点的值。`KUBECONFIG` 选择 kubeconfig；`context` 必须选择确切的租户，`api_server` 必须等于该上下文的 HTTPS 服务器（HA 流量使用 VIP）。记录实际的 `kube-system` 命名空间 UID、控制器的集群 UUID/名称以及完整的已加入节点 UUID/名称/角色清单。持续 API 流量固定到 `api_server`，并且必须持续观测到该 `kube-system` UID；任何目标漂移都会使运行以 `traffic_target_mismatch` 失败，与 API 中断预算无关。采集器在开始工作前会拒绝不匹配、新增/缺失节点、不支持的控制平面数量以及不健康的端点。

将 `deployment_source_revision` 设置为用于构建已部署候选版本的干净检出 HEAD。将 `expected_versions.controller` 设置为 `/api/version.version`，`agents` 设置为每个节点 UUID 的预期已部署代理版本，`kubernetes` 设置为每个节点的预期 `kubeletVersion`。采集器比较前后报告的版本并存储其 SHA-256 哈希，从不存储任意版本字符串。代理值来自控制器持久化的注册信息；请刷新/验证注册信息，并单独保留镜像/二进制摘要。版本报告和运维人员声明的源映射不能以密码学方式证明二进制的来源。不要通过将意外的报告版本复制到预期映射中来对不相关的部署进行资格验证。

对于经过身份验证的 HTTP 服务，可选的 `cwe_authorization_file` 和 `prometheus_authorization_file` 指定包含完整 Authorization 头值的私有文件。采集器在每次请求时重新读取每个文件。在控制器身份验证强制执行的情况下，`cwe_authorization_file` 必须包含 `Bearer <token>`，其中令牌是为控制器的受众签发的（参见[向 API 进行身份验证](../../controller/docs/authenticating-to-the-api.md)）。`controller/hack/nvke-token.sh` 打印裸令牌，因此请自行添加 `Bearer ` 前缀。令牌必须在通道结束前保持有效：要么使用覆盖 `timeout_seconds` 的 `NKE_TOKEN_TTL` 签发，要么在过期前就地重写文件。一旦通道启动，无效或过期的凭据仅报告为 `service_unavailable`，并且已发生变更的续期通道无法在该集群上重复。URL/路径/头永远不会进入证据。`cwe_url` 和 `prometheus_url` 必须使用 `https://`，采集器拒绝 HTTP 重定向，因此头不会离开到另一个主机或协议。TLS 使用正常的系统信任；不提供不安全绕过。`kubelet_ca_file` 是用于验证每个配置的 `kubelet_address:10250` 的公共 Kubernetes CA。允许直接访问所有 API 服务器和 kubelet 服务端点。

首先部署检查、监控和审计变更。应用迁移，启用适当的 ServiceMonitor/PrometheusRule 选择并建立 Alertmanager 路由。所有节点都需要新鲜的完整公开观测数据。每个受管 T2 节点已安装的 kubelet 服务证书必须是 kubelet 在其配置地址上实际提供的证书，并使用 `kubelet_ca_file` 进行身份验证。匹配的已批准且已签发的 CSR 仅在仍被列出时记录为可选证据：kube-controller-manager 在大约一小时后删除已批准和已签发的 CSR，并且这些工作流都不会创建服务 CSR。未提供的证书、未知证书、过期的检查、缺失的序列或缺失的审计操作都会使运行失败。采集器不修复这些状态或批准 CSR。参见 [CSR 故障排除](../../docs/tenant-clusters/kubelet-serving-certificates.md) 和[监控](../../controller/docs/credential-monitoring.md)。

预留变更窗口、备份和支持的恢复访问。不要重叠续期、密钥轮换、升级或回收。通道运行期间不要重启或重新部署控制器。选择足够长的超时以覆盖下一次检查（默认一小时）、调度器检查和 etcd 重写（最多六小时）。`timeout_seconds` 允许最多八小时；观测超时不会终止/重置工作流或释放其变更锁。

## 运行 T2 场景

在一个集群上按以下顺序运行 T2 通道：先 `scheduled-renewal`，然后 `alert-renewal` 和 `forced-renewal`，`etcd-key` 与它们独立。强制续期和告警续期都会强制续期每个控制平面，因此先运行其中任何一个都会使计划周期变为 `unchanged`（这会使该通道失败）并清除告警通道所需的警告窗口；`aggregate` 要求来自同一集群的所有 T2 报告，因此在其证书再次接近过期之前无法进行资格验证。告警通道还需要提前准备错开的过期时间：`alert_node` 上一个仍然有效的控制平面叶子证书已处于警告窗口内，并且其 `NVKECertificateExpiring` 告警正在触发。`alert_node` 上的每个叶子证书必须在告警通道启动前保持在 `warningDays` 内但在 `certRenewal.thresholdDays` 之外（默认情况下剩余 30–45 天）；否则计划通道或后台调度器会续期它，告警会解决，`alert-renewal` 会以 `expiry_alert_not_firing` 失败。使用另一个控制平面节点，其叶子证书在 `thresholdDays` 内，作为计划通道的候选。

从干净的候选检出运行。输出路径必须是新的且在仓库之外，以便后续运行保持干净。提供给 `--approve-cluster` 的确切集群 UUID 是运维人员的确认。省略它会在创建夹具或启动工作流之前执行基线检查并失败。第一个通道是 `scheduled-renewal`；在下面描述的时间启动它：

```bash
hack/rotation-demo/demo.sh run --config /operator/private/t2.json \
  --scenario scheduled-renewal \
  --approve-cluster 00000000-0000-0000-0000-000000000001 \
  --output /operator/private/scheduled-renewal.json
```

然后使用 `--scenario alert-renewal` 运行相同的命令，最后使用 `--scenario forced-renewal`，每个都使用新的输出路径。

`forced-renewal` 使用新的运维人员工作流 ID、正阈值和 `force:true` 启动 `RenewK8sCertsV2`。它要求 Temporal 执行完成且匹配成功的集群所有者、已清除的变更/故障状态、每个控制平面上叶子指纹已更改且过期时间更晚、CA 指纹未更改、静态 Pod 容器已替换、etcd 成员关系已修复且每个端点健康、所有节点 Ready、新鲜的过期/CSR 遥测数据、已安装和已提供的 kubelet 证书匹配，以及每个节点的续期/验证审计事件。公开指纹和过期时间来自持久化的只读检查，包括终端工作流完成后的新鲜检查。

基线和最终遥测检查在原始 `timeout_seconds` 截止时间内等待与独立公开证书快照匹配的过期序列。最终检查要求终端完成后有源样本和成功的证书/CSR 观测。值和 `timestamp(...)` 源样本时间来自一次 PromQL 评估；仅即时查询的响应时间不能确定新的抓取。采集器将自抓取以来经过的时间添加到采样年龄指标中，将 CSR 观测限制保持在 300 秒。每个返回的控制器副本都必须满足这些检查。保持运维人员、控制器和 Prometheus 时钟同步以进行这些挂钟时间比较。

`scheduled-renewal` 观测实际的控制器调度器。仅当集群的 `nextCertRenewalDue` 加上续期本身和下一次证书检查（默认一小时）适合 `timeout_seconds` 内时启动它，`timeout_seconds` 最多八小时，而 `certRenewal.interval` 默认为十二。首先从 `GET /api/clusters/{clusterID}` 读取 `nextCertRenewalDue`。还要足够早地启动它，以便基线检查（包括新鲜指标等待）在 `nextCertRenewalDue` 之前完成：调度器在通道启动其流量之前接受的周期会使通道以 `cycle_started_before_traffic` 失败，并且该周期已经续期了到期节点。它等待一个新的非强制、调度器命名的周期；它不会通过直接启动 Temporal 来合成调度。在配置的选择阈值内准备一个健康的、自然接近过期的候选。集群必须已经有一个 `nextCertRenewalDue`：控制器在首次看到集群后一个续期间隔设置它，在此之前不调度任何内容。普通的计划空操作无法证明续期，会使此场景失败。控制平面可能有错开的过期时间：计划通道读取每个节点的周期范围续期审计结果，并要求已续期节点上叶子指纹已更改、过期时间更晚且静态 Pod 容器已替换。具有 `renew/no_op` 和原因 `unchanged` 的节点必须保留其健康的叶子证书和容器，并具有成功的飞行后验证审计覆盖。缺失或冲突的每节点续期结果会失败；至少一个节点必须独立证明续期。每个执行器活动的最终审计尝试决定其结果：较早的失败尝试后跟成功的重试是有效的，并保留在审计证据中可见。尝试编号不排序不同的活动或工作流运行，其终端结果必须一致。所有 CA 指纹、etcd 成员关系、目标、流量和遥测检查仍适用于每个节点。添加的每节点证据仅记录节点 UUID 和固定的 `renewed`/`unchanged` 结果。工作流历史和运行手册输出保持私有。在强制续期之前运行此通道，强制续期会移动下一个到期时间并重置叶子过期时间。命名约定是关联，不是对启动工作流的人的身份验证证明。

`etcd-key` 仅对已加密的集群启动 `RotateEtcdEncryptionKey`。它在轮换前创建一个公开标记 Secret，检查加密的 secretbox 存储，通过每个 API 服务器持续读取其未更改的载荷，然后在完成后要求更改的存储头 SHA-256 和通过每个 API 服务器匹配的新写入探针。原始存储值保留在内存和受保护的命令捕获中，永远不会进入证据。它还要求未更改的证书/容器身份以及所有轮换阶段/检查和终端轮换审计观测。成功运行在观测停止后移除其公开夹具；失败的运行可能留下夹具以供诊断。在移除任何内容或启动另一个操作之前，请遵循[阶段感知恢复](../../controller/docs/etcd-encryption-key-rotation.md#phase-aware-recovery)。

每个场景在工作期间持续读取 API，并检查声明的错误比例和最大观测延迟。密钥轮换还会在每次迭代中从每个 API 服务器读取预先存在的 Secret。这些是有界串行读取探针，不是负载基准测试，也不保证防止更短的未采样中断。任何 Secret 读取或载荷失败都是致命的，无论 API 中断预算如何。观察者在完成前等待其有界的进行中命令批次；它不会在失败时删除预先存在的标记。证据包括样本/错误计数和最大请求批次延迟；健康快照独立检查每个成员的实际 etcd 成员、非学习者状态和所有集群端点。

## 证明过期告警的投递和解决

`alert-renewal` 是一个单独的强制续期通道。准备一个处于警告窗口内的仍然有效的叶子证书，按照[运行 T2 场景](#run-the-t2-scenarios)中的描述保持 `alert_node` 上的每个叶子证书在 `certRenewal.thresholdDays` 之外，等待 `NVKECertificateExpiring` 触发，并在运行前建立真实的接收器。永远不要仅仅为了制造此测试而在生产站点上缩短/替换证书。设置 `alert_node`、`alert_credential`、`alert_receipts_file` 和 `alert_receipts_since`（在过去一天内且在运行开始之前）。CA 告警不能使用此通道，因为续期不会轮换 CA。

捆绑的接收器是**必需的**：采集器只接受其 `nke.alert-receipt.v1` 记录，而不是站点接收器的记录。在告警触发前启动它并将 Alertmanager 路由到它，以便通道启动时匹配的触发回执已在 `alert_receipts_file` 中；否则采集器拒绝启动续期（`firing_notification_not_received`）。它在 `/alerts` 接受实际的 Alertmanager webhook 请求：

```bash
python3 hack/rotation-demo/alert-receiver.py \
  --bind 127.0.0.1 --port 9087 --output /operator/private/alert-receipts.jsonl
```

通过站点批准的可达转发/代理将评估器的 Alertmanager 路由到此运维人员控制的地址。将监听器置于现有访问控制之后并启用 `send_resolved: true`。默认回环绑定需要批准的转发；远程 Alertmanager 无法直接访问它。此辅助程序仅存储固定的告警状态、回执 ID、回执时间、凭据名称和目标标签的哈希。它丢弃注释、任意标签、URL 和消息体。保护接收器和回执文件免受伪造请求；它本身不验证任何发送者。回执证明在此配置的 webhook 处的观测，而不是投递到不相关的电子邮件/寻呼接收器。

该场景要求已加载的健康规则、在实际过期前触发、已续期的公开凭据、已解决的评估器状态，以及具有相同告警 ID 匹配目标/窗口的触发/解决 webhook 回执。触发回执必须在变更之前，解决回执必须在此次运行的续期开始之后，并且在该解决和完成之间没有新的触发回执；旧的解决不能满足检查。续期一次一个节点进行，因此告警可能在最后一个节点完成前解决。缺失投递或解决会失败。告警夹具测试和缺失序列不能替代这些检查。

## 单独观测手动 T1

使用[单独的 T1 运行手册](../../docs/site-operations/certificate-renewal.md)，以及[额外的证据指南](../../docs/operations/credential-lifecycle.md#manual-t1-evidence-and-qualification)。演示不复制该流程或自动化其变更。准备单独的 T1 配置，包含 `cluster_id`（也是 `--approve-cluster` 值和完成标记的一部分）、`api_server`、`profile`、站点 `context`/`kube_system_uid`、节点的稳定 UUID 证据身份、完整节点清单、预期 Kubernetes 版本、控制平面的 SSH 目标、唯一的 `manual_operation`、不存在的 `manual_completion_file`、`manual_audit_file` 和 `timeout_seconds`。为整个 T1 运行手册及其后检查设置 `timeout_seconds`，最多 28800（八小时）：相同的截止时间限制等待完成标记，7200 秒的默认值会使更长的维护在发生后以 `manual_completion_not_observed` 失败。将 `deployment_source_revision` 设置为此演示修订版；手动 T1 没有控制器/代理注册要求。T1 清单有意覆盖 kubeadm 叶子证书/配置和 CA，并排除 kubelet 证书/CSR。

使用该配置和明确批准启动 `--scenario manual-t1`。在开始运行手册之前等待固定消息 **"T1 baseline recorded and API observation running"**。只读 SSH 采集器在 `sudo` 下运行 [public-certificates.sh](public-certificates.sh)：固定的 OpenSSL 公开证书查询，以及对于每个 kubeadm kubeconfig，执行 `kubectl config view --raw` 读取，其嵌入的客户端证书使用 `base64` 解码以计算其公开指纹和过期时间。它从不将 PKI 或 kubeconfig 复制到节点外。T1 使用实际的公开快照而不是 T2 数据库清单。

通过现有的[手动审计工具](../../controller/docs/credential-audit.md)为每个控制平面节点记录检查/续期/验证观测，使用确切的 `kube-system` UID 作为 `--cluster`，节点名称作为 `--node`，配置的 `manual_operation` 作为 `--operation`，并带有每步后缀 `/<action>-<n>`（例如 `example-operator-change-window/renew-2`），其中 `<action>` 是 `check`、`approve`、`renew`、`rotate` 或 `verify`，`<n>` 是 1 到 99 且无前导零，每当一个步骤以其他方式相同的标志重复时。采集器拒绝其他操作值并报 `manual_audit_target_or_run_mismatch`，