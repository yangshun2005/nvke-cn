# kube-state-metrics

`kube-state-metrics.daemonset` 下的 DaemonSet 收集其所在节点上已调度的 Pod。
在 `kube-state-metrics.deployments` 下配置中心工作负载。每个中心工作负载的 `resource_types`
选择要收集的 Kubernetes 资源类型，并授予匹配的 RBAC。
设置 `track_unscheduled_pods: true` 可收集未分配节点的 Pod。
此功能仅支持中心工作负载，且要求 `resource_types` 恰好为 `[pods]`。
资源名称来自[上游资源列表](https://github.com/kubernetes/kube-state-metrics/blob/v2.20.0/docs/README.md#exposed-metrics)。

`mutatingadmissionpolicies` 和 `mutatingadmissionpolicybindings` 默认禁用，
因为 KSM 使用其 `admissionregistration.k8s.io/v1` API，旧版集群不提供这些 API。
仅当集群同时提供这两个 API 时，才将它们加入 `resource_types`。

`kube-state-metrics.namespaces` 设置命名空间范围，除非工作负载自带 `namespaces` 列表。
RBAC 遵循生效的命名空间选择。
空列表表示所有命名空间。空字符串会被忽略。

设置 `kube-state-metrics.metricLabelsAllowlist` 可暴露选定的 Kubernetes 标签，
例如 `["pods=[app.kubernetes.io/name]"]`。其他 Kubernetes 标签默认禁用。

Secret 指标需主动启用，因为收集需要 list/watch 访问完整的 Secret 对象，包括其内容。要启用它们，请将
`kube-state-metrics.deployments.configuration.resource_types` 设为 `[configmaps, secrets]`。
设置该工作负载的 `namespaces` 列表以限制访问。空列表授予整个集群的访问权限。

共享的 `kubernetes` 默认值可在每个工作负载下覆盖。`resources`、
`env` 和 `securityContext` 配置 KSM 容器，`podSecurityContext`
配置 Pod 安全，其余字段配置 Pod spec。
设置 `serviceAccount` 以使用现有账户。

覆盖调度设置时，确保节点 Collector 运行在 KSM DaemonSet 运行的每个节点上，
以便抓取本地 KSM 端点。

当中心工作负载超出其内存或抓取响应预算时，增加 `shards`。Helm 提供 `--shard` 和 `--total-shards`。每个分片作为一个
Deployment 运行，并在替换期间短暂停止收集。

[kube-state-metrics 设计](../designs/kube-state-metrics.md)比较了扩展 KSM 收集时考虑的
各种方案。