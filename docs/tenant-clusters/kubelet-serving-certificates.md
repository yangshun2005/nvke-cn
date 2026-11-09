# Kubelet 服务证书续期与故障排查

NVKE kubeadm 配置中的 T2 kubelet 使用 `serverTLSBootstrap: true`，通过 `kubernetes.io/kubelet-serving` CSR 申请服务证书。NVKE 控制器的租户 CSR 批准器只批准与其托管节点身份匹配的请求；Kubernetes 签名器签发证书，kubelet 安装证书。此流程独立于 `RenewK8sCertsV2`（后者续期控制平面 kubeadm 叶子证书）。

使用默认的 `/var/lib/kubelet/pki/kubelet-server-current.pem` 布局，并保持 kubelet 到 API 的认证、tenant-manager 访问、批准 RBAC 和 Kubernetes 签名器配置正常工作。自定义证书位置、自签名 kubelet 服务配置和未托管的 T1 节点不属于本流程范围。切勿仅为让请求出现而删除生产服务证书；刻意的提前轮换测试应在选定的资格环境中进行，并具备节点恢复访问能力。

## 观察真实请求

在操作机上，从清单中选择租户和预期节点：

```bash
TENANT_KUBECONFIG=/secure/tenant.kubeconfig
TENANT_CONTEXT=example-tenant-context
NODE=example-worker-1
kubectl --kubeconfig "$TENANT_KUBECONFIG" --context "$TENANT_CONTEXT" \
  get node "$NODE" -o wide
kubectl --kubeconfig "$TENANT_KUBECONFIG" --context "$TENANT_CONTEXT" get csr \
  -o custom-columns='NAME:.metadata.name,SIGNER:.spec.signerName,REQUESTOR:.spec.username,CREATED:.metadata.creationTimestamp,CONDITIONS:.status.conditions[*].type'
```

确认节点名称和 InternalIP 与控制器期望状态一致，而不仅是 Kubernetes Node 对象。选择该节点新观察到的服务 CSR，然后检查有限的公开字段及其签发后的签名证书：

```bash
CSR=replace-with-selected-csr-name
kubectl --kubeconfig "$TENANT_KUBECONFIG" --context "$TENANT_CONTEXT" get csr "$CSR" \
  -o jsonpath='{.spec.signerName}{"\n"}{.spec.username}{"\n"}{.spec.groups}{"\n"}{.spec.usages}{"\n"}{range .status.conditions[*]}{.type}{" "}{.status}{" "}{.reason}{"\n"}{end}'
set -o pipefail
kubectl --kubeconfig "$TENANT_KUBECONFIG" --context "$TENANT_CONTEXT" get csr "$CSR" \
  -o jsonpath='{.spec.request}' | base64 --decode |
  openssl req -noout -verify -subject -text
kubectl --kubeconfig "$TENANT_KUBECONFIG" --context "$TENANT_CONTEXT" get csr "$CSR" \
  -o jsonpath='{.status.certificate}' | base64 --decode |
  openssl x509 -noout -dates -fingerprint -sha256
```

这些示例使用 GNU `base64 --decode`；在 macOS 上使用 `base64 -D`。请求主题/SAN 是公开身份数据；仅保留在私有诊断记录中。`status.certificate` 为空时，最后一条命令在签发前会失败；这属于未完成阶段，而非安装成功。

控制器的身份允许列表要求以下全部条件：

- 请求者为 `system:node:<node>` 且属于 `system:nodes`。
- 节点名称存在于控制器期望状态中，且其记录的 IP 匹配。
- 一个有效、已签名的 PEM 证书请求，主题 CN 为 `system:node:<node>`，组织为 `system:nodes`。
- 用途为 `server auth`，仅可附带可选的 `digital signature` 和 `key encipherment`。
- 至少一个 DNS/IP SAN。每个 DNS SAN 等于托管节点名称，每个 IP SAN 等于其记录的 IP。拒绝 Email 和 URI SAN。

