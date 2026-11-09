# kube-state-metrics

## 目的

NVKE 遥测流水线（NKTP）使用原生 kube-state-metrics（KSM）暴露 Kubernetes 对象状态。
在 AI 规模 Kubernetes 集群中，KSM 必须平衡抓取响应大小、自身每 Pod 及聚合 CPU 和内存用量、Kubernetes API 服务器负载，
以及并发 API watch。以下设计比较了应对这些扩展压力的方式及各自权衡。

## 设计方案

评级比较各设计在各维度上的扩展表现。

- **🟢 良好**：在该维度上扩展良好。
- **🟡 混合**：可扩展，但有显著权衡。
- **🟠 差**：在该维度上扩展效率低。
- **🛑 不足**：存在已知扩展瓶颈且无法解决，因为受影响的工作负载无法进一步拆分。

### 1. 单体 KSM

单个 KSM Pod 通过 API 服务器 list 和 watch 请求收集所有选定的 Kubernetes 对象，保留其状态并暴露其指标。
[`--resources` 选项][ksm-options]选择资源类型。

```text
# Pseudocommand for each KSM pod
ksm --resources=<all-resource-types>
```

- **抓取响应大小。** 🛑 *不足*。大规模下，单次抓取响应可能超出单个 Collector Pod 的处理能力。
- **每 Pod KSM 需求。** 🛑 *不足*。所有处理和状态都集中在一个 Pod 中。
- **聚合 KSM 需求。** 🟢 *良好*。每个对象只解码一次，仅有一个 KSM 进程的开销。
- **API 服务器负载。** 🟢 *良好*。API 服务器将每个对象提供给一个 KSM 实例，无重复 API 服务器负载。
- **API 服务器 watch。** 🟢 *良好*。仅一组资源 watch。

### 2. 哈希分片

KSM 的[哈希分片][ksm-hash-sharding]使用每个对象的唯一标识符（UID）将对象分配到分片。

```text
# Pseudocommand for each KSM pod
ksm --resources=<all-resource-types> --shard=0 --total-shards=2
ksm --resources=<all-resource-types> --shard=1 --total-shards=2
```

- **抓取响应大小。** 🟢 *良好*。`N` 个分片将每个 KSM Pod 的抓取响应大小降至约 `1/N`。必须配置足够分片以保持每个响应
  可管理。单个对象的指标仍在一个分片上。总抓取响应字节数大致不变，只是分到各分片。
- **每 Pod KSM 需求。** 🟡 *混合*。`N` 个分片将每个 KSM Pod 保留的状态降至约 `1/N`。每个分片仍需先解码所有接收到的对象，
  再选择保留哪些。
- **聚合 KSM 需求。** 🟠 *差*。KSM 总解码工作量增长约 `N` 倍。用于保留对象状态的总内存大致不变，
  只是分到各分片。CPU 和内存请求在所有分片上累加。
- **API 服务器负载。** 🟠 *差*。向所有 `N` 个分片提供相同对象，使 API 服务器负载增加约 `N` 倍。
- **API 服务器 watch。** 🟡 *混合*。每个分片都为所有选定资源类型打开 watch，使 API 服务器 watch 数乘以 `N`。

### 3. 分区收集

不同的 KSM 实例通过 [KSM 的 `--resources` 选项][ksm-options]选择，向 API 服务器请求不同的资源类型。

```text
# Pseudocommand for each KSM pod
ksm --resources=pods
ksm --resources=deployments,statefulsets,daemonsets
ksm --resources=<remaining-resource-types>
```

- **抓取响应大小。** 🛑 *不足*。在源头过滤可减少每个实例在抓取响应中暴露的数据。总抓取响应字节数大致
  不变，只是分到各实例。某个分区仍可能过大，例如 Pods。
- **每 Pod KSM 需求。** 🛑 *不足*。在源头过滤可减少每个实例处理和保留的数据。资源分区可能不均衡，因此每个
  实例可能需要独立的 CPU 和内存预算。
- **聚合 KSM 需求。** 🟢 *良好*。在所有实例中，处理对象所需的总 CPU 和保留其状态所需的内存与单体 KSM 大致
  相同。
