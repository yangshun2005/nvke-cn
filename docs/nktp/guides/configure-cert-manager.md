# 配置 cert-manager

选择 NVKE 遥测流水线（NKTP）获取 TLS 证书的方式：

- **使用 cert-manager（默认且推荐）。** cert-manager 为内部 Collector mTLS、Operator 的准入 Webhook 以及 Collector 到 Target Allocator 的 mTLS 签发并续期证书。这些功能默认启用，遵循部署 NKTP 的最佳实践。
- **不使用 cert-manager。** 自行提供并续期内部 Collector 和 Target Allocator mTLS 所需的证书。准入 Webhook 可使用自有证书或由 Helm 生成。安装前需替换全部三项集成。

## 使用 cert-manager 安装 NKTP

使用默认值安装 NKTP 前，请先按 [cert-manager 安装说明](https://cert-manager.io/docs/installation/) 安装 cert-manager。NKTP 不捆绑也不安装 cert-manager。以下各节介绍其三种用途及使用时的可用设置。

### Target Allocator

NKTP 显式启用由 cert-manager 支持的、每个 Collector 与其 Target Allocator 之间的 mTLS。Operator 为每一对组件创建 CA 以及服务端和客户端证书，详见 [使用 cert-manager（默认）](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/target-allocator/README.md#with-cert-manager-default)。

### 准入 Webhook

NKTP 默认使用已有的 cert-manager 安装来签发并续期 Operator 的准入 Webhook 证书。上游 Operator Chart 在 [TLS 证书要求](https://github.com/open-telemetry/opentelemetry-helm-charts/blob/opentelemetry-operator-0.126.0/charts/opentelemetry-operator/README.md#tls-certificate-requirement) 中记录了可用方案。

### 内部 mTLS

默认情况下，NKTP 使用 cert-manager 为每个 Collector 创建自签名 CA、Issuer 和证书，用于 Collector 到 Gateway 以及 Collector 到 Watchdog 的流量加密。`certManager.internalTLS.issuerRef` 必须引用一个 CA `Issuer` 或 `ClusterIssuer`，该 Issuer 需在生成的 TLS Secret 中填充 `ca.crt`，同时包含 `tls.crt` 和 `tls.key`。NKTP 挂载 `ca.crt` 以验证服务端和客户端证书。内部 mTLS 不支持 ACME Issuer：它们不会填充 `ca.crt`，导致所需的 Secret 卷挂载失败。参见 cert-manager 的 [证书 Secret 文档](https://cert-manager.io/docs/usage/certificate/#target-secret)。保留 `issuerRef: {}` 以使用 NKTP 生成的内置 CA 和 Issuer。

```yaml
certManager:
  internalTLS:
    enabled: true
    issuerRef: {}
```

集群事件读取器、DaemonSet 和大目标指标 Collector 各自使用客户端证书，通过 mTLS 向 Gateway 和 Watchdog 发送遥测数据。每个 Gateway 使用服务端和客户端证书。

Collector Sidecar 以及 DaemonSet 的 Target Allocator 和 Operator 日志导出器通过 OTLP/gRPC 在 4317 端口以 mTLS 向 Watchdog 发送数据。Target Allocator 导出器配置为通过明文 OTLP/gRPC 在 4320 端口发送自身指标。

对于 `nktp` 命名空间中的发布版本：

```bash
kubectl get issuers,certs -n nktp
```

示例输出：

```text
NAME                                                                  READY   AGE
issuer.cert-manager.io/nktp-internal-ca-issuer                        True    14s
issuer.cert-manager.io/nktp-internal-selfsigned-issuer                True    14s

NAME                                                                   READY   SECRET                                          AGE
certificate.cert-manager.io/nktp-cluster-events-client                 True    nktp-cluster-events-client-tls                  14s
certificate.cert-manager.io/nktp-daemonset-client                      True    nktp-daemonset-client-tls                       14s
certificate.cert-manager.io/nktp-gateway-logs-client                   True    nktp-gateway-logs-client-tls                    14s
certificate.cert-manager.io/nktp-gateway-logs-server                   True    nktp-gateway-logs-server-tls                    14s
certificate.cert-manager.io/nktp-gateway-metrics-client                True    nktp-gateway-metrics-client-tls                 14s
certificate.cert-manager.io/nktp-gateway-metrics-server                True    nktp-gateway-metrics-server-tls                 14s
certificate.cert-manager.io/nktp-gateway-traces-client                 True    nktp-gateway-traces-client-tls                  14s
certificate.cert-manager.io/nktp-gateway-traces-server                 True    nktp-gateway-traces-server-tls                  14s
certificate.cert-manager.io/nktp-internal-ca                           True    nktp-internal-ca                                14s
certificate.cert-manager.io/nktp-large-target-metrics-client           True    nktp-large-target-metrics-client-tls            14s
certificate.cert-manager.io/nktp-watchdog-server                       True    nktp-watchdog-server-tls                        14s
```

**注意：** Gateway 到后端**不**属于此功能范围。用户需单独配置该通信。

## 不使用 cert-manager 安装 NKTP

安装前需替换全部三项 cert-manager 集成。以下配置停止内部证书创建，并使用 Helm 生成的准入 Webhook 证书。同时需按下文所述提供内部 Collector 和 Target Allocator 证书；仅设置这些值并不足够。

```yaml
certManager:
  internalTLS:
    enabled: false
  targetAllocators:
    enabled: false
opentelemetry-kube-stack:
  opentelemetry-operator:
    admissionWebhooks:
      certManager:
        enabled: false
      autoGenerateCert:
        enabled: true
        recreate: false
```

### 用户提供的内部 Collector 证书

设置 `certManager.internalTLS.enabled: false`，并在发布命名空间中创建 Collector 和 Sidecar 的 `kubernetes.volumes` 值所引用的 TLS Secret。提供 `ca.crt`、`tls.crt` 和 `tls.key`，证书需对配置的服务和对端有效。保持默认的 Secret 名称、挂载和 Collector TLS 设置可保留内部 mTLS。若更改名称或路径，需同步更新卷引用、挂载和 Collector 配置。你需自行负责这些证书的签发和续期。

### 用户提供的 Target Allocator 证书

设置 `certManager.targetAllocators.enabled: false`，这会在 Watchdog、DaemonSet 和大目标 Allocator 上设置 `mtls.useCertManager: false`，并引用发布命名空间中已存在的证书。`mtls.enabled` 仍为每个 Allocator 的独立设置。在每个启用的 Allocator 下应用相同的 `tls` 块，位置为 `watchdog.kubernetes.targetAllocator`、`daemonset.kubernetes.targetAllocator` 和 `large_target_metrics.kubernetes.targetAllocator`。例如：

```yaml
daemonset:
  kubernetes:
    targetAllocator:
      mtls:
        enabled: true
        tls:
          certificateAuthorityCertificate:
            configMap:
              name: target-allocator-ca
          serverCertificate:
            certificateSecret:
              name: target-allocator-server
            keySecret:
              name: target-allocator-server
          clientCertificate:
            certificateSecret:
              name: target-allocator-client
            keySecret:
              name: target-allocator-client
```

关于可接受的 Secret 和 ConfigMap 引用、默认键及证书轮换行为，参见 Operator 的 [用户提供证书文档](https://github.com/open-telemetry/opentelemetry-operator/blob/main/docs/target-allocator/README.md#without-cert-manager-user-provided-certificates)。

### 不使用 cert-manager 的 Webhook 证书

上述配置让 Helm 生成自签名 Webhook 证书。此配置在升级时复用该证书；证书默认 365 天后过期，且不会自动续期。

若要改为提供自有 Webhook 证书，还需设置 `admissionWebhooks.autoGenerateCert.enabled: false`，并在 `opentelemetry-kube-stack.opentelemetry-operator` 下提供上游 Chart 的 `admissionWebhooks.certFile`、`keyFile` 和 `caFile` 值。参见上游 Chart 的 [TLS 证书要求](https://github.com/open-telemetry/opentelemetry-helm-charts/blob/opentelemetry-operator-0.126.0/charts/opentelemetry-operator/README.md#tls-certificate-requirement)。

### 将内部流量切换为明文

仅在配合 [明文示例](https://github.com/dsx-ai-factory/nvke/blob/main/docs/nktp/examples/disable-internal-mtls.values.yaml) 中的配置和挂载覆盖时，才设置 `certManager.internalTLS.enabled: false`。此标志仅停止证书创建，不改变 Collector 配置。该示例仅更改内部流量；还需应用上述 Webhook 和 Target Allocator 覆盖，才能移除 cert-manager 依赖。