预期进展为 pending、`Approved=True` 且原因为 `NKEValidatedNodeIdentity`，然后是非空签名证书。仅 `Approved` 不能证明已签名，仅签名不能证明节点已安装。身份不匹配的请求会被跳过并保持 pending；控制器不会将其拒绝作为自动纠正。

## 证明节点安装

在该节点上，使用特权会话并仅记录公开元数据。符号链接目标包含 kubelet 的服务证书**及其私钥**，因此仅在本地读取，切勿将其复制出节点作为证据：

```bash
sudo test -L /var/lib/kubelet/pki/kubelet-server-current.pem
sudo openssl x509 -in /var/lib/kubelet/pki/kubelet-server-current.pem \
  -noout -dates -fingerprint -sha256
sudo grep -E '^(serverTLSBootstrap|rotateCertificates):' /var/lib/kubelet/config.yaml
sudo systemctl is-active kubelet
```

将安装的 DER 指纹和 NotAfter 与签发的 CSR 证书比较。真实续期要求指纹相对基线发生变化且到期时间更晚。链接缺失、指纹不同、到期时间旧或 kubelet 停止均需调查。检查 RPC 在本地读取此固定的证书和密钥文件，仅返回公开证书元数据；它不会修改 kubelet 配置或触发续期。
验证新的 `kubelet-serving` 检查观察结果和节点就绪状态。控制器按检查间隔（默认一小时）刷新观察结果，因此新安装的证书可能需要该时长才会出现。在节点上，将安装的 NotAfter 打印为 Unix 秒（GNU `date`）：

```bash
date -u -d "$(sudo openssl x509 -in /var/lib/kubelet/pki/kubelet-server-current.pem \
  -noout -enddate | cut -d= -f2)" +%s
```