- **API 服务器负载。** 🟢 *良好*。每种资源类型由一个 KSM 实例 watch，避免哈希分片的重复 API 服务器负载。
  对于相同对象覆盖范围，这些对象的 API 服务器负载约为单体基线的 `1x`，而非哈希分片的 `Nx`。
- **API 服务器 watch。** 🟢 *良好*。仅一组资源 watch，分到各 KSM 实例。

### 4. DaemonSet

每个节点运行一个 KSM Pod，各自通过 [`--resources` 选项][ksm-options]收集所有选定资源类型。

```text
# Pseudocommand for each KSM pod
ksm --resources=<all-resource-types> # On node-a
ksm --resources=<all-resource-types> # On node-b
# One Pod on each remaining node.
```

- **抓取响应大小。** 🛑 *不足*。每个响应仍覆盖整个集群。
- **每 Pod KSM 需求。** 🛑 *不足*。每个实例保留整个集群选定对象的状态。
- **聚合 KSM 需求。** 🟠 *差*。相同收集的 CPU 和内存成本在每个节点重复。
- **API 服务器负载。** 🟠 *差*。每个实例收集相同对象，使 API 服务器负载乘以节点数。
- **API 服务器 watch。** 🟠 *差*。每个节点打开全套资源 watch。

### 5. Pod DaemonSet + 哈希分片

使用 [KSM 的节点过滤][ksm-daemonset-sharding]在其节点上收集已调度的 Pod。
集中收集未调度 Pod 和所有其他资源类型，并在整个集中收集上使用[哈希分片][ksm-hash-sharding]。
[`--track-unscheduled-pods` 选项][ksm-options]将集中 Pod 收集限制为无节点分配的 Pod。

```text
# Pseudocommand for each KSM pod
ksm --resources=<all-resource-types> --track-unscheduled-pods --shard=0 --total-shards=2
ksm --resources=<all-resource-types> --track-unscheduled-pods --shard=1 --total-shards=2

ksm --resources=pods --node=node-a
ksm --resources=pods --node=node-b
# One Pod on each remaining node.
```

- **抓取响应大小。** 🟢 *良好*。DaemonSet 响应跟随本地 Pod 数量而非整个集群。
  哈希分片将集中响应分到足够分片，使每个响应可管理。
- **每 Pod KSM 需求。** 🟡 *混合*。DaemonSet 需求跟随节点本地 Pod 数。哈希分片减少每个集中 Pod 保留的状态，
  但每个集中分片仍解码所有未调度 Pod 和其他选定资源类型。
- **聚合 KSM 需求。** 🟠 *差*。已调度 Pod 收集未重复，但总解码工作量仍可能达到单体基线的 `N` 倍中相当大的一部分，
  因为每个集中分片都解码所有剩余资源类型。DaemonSet 在每个节点预留 CPU 和内存请求。
- **API 服务器负载。** 🟠 *差*。已调度 Pod 收集避免重复 API 服务器负载，但总 API 服务器负载仍可能达到单体基线的
  `N` 倍中相当大的一部分，因为所有 `N` 个集中分片都收集每种剩余资源类型，包括不需要分片的类型。
- **API 服务器 watch。** 🟡 *混合*。每节点一个 Pod watch，加上每个分片全套集中资源 watch。

### 6. Pod DaemonSet + 资源分区

使用 [KSM 的节点过滤][ksm-daemonset-sharding]，使每个 DaemonSet 实例只收集自己节点上的已调度 Pod。
对未调度 Pod 和其他资源类型使用独立的集中工作负载。
低基数资源类型可配置为共享一个 KSM 实例，高基数类型则单独运行。
KSM 的 [`--track-unscheduled-pods` 选项][ksm-options]覆盖无节点分配的 Pod。

```text
# Pseudocommand for each KSM pod
ksm --resources=pods --track-unscheduled-pods
ksm --resources=deployments,statefulsets,daemonsets
ksm --resources=<remaining-resource-types>

ksm --resources=pods --node=node-a
ksm --resources=pods --node=node-b
# One Pod on each remaining node.
```

