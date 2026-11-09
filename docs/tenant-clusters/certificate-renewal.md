# 续期租户 Kubernetes 叶子证书

这是 NVKE 控制器标准 `RenewK8sCertsV2` 工作流的运维流程。
定时和强制周期使用相同的健康、执行器和所有权门禁。
[实现与失败契约](../../controller/docs/certificate-renewal.md)
描述了感知回执的回退、静态 Pod 恢复和保留锁行为。站点管理集群请使用单独的
[站点管理证书续期运行手册](../site-operations/certificate-renewal.md)。
审计和公共 CA/叶子证书对比请参阅[额外手动证据要求](../operations/credential-lifecycle.md#manual-t1-evidence-and-qualification)。

## 前置条件与目标选择

使用已配置的、由控制器管理的 T2 集群，需具备 kubeadm 静态 Pod、堆叠式本地
etcd、默认 PKI 路径以及健康且未过期的本地 CA。冻结控制平面
成员关系，并安排维护负责人、控制台访问和恢复备份。
所有控制平面节点必须已加入/Ready，其代理可达，所有 etcd
成员健康且所有本地 API 就绪。将代理升级到符合此工作流要求的修订版本；仅能连通 gRPC 端口
并不代表支持 V2 续期，对于拥有多个控制平面的集群，也不代表支持 kube-vip
排空/恢复。

HA 基线为三个控制平面，并具备可用的 VIP/负载均衡器。
续期在拥有三个或更多控制平面节点的集群上运行，或在以单个控制平面创建的集群上运行。
恰好两个控制平面的集群永远不会续期：其周期会被记录为失败，阶段为 `two-control-plane-voters`。请在证书过期前将其扩容到三个或更多控制平面节点；
仅当 `controlPlaneConfig.desiredCount` 为 3 或更多时才会恢复续期。双控制平面集群
不能缩容到一个。允许单控制平面，但预期会出现 API 中断。验证客户端的实际端点和计划中的中断/错误预算。
不要安排同时进行的升级、节点替换、加密轮换
或其他集群变更。

在运维机器上，选择站点并获取私有租户 kubeconfig：

```bash
umask 077
SITE_NAME=example-site
CWE_URL=https://controller.example.invalid
SITE_KUBECONFIG=/secure/site.kubeconfig
CLUSTER_NAME=example-tenant
CLUSTER_ID=00000000-0000-0000-0000-000000000001
TENANT_KUBECONFIG=/secure/tenant.kubeconfig
TENANT_CONTEXT=example-tenant-context
CWE_CURL_CONFIG=/secure/controller-curl.conf

nvkectl config set-site --name "$SITE_NAME" --url "$CWE_URL" \
  --kubeconfig "$SITE_KUBECONFIG"
nvkectl get kubeconfig "$CLUSTER_NAME" --output-file "$TENANT_KUBECONFIG"
kubectl --kubeconfig "$TENANT_KUBECONFIG" --context "$TENANT_CONTEXT" \
  get namespace kube-system -o jsonpath='{.metadata.uid}{"\n"}'
kubectl --kubeconfig "$TENANT_KUBECONFIG" --context "$TENANT_CONTEXT" get nodes -o wide
kubectl --kubeconfig "$TENANT_KUBECONFIG" --context "$TENANT_CONTEXT" \
  -n kube-system get pods -o wide
kubectl --kubeconfig "$TENANT_KUBECONFIG" --context "$TENANT_CONTEXT" get --raw='/readyz?verbose'
```

将每个示例替换为清单中的值，并使用所获取 kubeconfig 中存在的上下文。确认 `kube-system` UID、节点集和 API 端点
与所选租户匹配。预期输出为 Ready 节点、健康的控制平面
Pod 和成功的 `/readyz` 检查。遇到意外的身份或健康状态时停止。
`nvkectl get kubeconfig` 必须读取管理集群访问 Secret，且
运维人员必须另外具备到租户 API 的网络可达性。

将站点的认证和信任设置放入权限为 0600 的 curl
配置文件：来自[调用 API](../operate/api-authentication.md#call-the-api) 的 bearer token
作为 `Authorization` 头，并且当控制器的 TLS 端点使用
私有 CA 时，放入该 CA 包。按该页面设置 `TOKEN`；`SITE_CA_BUNDLE` 是
占位路径：

```bash
SITE_CA_BUNDLE=/secure/site-ca.crt
(umask 077 && printf 'header = "Authorization: Bearer %s"\ncacert = "%s"\n' \
  "$TOKEN" "$SITE_CA_BUNDLE" > "$CWE_CURL_CONFIG")
```

当端点证书链到公共 CA 时，省略 `cacert` 行。
token 过期时替换该文件。不要启用 curl 跟踪或打印
该配置。仅读取相关的
运维集群字段用于私有诊断，而不是导出完整的集群对象。在将状态诊断纳入
脱敏证据之前先进行审查：

```bash
set -o pipefail
curl --fail --silent --show-error --config "$CWE_CURL_CONFIG" \
  "$CWE_URL/api/clusters/$CLUSTER_ID" |
  jq '{id, name, nextCertRenewalDue, conditions: {
    provisioned: .conditions.provisioned, deleting: .conditions.deleting,
    updateInProgress: .conditions.updateInProgress,
    certificateRenewal: .conditions.certificateRenewal}}'
```

`provisioned.status` 必须为 `"True"`，删除状态必须不存在或状态为
`"False"`，且 `updateInProgress` 不存在/为 null。条件状态是
区分大小写的字符串 `"True"`、`"False"` 和 `"Unknown"`；未知的删除
状态不是非活动删除。过去的时间表示定时检查已逾期；它
并不表示证书过期。已存在已接受或失败的周期需要
在再次操作前进行检查。

在**每个**控制平面节点上，使用特权会话记录基线：

```bash
sudo kubeadm version -o short
sudo kubeadm certs check-expiration
sudo sha256sum /etc/kubernetes/pki/ca.crt \
  /etc/kubernetes/pki/front-proxy-ca.crt /etc/kubernetes/pki/etcd/ca.crt
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt \
  -noout -dates -fingerprint -sha256
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps \
  --label io.kubernetes.pod.namespace=kube-system
```

记录公开输出和全部四个组件容器 ID。使用
[节点预检命令](../site-operations/certificate-renewal.md#1-select-a-site-and-prepare-maintenance)验证本地
`admin.conf`、静态 Pod 就绪状态以及实际 etcd 成员关系/端点健康。
仅选择租户节点的堆叠 etcd 容器；不要将其与
其他 etcd 实例混淆。确认此配置所需的三个本地 CA 密钥可用。遇到缺失/过期/外部 PKI、learner、成员关系变更或
端点不健康时停止。在变更前使用站点备份流程保护 etcd 快照和相应密钥。

## 定时检查

控制器 ConfigMap 中的 `certRenewal.controllerEnabled: true`
（`controllerConfig.name`，默认 `nvke-controller-config`）启用
调度。默认值为十二小时的 `certRenewal.interval` 和三十天的
`certRenewal.thresholdDays`。部署覆盖配置时使用
[已验证配置表](../../controller/docs/credential-monitoring.md#configuration)。定时周期在 `nextCertRenewalDue`
到期时启动，并检查实际叶子证书过期时间。控制器首次看到没有到期时间的
集群时，只会设置一个间隔后的时间，因此
新集群在稳定期间不会被续期；该首个周期还会
等待每个控制平面节点就绪。成功检查在
无需续期时可能为 `unchanged`；这些节点会跳过 kube-vip 排空和续期派发。
下次到期时间仅在成功时推进。失败的定时周期仍保持
逾期：已释放的失败会在固定一小时退避后以新的
工作流 ID 重试，而保留的 `certificate_renewal` 锁会停止自动重试。

观察持久化的 `certificateRenewal.cycleID` 和该工作流的状态。
绝不要编辑到期时间来强制测试，也不要将调度器 ID 复用于手动工作。
在定时和强制运行期间保持经过认证的 API 流量运行，
并使用站点正常的延迟/错误监控。
[手动运行手册的有界 API 监控](../site-operations/certificate-renewal.md#3-renew-one-control-plane-node)
是基本回退方案；它必须使用此租户的显式 kubeconfig/context。

## 强制续期与终端观察

在控制器 worker 轮询的命名空间和任务队列中启动工作流：
控制器 ConfigMap 中的 `temporal.namespace` 和 `temporal.taskQueue`
（默认 `cloud-workflow-engine` 和 `cloud-workflow-engine-tasks`）。错误的
队列会启动一个没有 worker 轮询的工作流。传入站点 Temporal 前端的 Temporal CLI 地址和 TLS
标志；请参阅
[强制续期契约](../../controller/docs/certificate-renewal.md#forced-renewal)。
下面的地址是占位符。即使阈值检查本会是无操作，强制周期也会续期整个叶子证书
包。

```bash
TEMPORAL_ADDRESS=temporal.example.invalid:7233
TEMPORAL_NAMESPACE=cloud-workflow-engine         # temporal.namespace
TEMPORAL_TASK_QUEUE=cloud-workflow-engine-tasks  # temporal.taskQueue
RENEWAL_WORKFLOW_ID="operator-k8s-cert-renewal-${CLUSTER_ID}-$(date -u +%Y%m%dT%H%M%SZ)"

temporal workflow start --address "$TEMPORAL_ADDRESS" \
  --namespace "$TEMPORAL_NAMESPACE" --task-queue "$TEMPORAL_TASK_QUEUE" \
  --type RenewK8sCertsV2 --workflow-id "$RENEWAL_WORKFLOW_ID" \
  --id-reuse-policy RejectDuplicate --id-conflict-policy Fail \
  --input "{\"clusterID\":\"$CLUSTER_ID\",\"thresholdDays\":30,\"force\":true}"

temporal workflow describe --address "$TEMPORAL_ADDRESS" \
  --namespace "$TEMPORAL_NAMESPACE" --workflow-id "$RENEWAL_WORKFLOW_ID"
```

保留返回的工作流和运行 ID。`start` 确认提交；
它并不代表续期成功。重复 `describe` 和过滤后的 REST
读取直到终端状态，在 Temporal UI 中检查匹配的运行以查看活动
结果。成功周期报告 `succeeded`，结果为 `renewed` 或 `unchanged`，
包含完成时间戳、推进后的到期时间和已释放的变更锁。将
Temporal 失败/取消/超时以及不匹配/缺失的集群所有权视为
未解决，直到检查完成。

工作流一次处理一个控制平面节点。它在存在 kube-vip 时排空/恢复
kube-vip，并逐个重启 etcd、API server、controller-manager 和
scheduler，并在组件和节点之间进行健康检查。仅支持 V1 的
代理会被能力不同的 V2 启动 RPC 拒绝；V2 可以选择其
内嵌的感知回执兼容执行器。不要启动旧版
`RenewK8sCerts` 工作流作为手动变通方案。

## 验证实际结果

在终端成功的强制周期后，重复**每个节点**的过期时间、
CA 校验和、叶子证书指纹和 CRI 观察，以及所有租户健康检查。
要求叶子证书过期时间更晚且叶子证书指纹已更改、CA 校验和相同、
全部四个控制平面组件容器已替换、etcd 成员关系和每个端点健康且精确匹配、
本地 API/镜像 Pod 健康、所有节点 Ready，
且流量在批准预算内。无操作的定时周期应保留
叶子证书指纹并避免续期重启；明确记录该结果。

叶子证书续期不会重新签发已下载的 `nvke:breakglass` kubeconfig。
`nvkectl get kubeconfig` 返回租户访问 breakglass 客户端证书，
而不是 kubeadm 叶子证书。仅当其剩余生命周期
低于 `thresholdDays` 或集群 CA 变更时，租户访问才会重新签发它，而叶子证书续期会保留
CA，因此健康续期通常会使该 kubeconfig 保持不变；请参阅
[Kubernetes API 访问](https://github.com/dsx-ai-factory/nvke/blob/main/docs/tenant-clusters/kubernetes-api-access.md)。不要通过比较 kubeconfig
副本来判断续期。而应在每个节点上验证已续期的叶子证书：

```bash
sudo kubeadm certs check-expiration
```

每个已续期的叶子证书必须显示比基线更晚的过期时间，且 CA 行
必须保持不变。结合上述集群状态字段
（`certificateRenewal.state`、其结果和完成时间，以及推进后的
`nextCertRenewalDue`）和节点观察，这就是续期证据。
检查所获取的 kubeconfig 仍可通过所选端点工作：

```bash
kubectl --kubeconfig "$TENANT_KUBECONFIG" --context "$TENANT_CONTEXT" auth whoami
kubectl --kubeconfig "$TENANT_KUBECONFIG" --context "$TENANT_CONTEXT" get nodes
```

将下载的 kubeconfig 排除在维护证据之外，并仅通过站点的凭据流程分发。
检查最新的检查指标、任何
触发的告警以及
[T2 审计导出](../../controller/docs/credential-audit.md#export-and-retention)。

## 失败与恢复

在任何失败节点、排空/恢复或健康门禁上停止后续维护。
检查 `failedNodeID`、`phase`、`message`、原始 Temporal 运行和所选
执行器。到期时间保持不变。当执行器终止未确认时，失败周期可能保留
`updateInProgress.type=certificate_renewal`；健康的 `/readyz` 或已关闭的工作流
不能证明节点工作和清单恢复已停止。

遵循[组件恢复契约](../../controller/docs/certificate-renewal.md#status-and-failure-handling)：
确认同一操作 ID 在其所选执行器上已终止，包括
所有延迟尝试和恢复，修复节点，并保留持久回执文件。没有公开的保留锁释放 API。恢复负责人
必须在 DBA 能够仅在经过所有权检查的集群行事务中释放该失败周期的
锁之前审查终端证据。工作流 ID 和
锁类型必须仍然匹配，且 `certificateRenewal.state` 必须为
`failed`。意外的所有者意味着停止并调查。如果状态仍为
`accepted`，例如因为其工作流已终止，则停止并升级，
不要释放锁。对于定时周期，调度器会持续重新排队
已接受的周期而不派发它，因此释放锁不会
恢复续期。对于强制周期（`forced: true`），则会恢复：一旦
`nextCertRenewalDue` 已过，调度器会启动新的定时周期，Begin
会接纳它并覆盖此状态，而终止运行的节点
工作仍未确认。
不要使用通用的条件清除 SQL 命令、重放成功的续期
以获取证据，或删除回执来解除阻塞。

一旦失败所有者的工作被证明已停止、其所有权已安全释放
且所有预检门禁健康，控制器可以自动重试。
在等待该重试或启动强制工作流之前，确认
集群行的 `certificate_renewal` 预留列为空；REST API
不会显示它。被隔离的预留会使准入拒绝两者并重启
一小时退避；由旧 V1 工作流留下的预留不会阻止
准入，但第一个需要续期的节点随后会被拒绝且不进行
续期，无论预留指定哪个节点，因此周期会在那里失败。
`failedNodeID` 随后会指定该第一个节点，它可能是也可能不是
预留节点；`certificate_renewal` 列标识预留。如果它保持设置，请遵循
[组件恢复契约](../../controller/docs/certificate-renewal.md#status-and-failure-handling)的预留
段落。自动重试需要启用调度（`certRenewal.controllerEnabled`），
并且不早于失败后一小时和 `nextCertRenewalDue` 两者之后启动，
因此对于在其到期时间之前失败的强制周期，它最多可能等待
`certRenewal.interval`。该重试会获取锁，因此在此期间启动的强制工作流会被拒绝。等待该自动重试并检查其结果。
仅在其完成后且仍需要强制语义时，或如果调度已禁用，在锁
释放后，才启动一个**新的**唯一命名的强制工作流。
对于过期 CA、外部 PKI 或丢失的恢复密钥，保留备份并使用
单独审查的恢复计划。本指南不提供自动 PKI 恢复。

记录确切修订版本、配置/拓扑、UTC 时间、每个门禁和连续
流量结果、公开的前后观察、终端工作流结果、
告警/审计证据和未执行的检查。实时资格认证是单独的
验收运行；请参阅[alpha 凭据运维](../operations/credential-lifecycle.md)。