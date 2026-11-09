# 故障排查与常见问题

## 引导

### 引导卡住 30 分钟后失败，且无明确错误

最常见原因是节点准备不完整。`nvkectl bootstrap site` 通过 `nvke-agent` gRPC 向节点下发 kubeadm 工作流；节点准备不完整时，会停在 `cluster state status=Provisioning` 并静默等待直至超时。请检查：

- containerd 已安装，且 `/etc/containerd/config.toml` 中 SystemdCgroup 设置为 `true`
- 内核模块已加载：`lsmod | grep -E 'overlay|br_netfilter'`
- `/opt/cni/bin/` 下存在 CNI 插件
- 所有节点上 `nvke-agent` 正在运行：`systemctl status nvke-agent`
- 从 init 节点可访问每个 join 节点的 15005 端口

完整节点准备清单见[预配置节点](../site-bootstrap/pre-provisioned-nodes.md)。

### 引导失败，报错 "bootstrap cluster … from an earlier run is still running"

删除残留的 Kind 集群前，如果 etcd 加密密钥是在失败的那次运行中生成的，请先导出——它可能是唯一可恢复的副本。密钥位于引导集群的控制器命名空间中，而非默认 kubeconfig：`nvke-controller`；若集群由旧版 `nvkectl` 创建，则为 `cloud-workflow-engine`。搜索所有命名空间，并从密钥所在命名空间保存每个密钥：

```bash
umask 077
export KIND_EXPERIMENTAL_PROVIDER=nerdctl
kind get kubeconfig --name nvkectl-bootstrap > bootstrap.kubeconfig
kubectl --kubeconfig bootstrap.kubeconfig get secrets -A | grep etcd-encryption-key-
# Save each Secret listed above, from the namespace it is listed in:
kubectl --kubeconfig bootstrap.kubeconfig -n <namespace> \
  get secret <etcd-encryption-key-name> -o yaml > etcd-encryption-key-backup.yaml
```

将 `etcd-encryption-key-backup.yaml` 存放到安全位置，然后删除集群。务必显式使用 `nerdctl`——直接运行 `kind delete cluster` 会用 Docker provider 查找集群，并以 0 退出而不删除任何内容：

```bash
KIND_EXPERIMENTAL_PROVIDER=nerdctl kind delete cluster --name nvkectl-bootstrap
```

### `clusterctl init` 在 GitHub release URL 上返回 404

这是因为 `nvkectl` 是用 `go build` 而非 `make build` 构建的。`capiVersion` 链接器标志仅由 Makefile 设置；直接 `go build` 会将其留为 `dev`，运行时构造出无效的 GitHub release URL。

请始终使用以下命令构建：
```bash
make -C cli build
```

验证版本已写入：
```bash
nvkectl version
```

输出应显示真实版本字符串，而非 `dev`。

### 引导失败，报错 "port 6443 is accepting connections on …"

上一次引导尝试在某个节点上遗留了控制平面。重试前先重置：

```bash
ls /etc/kubernetes/manifests   # kube-apiserver.yaml here means a control plane
sudo kubeadm reset -f
```

### 节点加入后始终不进入 Ready

检查 join 节点上的 `nvke-agent`：

```bash
sudo systemctl status nvke-agent
sudo journalctl -u nvke-agent -n 50
```

确认 init 节点可访问 join 节点的 15005 端口：

```bash
curl -v telnet://<join-node-ip>:15005
```

对于未启用 mTLS 的预配置节点（仅限开发环境——见下方说明），确认 `/etc/default/nvke-agent` 中设置了 `--disable-mtls`。

> **注意：** `--disable-mtls` 会关闭控制器与 `nvke-agent` 之间 gRPC 连接的传输认证和加密。仅在隔离的开发环境中使用。生产部署使用 SPIFFE/SPIRE 支持的 mTLS，不设置此标志。

## 租户集群

### `nvkectl get providers` 显示 `nico: NOT-CONFIGURED`

CAPNICo 未就绪。请检查：

1. 管理集群的 `capnico-system` 中存在 `nico-credentials` Secret
2. 从管理集群可访问 Secret 中的 NICo API 端点
3. CAPNICo 控制器正在运行：`kubectl get pods -n capnico-system`
4. CAPNICo 日志中的认证错误：`kubectl logs -n capnico-system deploy/controller-manager`

### `nvkectl get providers` 显示 `nico: CONFIGURED`，但 `CREDENTIALS` 不是 `HEALTHY`

`CREDENTIALS` 是 CAPNICo 对凭据 Secret 的最近一次检查结果，读取自引用该 Secret 的 `NicoIdentity`。