- **抓取响应大小。** 🛑 *不足*。DaemonSet 响应跟随本地 Pod 数量而非整个集群。
  即使有专用集中 KSM 实例，高基数资源类型仍可能超出单个 Pod 的抓取响应预算。
- **每 Pod KSM 需求。** 🛑 *不足*。DaemonSet 将 Pod（通常是最高的基数资源类型之一）的收集分布到各节点，使
  每实例 KSM 需求随节点本地 Pod 数而非集群范围 Pod 数增长。
  按节点上监控的 Pod 数为每个 DaemonSet Pod 的 CPU 和内存预算设定大小。集中工作负载有独立的 CPU 和内存预算。
  即使有专用 KSM 实例，高基数资源类型仍可能超出单个 Pod 的内存预算。
- **聚合 KSM 需求。** 🟡 *混合*。每个对象由一个 KSM 实例接收和解码。DaemonSet 的请求在每个节点预留，
  此外还有集中工作负载的请求。
- **API 服务器负载。** 🟢 *良好*。API 服务器将每个对象提供给一个 KSM 实例，无重复 API 服务器负载。
- **API 服务器 watch。** 🟡 *混合*。仍有每节点 watch 开销，加上集中资源 watch。

### 7. Pod DaemonSet + 资源分区 + 选择性分片

先按资源类型划分集中工作负载。仅当单个资源类型仍超出 Pod 内存或抓取响应预算时，才将哈希分片作为最后手段。
例如，Deployments、StatefulSets 和 DaemonSets 可能小到可以一起收集。
Jobs 可能单独收集但仍超出单个 Pod 预算，因此此示例中仅对 Jobs 分片。

```text
# Pseudocommand for each KSM pod
ksm --resources=pods --track-unscheduled-pods
ksm --resources=deployments,statefulsets,daemonsets
ksm --resources=jobs --shard=0 --total-shards=2
ksm --resources=jobs --shard=1 --total-shards=2
ksm --resources=<remaining-resource-types>

ksm --resources=pods --node=node-a
ksm --resources=pods --node=node-b
# One Pod on each remaining node.
```

- **抓取响应大小。** 🟢 *良好*。DaemonSet 响应跟随本地 Pod 数量而非整个集群。
  选择性哈希分片划分剩余大型资源类型，使集中抓取响应可管理。
- **每 Pod KSM 需求。** 🟢 *良好*。DaemonSet 将 Pod（通常是最高的基数资源类型之一）的收集分布到各节点，使
  每实例 KSM 需求随节点本地 Pod 数而非集群范围 Pod 数增长。
  按节点上监控的 Pod 数为每个 DaemonSet Pod 的 CPU 和内存预算设定大小。集中工作负载有独立的 CPU 和内存预算。
  选择性哈希分片减少这些高基数资源类型每 Pod 保留的状态，但每个分片仍解码其资源类型的所有对象。
- **聚合 KSM 需求。** 🟡 *混合*。小型资源类型共享实例。资源分区避免重复 KSM 解码工作。分片只为需要它的资源类型增加请求
  和解码工作，而非在所有收集中成倍增加。DaemonSet 仍在每个节点预留请求。
- **API 服务器负载。** 🟢 *良好*。资源分区避免重复 API 服务器负载。选择性哈希分片只为需要它的资源类型增加 API 服务器负载，
  而非在所有收集中成倍增加。
- **API 服务器 watch。** 🟡 *混合*。DaemonSet 的每节点 watch 开销在大规模下仍需验证。
  选择性哈希分片带来的额外 watch 预计与每节点 watch 相比可忽略。

## 设计比较

| 设计 | 抓取响应大小 | 每 Pod KSM 需求 | 聚合 KSM 需求 | API 服务器负载 | API 服务器 watch |
| --- | --- | --- | --- | --- | --- |
| 单体 KSM | **🛑 不足** | **🛑 不足** | **🟢 良好** | **🟢 良好** | **🟢 良好** |
| 哈希分片 | **🟢 良好** | **🟡 混合** | **🟠 差** | **🟠 差** | **🟡 混合** |
| 分区收集 | **🛑 不足** | **🛑 不足** | **🟢 良好** | **🟢 良好** | **🟢 良好** |
| DaemonSet | **🛑 不足** | **🛑 不足** | **🟠 差** | **🟠 差** | **🟠 差** |
| Pod DaemonSet + 哈希分片 | **🟢 良好** | **🟡 混合** | **🟠 差** | **🟠 差** | **🟡 混合** |
| Pod DaemonSet + 资源分区 | **🛑 不足** | **🛑 不足** | **🟡 混合** | **🟢 良好** | **🟡 混合** |
| Pod DaemonSet + 资源分区 + 选择性分片 | **🟢 良好** | **🟢 良好** | **🟡 混合** | **🟢 良好** | **🟡 混合** |

