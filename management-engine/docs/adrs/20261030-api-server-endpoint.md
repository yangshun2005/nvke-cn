# NVKECluster 上由调用方提供的 API server 端点

**状态：** 提议中
**日期：** 2026-10-30

## 背景

NVKE 控制器会拒绝多个控制平面节点、升级以及控制平面节点替换，除非集群创建时带有 `controlPlaneVIP`（[`control_plane_vip.go`](../../../controller/internal/server/control_plane_vip.go)）。它只在创建时接受该 VIP；其更新请求没有对应字段（[`models.gen.go`](../../../controller/api/models.gen.go)）。`NVKECluster` 无法传入 VIP，仓库中也没有任何组件负责分配 VIP 或校验其唯一性。

## 决策

我们将新增可选的 `spec.apiServerEndpoint.address`，仅在创建时作为 `controlPlaneVIP` 发送给控制器。

- 该地址必须是单播 IPv4，且位于 Pod 网络和服务网络之外。调用方需确保没有其他集群使用它。
- 错误通过 `ClusterReconciled` 条件上报，且绝不提及另一个集群或租户。
- 该字段与 `apiServerAccess` 平级，而非嵌套其中：前者是 API server 的提供位置，后者是允许谁访问它。

## 影响

通过集群服务创建的集群可以运行高可用控制平面并支持升级。

在唯一性校验落地之前，一个租户可以占用另一个租户的地址，导致其 API server 不可用。

集群服务会在第一个通过校验的 reconcile 中创建集群，因此只有在 spec 的其他部分被拒绝时，该地址才可编辑。此后，地址错误就意味着重建集群。

后续需求均可增量添加：

- **托管分配：** 不设置 spec，在 status 中上报所选地址。这也是 status 拥有独立副本的原因。
- **主机名和端口：** 添加到 status 类型中，该类型因此与 spec 类型分离。字段命名为 `address` 而非 `vip` 也是同一原因。
- **公共端点：** 当 `apiServerAccess.publicAccessEnabled` 为 true 时，在 status 中上报。
- **唯一性与按租户划分的地址段：** 仅做校验，不改动 API。

## 备选方案

- **调用方提供、上报后固定（已选）：** 满足控制器需求，且服务无需管理地址生命周期。
- **仅由服务分配：** 对用户更简单，但仓库中没有任何组件负责分配地址。
- **`spec.controlPlaneVIP`：** 改动最小，但名称绑定单一机制，无法容纳主机名或端口。
- **首次 apply 时固定：** 更简单，但会阻止修复从未创建成功的集群上的拼写错误。
- **放在 `apiServerAccess` 下：** 混淆了位置与访问，且访问设置可在创建后变更。