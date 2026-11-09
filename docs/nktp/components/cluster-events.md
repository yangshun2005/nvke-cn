# 集群事件

集群事件读取器将 Kubernetes Event 日志发送到日志网关。两个副本通过领导者选举共享同一个 watch。其自身的指标、日志和追踪通过[边车](../architecture/self-telemetry.md)发送到 watchdog。

所有 Event 日志都会获得 NVKE 遥测流水线（NKTP）资源属性 `nvidia.telemetry.class: k8s.event`，从而独立于上报控制器或 service-name 回退来标识其类别。

Event 日志使用上报控制器作为 `service.name`，当该值缺失或为空时回退到 `kube-apiserver`。读取器将 Kubernetes 的 `reportingController` 和核心 API 的 `reportingComponent` 都暴露为 `k8s.event.reporting_controller`；不暴露旧版 `source.component`。Loki 将 Event 的 service name 暴露为 `service_name`。读取器自身的遥测继续使用 `nktp-cluster-events`。

该回退可能将来源不明的 Event 错误地与 `kube-apiserver` 遥测归为一组。要选择其他回退值，请覆盖以下语句，将 `unknown_service` 替换为任意所需名称，例如 `k8s-events`。Helm 会替换整个语句列表，因此请保留分类语句和控制器语句。删除最后一条语句可在控制器不可用时保持 service name 未设置。

```yaml
cluster_events:
  otelcol:
    config:
      processors:
        transform/event_service:
          log_statements:
            - context: log
              statements:
                - set(resource.attributes["nvidia.telemetry.class"], "k8s.event")
                - set(resource.attributes["service.name"], attributes["k8s.event.reporting_controller"]) where (resource.attributes["service.name"] == nil or resource.attributes["service.name"] == "") and attributes["k8s.event.reporting_controller"] != nil and attributes["k8s.event.reporting_controller"] != ""
                - set(resource.attributes["service.name"], "unknown_service") where resource.attributes["service.name"] == nil or resource.attributes["service.name"] == ""
```

使用 `cluster_events.otelcol.config` 配置 Collector，使用 `cluster_events.kubernetes` 配置 Kubernetes 工作负载。替换其环境变量列表时，请保留 `GOMEMLIMIT`、`GOGC` 和 `POD_NAMESPACE`。在 Collector 配置中设置 `receivers.k8s_events.namespaces` 以限制被 watch 的命名空间。领导者交接期间的事件可能会丢失。Collector Pod 继承 `priorityClass.name`；覆盖 `cluster_events.kubernetes.priorityClassName` 以使用其他优先级类，或设为 `""` 以退出。 

Chart 在发布命名空间中创建选举 Lease，并将读取器的 Lease 权限限制为该名称。在 Collector 配置中设置 `extensions.k8s_leader_elector.lease_name` 可更改名称。请使用字面名称，以便 Helm 创建匹配的 Lease。如果使用其他 Lease 命名空间，请在那里单独配置 Lease 及其 RBAC。

按活跃读取器而非活跃与备用副本的平均值来规划容量。