## 建议

对于 AI 规模集群，使用设计 7，即结合 Pod DaemonSet、资源分区和选择性哈希分片。

## 配置形态

```yaml
kube-state-metrics:
  namespaces: [] # All namespaces.
  daemonset:
    enabled: true
    kubernetes: ... # Similar to OTel workloads.
  deployments:
    unscheduled-pods:
      enabled: true
      resource_types: [pods]
      track_unscheduled_pods: true # Requires resource_types to be exactly [pods].
      shards: 1
      kubernetes: ...
    workloads:
      enabled: true
      resource_types: [deployments, statefulsets, daemonsets, replicasets, replicationcontrollers, jobs, cronjobs, horizontalpodautoscalers, poddisruptionbudgets]
      track_unscheduled_pods: false
      shards: 1
      kubernetes: ...
    networking:
      enabled: true
      resource_types: [services, endpointslices, ingresses, networkpolicies]
      track_unscheduled_pods: false
      shards: 1
      kubernetes: ...
    storage:
      enabled: true
      resource_types: [persistentvolumes, persistentvolumeclaims, volumeattachments]
      track_unscheduled_pods: false
      shards: 1
      kubernetes: ...
    configuration:
      enabled: true
      resource_types: [configmaps]
      track_unscheduled_pods: false
      shards: 1
      kubernetes: ...
    cluster:
      enabled: true
      resource_types: [nodes, namespaces, storageclasses, resourcequotas, limitranges, leases, certificatesigningrequests, mutatingwebhookconfigurations, validatingadmissionpolicies, validatingadmissionpolicybindings, validatingwebhookconfigurations]
      track_unscheduled_pods: false
      shards: 1
      kubernetes: ...
```

部署名称由用户定义，并组成如 `nktp-ksm-workloads` 的名称。`kubernetes` 透传工作负载设置。

共享命名空间列表适用，除非工作负载提供自己的 `namespaces` 列表。空列表选择所有命名空间。空字符串被忽略。
Helm 根据每个集中工作负载的 `resource_types` 和有效命名空间选择生成收集参数和 RBAC。

DaemonSet 始终收集其节点上的已调度 Pod。`track_unscheduled_pods: true` 选择无节点分配的 Pod，不论 Pod 阶段，
仅支持集中工作负载，且要求 `resource_types` 恰好为 `[pods]`。默认为 `false`。
Helm 通过 downward API 提供 DaemonSet 的 `--node` 参数和 `NODE_NAME`。

`shards` 设置集中工作负载的哈希分片数。Helm 提供 `--shard` 和 `--total-shards`。

## 建议的默认分组

| 工作负载 | 资源 |
| --- | --- |
| `daemonset` | 已调度 Pod，按节点收集 |
| `deployments.unscheduled-pods` | 无节点分配的 Pod |
| `deployments.workloads` | 控制器、Jobs、自动扩缩器和中断预算 |
| `deployments.networking` | Services、EndpointSlices、Ingresses 和 NetworkPolicies |
| `deployments.storage` | PersistentVolumes、PersistentVolumeClaims 和 VolumeAttachments |
| `deployments.configuration` | ConfigMaps |
| `deployments.cluster` | Nodes、Namespaces、StorageClasses、配额、限制、Leases、证书和支持的准入资源 |

默认将每个选定资源类型分配到一个收集组，每个集中工作负载一个分片，除非其大小需要更多。

Secret 指标为可选启用，因为收集需要 list/watch 访问完整 Secret 对象，包括其内容。
将 `secrets` 添加到工作负载的 `resource_types