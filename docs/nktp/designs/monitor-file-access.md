# 监控文件访问限制：理由与兼容性

采集器可使用挂载的 ServiceAccount 令牌向受保护的指标端点进行身份验证。例如，vLLM 和 Dynamo operator 使用 controller-runtime 的认证指标服务器（由 [Kubebuilder](https://book.kubebuilder.io/reference/metrics.html) 记录），该服务器会校验令牌身份及其 `get /metrics` 权限。

能创建或更新指定 ServiceMonitor 的用户可滥用 `bearerTokenFile`，将采集器的令牌发送到其控制的端点。设置 `denyFSAccessThroughSMs: true` 可缓解此凭据泄露（见 [CVE-2026-47701](https://www.cve.org/CVERecord?id=CVE-2026-47701)），方式是丢弃采集器容器中引用 bearer 令牌或 TLS 文件的生成抓取任务。参见[上游公告](https://github.com/open-telemetry/opentelemetry-operator/security/advisories/GHSA-cxh2-4639-vmc5)。

为兼容使用文件引用的监控，node 和 large-target Target Allocator 默认将此设置设为 `false`，包括：

- [Dynamo 的 operator ServiceMonitor，提交 `02da7f4`](https://github.com/ai-dynamo/dynamo/blob/02da7f44861b65758b960770e0763e9faa8e4ca4/deploy/helm/charts/platform/components/operator/templates/operator-servicemonitor.yaml#L46)。
- [opentelemetry-kube-stack 的 API-server ServiceMonitor，提交 `042e2bb`](https://github.com/open-telemetry/opentelemetry-helm-charts/blob/042e2bb5c33639aade252ba72251e5ad414af4af/charts/opentelemetry-kube-stack/templates/exporters/kube-api-server/servicemonitor.yaml#L20-L38)。
- [vLLM production-stack 的 operator ServiceMonitor，提交 `7b1164d`](https://github.com/vllm-project/production-stack/blob/7b1164dbe71b1f02ede85152bdfc2f383376dc2d/operator/config/prometheus/monitor.yaml#L16)。

三者均引用 `/var/run/secrets/kubernetes.io/serviceaccount/token` 处的 `bearerTokenFile`；kube-stack 监控还引用 `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt` 处的 `tlsConfig.caFile`。

Secret 和 ConfigMap 引用可在不允许监控选择本地文件路径的情况下保留认证抓取。请遵循[限制监控文件访问](../guides/restrict-monitor-file-access.md)迁移任一 allocator 池并启用该限制。