- `FAILING (reason)`：修复 reason 所指的问题。`CredentialsNotFound` 和 `InvalidConfiguration` 指向 `nico-credentials` Secret；`AuthenticationFailed` 和 `AccessDenied` 表示 NICo 拒绝了凭据或禁止访问。Secret 变更后 CAPNICo 会在数秒内重新检查。
- `UNKNOWN (ValidationFailed)`：CAPNICo 无法访问 NICo，或检查超时、TLS 验证失败。检查 Secret 中的端点及从管理集群到该端点的网络路径。
- `STALE`：到了应检查的时间但 CAPNICo 未检查。nvkectl 能看到原因时，`DETAIL` 会指明：管理器未运行，或它监视的是其他命名空间或 `--watch-filter`。`DETAIL` 未指明原因时，检查管理器是否来自不含 `NicoIdentity` 控制器的 CAPNICo 版本——该版本从不检查 `NicoIdentity`。NICo 故障期间存在大量 `NicoIdentity` 对象时，检查也可能变慢而非停止，因为 CAPNICo 逐个检查。参见[调试 NicoIdentity 凭据健康状态](../operate/debug-nicoidentity-credentials.md)。
- `PENDING`：CAPNICo 尚未检查该 `NicoIdentity` 或其最新变更。几秒后重新运行命令；若仍无检查，会转为 `STALE`。
- `NO-IDENTITY`：没有 `NicoIdentity` 引用该 Secret。请创建一个；参见[基础设施提供商](../../cli/README.md#infrastructure-providers)。
- `UNSUPPORTED`：已安装的 CAPNICo 版本早于 `NicoIdentity`。
- `UNREADABLE`：站点 kubeconfig 无法列出 `NicoIdentity` 对象，`DETAIL` 指明错误。若列出被禁止，在 Secret 所在命名空间授予 `nicoidentities.infrastructure.cluster.x-k8s.io` 的 `list` 权限；kubeconfig 权限见[向 CWE 认证](../../cli/README.md#authenticating-to-cwe)。

### `nvkectl create cluster` 返回 500

集群 spec YAML 中有未替换的占位符值。常见的有：`orgID`、`poolID`、发布版本名称和配置 profile 名称。它们必须分别存在于你的 NICo 站点和管理平面上。请从 `cli/examples/nico-cluster.yaml` 开始，替换所有 UUID 和 profile 名称为你的环境值。

### 集群卡在 `Provisioning` 无进展

打开 Temporal Web UI（见[可观测性](observability.md)），找到该集群的活动工作流。当前 activity 显示卡在哪里。常见原因：

- NICo 没有所请求实例类型的可用容量——检查 NICo 站点的分配
- 配置的子网 IP 地址耗尽
- CAPNICo 无法访问 NICo API——检查 `capnico-system` Pod 日志

### 期望数量增加后节点卡在 `Pending`

自动配置在等待匹配的容量。请检查：
- NICo 站点有匹配节点池实例类型的可用主机
- 配置的子网有空闲 IP 地址
- CAPNICo 已安装且凭据可用（`nvkectl get providers` 显示 `nico: CONFIGURED`，在 CAPNICo 提供 `NicoIdentity` 时 `CREDENTIALS` 为 `HEALTHY`）

## 管理平面

### `nvkectl get clusters` 返回 401

管理平面启用了认证。`nvkectl` 使用传给 `nvkectl config set-site` 的 kubeconfig 生成 bearer token。确保管理集群上存在 `nke-clients/nke-client` 服务账号，且 kubeconfig 身份具有所需权限。参见[配置 API 认证](../../controller/docs/configuring-api-authentication.md)。

### 引导后节点不可调度

`nvkectl bootstrap site` 后所有节点都是控制平面节点；所有节点加入后，该命令会移除每个节点上的控制平面 `NoSchedule` 污点。仅当移除失败且命令已提示时，节点才会保留污点。安装工作负载前手动移除污点：

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule- \
  --kubeconfig <sitename>.kubeconfig
```

### 数据库 Pod 处于 `Pending`——无可用持久卷

引导后站点集群没有存储类。部署 NVKE 组件栈前先安装存储 provisioner。以下示例使用 local-path-provisioner v0.0.37：

```bash
# Verify the manifest before applying
curl -sL https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.37/deploy/local-path-storage.yaml \
  | sha256sum
# Apply once verified
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.37/deploy/local-path-storage.yaml \
  --kubeconfig <sitename>.kubeconfig
```

## 通用

### 如何查看正在运行的 NVKE 版本？

```bash
nvkectl version                          # CLI version
kubectl get deploy -n nvke-controller nvke-controller \
  -o jsonpath='{.spec.template.spec.containers[0].image}' \
  --kubeconfig <sitename>.kubeconfig    # controller image tag
```

### 控制器工作流日志在哪里？

从管理集群端口转发 Temporal Web UI，查找 `nvke-controller` 命名空间中的工作流。参见[可观测性](observability.md)。

### 如何获取租户集群的 kubeconfig？

```bash
nvkectl get kubeconfig <cluster-name> --output-file tenant.kubeconfig
```