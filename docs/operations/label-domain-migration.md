# `nvke.nvidia.com` 标签与注解域

NVKE 将其拥有的标签、注解、污点和节点状况写在 `nvke.nvidia.com/` 下。过去写在 `nke.nvidia.com/` 下，现在仍会在所有位置*读取*旧形式，因此现有站点无需重新打标签，也无需维护窗口。

全新安装无需任何操作。以下说明面向现有站点，以及 NVKE 之外设置或监听这些键的任何系统。

## 变更内容

| 位置 | 之前 | 之后 |
|---|---|---|
| NVKE 写入 | `nke.nvidia.com/<name>` | `nvke.nvidia.com/<name>` |
| NVKE 读取 | `nke.nvidia.com/<name>` | 两者，优先新形式 |
| NVKE 删除 | `nke.nvidia.com/<name>` | 两者 |

域名后的名称不变，因此 `nke.nvidia.com/reboot` 变为 `nvke.nvidia.com/reboot`，其余部分均不变。

## 无需执行的操作

- **无需为现有对象重新打标签。** NVKE 读取两个域。由早期版本打标签的对象照常工作。
- **无需与 NVKE 中的其他组件协调升级。** NVKE 中每个读取方都接受两种形式，无论以何种顺序上线。CAPNICo 是例外，它不是 NVKE 的读取方——见下文。
- **无需改变请求重启或替换的方式。** `nke.nvidia.com/reboot` 和 `nke.nvidia.com/terminate` 仍然有效。新自动化中建议使用新形式；旧形式在本版本中不会移除。

NVKE 重写对象时，对象会自行收敛到新域。节点拓扑标签在节点下次 reconcile 时迁移；kubeconfig Secret 归属标签在 Secret 下次发布时迁移；provisioning 污点由下一个加入的节点写入新域。

## 必须检查的事项

**`cluster-api-provider-nico` 的重启和修复标志必须指向新域。** 这是唯一一对只配置了单一读取方的键。CAPNICo 分别只监听一个注解，由 `--reboot-annotation` 和 `--repair-annotation` 指定，并静默忽略其他任何键下的请求。NVKE 只写入 `nvke.nvidia.com/*`，因此仍指向 `nke.nvidia.com/*` 的 CAPNICo 根本看不到请求：

```yaml
- --reboot-annotation=nvke.nvidia.com/reboot
- --repair-annotation=nvke.nvidia.com/machine-health-issue
```

不匹配不会在任何地方产生错误。CAPNICo 查找其配置的键，找不到便返回——没有日志行、没有指标、Machine 上没有状况。重启工作流等待一个永远不会到来的 boot-ID 变化；修复路径删除实例，却从未将其标记为待修复。第一个迹象是需要回收的节点没有被回收。

本仓库中的清单已设置新值。如果站点使用自己的清单部署 CAPNICo，请在升级 NVKE **之前**迁移这些标志。验证实际运行的内容。读取每个运行中 Pod 的参数，而不是 Deployment 模板：在未完成的滚动更新中，模板可能显示新标志，而旧 Pod 仍在运行旧标志。

```sh
kubectl -n capnico-system get pods -l control-plane=controller-manager \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\t"}{.spec.containers[?(@.name=="manager")].args}{"\n"}{end}'
```

列出的每个 Pod 都必须显示两个新标志，且不得有 Pod 仍在滚动更新。

**NVKE 之外读取这些键的任何系统。** 匹配 `nke.nvidia.com/...` 的仪表盘、告警规则、脚本和故障修复自动化，对 NVKE 尚未重写的对象仍然有效，一旦重写便不再匹配。迁移期间同时匹配两者，例如：

```promql
kube_node_labels{label_nvke_nvidia_com_site_name!=""} or kube_node_labels{label_nke_nvidia_com_site_name!=""}
```

设置 `NeedsRecycle` 或 `NeedsRepair` 的任何系统（如故障修复系统）可使用任一域；NVKE 两者都读取。

## 未变更的内容

以下名称中包含 `nke.nvidia.com`，但**不属于**本次变更，因为重命名是独立的破坏性操作：

| 名称 | 保留原因 |
|---|---|
| `auth.nke.nvidia.com/v1alpha1` | 是 API 组，不是标签域 |
| `owner.nke.nvidia.com/<kind>` | Informer 标签选择器无法同时匹配两个键，重命名会隐藏早期版本打标签的所有对象 |
| `controller.nke.nvidia.com`、`management-engine.nke.nvidia.com` | 领导者选举 ID；在滚动更新中途更改会导致两个领导者同时运行 |
| `internal.nke.nvidia.com` | 见 [crd-group-migration.md](crd-group-migration.md) |