然后读取控制器指标，可通过 Prometheus 查询或直接按[读取管理平面指标](../operate/observe.md#read-the-management-plane-metrics)操作。在操作机上转发指标端口后，将 `CLUSTER` 设为工作负载集群名称，`NODE` 如上：

```bash
CLUSTER=example-tenant
curl -s http://127.0.0.1:9090/metrics | awk -v cluster="$CLUSTER" -v node="$NODE" '
  /^nvke_controller_certificate_(observation_available|not_after_timestamp_seconds)\{/ &&
  index($0, "credential=\"kubelet-serving\"") &&
  index($0, "tenant_cluster=\"" cluster "\"") && index($0, "node=\"" node "\"") {
    printf "%s %.0f\n", $1, $NF
  }'
```

要求 `certificate_observation_available` 为 `1`，且 `certificate_not_after_timestamp_seconds` 等于上面安装的 NotAfter。值为 `0`、序列缺失或 NotAfter 较旧，表示控制器仍报告之前的证书或无新观察结果；等待下一次检查，若持续存在则调查。然后检查节点就绪状态和租户 API：

```bash
kubectl --kubeconfig "$TENANT_KUBECONFIG" --context "$TENANT_CONTEXT" \
  wait --for=condition=Ready "node/$NODE" --timeout=60s
kubectl --kubeconfig "$TENANT_KUBECONFIG" --context "$TENANT_CONTEXT" \
  get --raw='/readyz?verbose'
```

节点文件证明已安装材料。此外，在该节点上使用租户 CA 和证书携带的身份验证实际 kubelet 端点：IP SAN 使用其清单 IP，仅 DNS SAN 使用托管节点名称（上面的 CSR 输出会显示是哪种）。连接始终发往清单 IP。此示例使用默认 kubelet 端口和 GNU `timeout`；保留 TLS 错误并要求两条管道命令均成功：

```bash
NODE_IP=192.0.2.10  # Replace with this node's managed InternalIP.
NODE_NAME=example-worker-1  # Replace with this node's managed name.
VERIFY_ARGS=(-verify_ip "$NODE_IP")
# For a DNS-only SAN, use instead: VERIFY_ARGS=(-verify_hostname "$NODE_NAME")
NODE_CONNECT_HOST="$NODE_IP"
case "$NODE_IP" in *:*) NODE_CONNECT_HOST="[$NODE_IP]" ;; esac
set -o pipefail
sudo timeout 10s openssl s_client -connect "$NODE_CONNECT_HOST:10250" \
  -CAfile /etc/kubernetes/pki/ca.crt "${VERIFY_ARGS[@]}" \
  -verify_return_error </dev/null |
  openssl x509 -noout -dates -fingerprint -sha256
```

要求 CA 检查通过且 IP 或主机名匹配，并且公开指纹与签发和安装的证书相同。端点证书旧、信任失败、连接失败或超时均属验证不完整。生产验收检查中不得绕过 TLS 验证。非默认端口/信任布局需要各自验证的端点探测。

## 故障与恢复

| 观察结果 | 检查与恢复 |
| --- | --- |
| 临近到期无新服务 CSR | 检查安装的到期时间、`serverTLSBootstrap`、kubelet 客户端认证、API 可达性和 kubelet 日志。修复受支持的 kubelet 配置；不要将客户端 CSR 签名器与服务签名器混淆。 |
| 身份跳过的 pending CSR | 对照请求者、主题、用途和 SAN 检查控制器期望的节点名称/IP。纠正实际清单/引导问题，让 kubelet 请求有效材料，并验证新请求。不要手动批准不匹配的请求。 |
| 有效 CSR 处于 pending 无决策 | 检查 tenant-manager 访问、控制器健康和批准 RBAC。恢复该访问；控制器可协调 pending 请求。保留失败区间作为告警/审计证据。 |
| 已批准但证书为空 | 检查 Kubernetes controller-manager 签名器可用性、签名证书/密钥配置和失败条件。恢复签名；批准不是签发操作。 |
| 已拒绝或失败 | 将请求视为终态；检查签发者/批准者记录的原因。控制器不会覆盖拒绝。修复根本原因并调查 kubelet 是否请求新材料。 |
| 已签发但节点仍使用旧证书 | 检查 kubelet 日志、PKI 权限和安装的当前链接。按站点节点恢复流程修复；新服务证书可能尚未加载。手动干预期间保持其余节点不动。 |
| CSR 指标缺失/过期 | 恢复 tenant-manager API 观察。缓存的零计数不能证明无故障。 |

在节点上，`sudo journalctl -u kubelet --since '30 minutes ago'` 提供本地诊断。在管理集群上，使用[控制器和 Temporal 诊断](https://github.com/dsx-ai-factory/nvke/blob/main/docs/operations/observability.md)。日志和 CSR 名称可能包含敏感操作上下文；保持原始诊断私有，仅导出经审查的固定原因/结果证据。不要仅为隐藏失败/拒绝的清单告警而删除终态 CSR 对象；恢复后遵循站点的证据保留和 Kubernetes 清理策略。

[监控指南](../../controller/docs/credential-monitoring.md#alert-responses)定义了过期、失败和不可用 CSR 告警。计数是当前对象清单；已批准但无签名证书的请求仍计为 pending。[审计指南](../../controller/docs/credential-audit.md#recorded-observations)记录身份/批准和观察到的签名结果；仅 `verify/observed` 不声称 kubelet 已安装其证书。将节点侧公开检查与这些事件关联，并保留任何观察缺口。

对于发布验收，通过受控的到期/续期运行观察实际 kubelet 的请求、批准、签发和安装，保留前后公开指纹和新鲜度/告警恢复，并记录目标身份、版本和流量影响。包括[实时验收清单](../../controller/docs/credential-monitoring.md#offline-verification-and-live-acceptance)要求的监控证据：警告窗口规则触发、配置接收器收到、续期后解决，以及不可用检查和过期或失败 CSR 告警。模拟 CSR 不能替代该资格验证。