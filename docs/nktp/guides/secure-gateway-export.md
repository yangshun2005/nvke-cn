# 安全网关导出

NVKE 遥测流水线（NKTP）默认通过明文 OTLP/gRPC 导出到其内置的 LGTM 后端，地址为 `lgtm:4317`。LGTM 已同时启用 [gRPC 和 HTTP 接入](https://github.com/grafana/docker-otel-lgtm/blob/v0.38.0/docker/otelcol-config.yaml)。该后端仅用于本地开发。向生产后端发送日志、指标和追踪前，请先配置 TLS。

为每种信号选择后端支持的协议和端点。Collector 的 [`otlp` 导出器](https://github.com/open-telemetry/opentelemetry-collector/blob/v0.162.0/exporter/otlpexporter/README.md)使用 gRPC；[`otlp_http`](https://github.com/open-telemetry/opentelemetry-collector/blob/v0.162.0/exporter/otlphttpexporter/README.md)使用 HTTP。仅改端口不会改变协议。例如，直接写入 [Prometheus](https://prometheus.io/docs/guides/opentelemetry/) 或 [Loki](https://grafana.com/docs/loki/latest/send-data/otel/) 需要 OTLP/HTTP。LGTM 通过其内置 Collector 接受 gRPC，并在内部将指标和日志转发到这些 HTTP 端点。

本指南为三个网关 Collector 配置基于 gRPC 的 mTLS。[示例](https://github.com/dsx-ai-factory/nvke/blob/main/docs/nktp/examples/gateway-mtls.values.yaml)展示了覆盖配置。进入网关的连接所用的接收端 TLS 需单独配置。

## 使用现有证书配置 mTLS

前置条件，你需要已具备：
- 用于验证后端服务器证书的 CA 证书；
- 由后端信任的 CA 签发的客户端证书；以及
- 客户端证书对应的私钥。

1. 在 Helm 发布版本命名空间中创建包含上述三个文件的 Secret。运行前请修改下面的命名空间和本地证书路径：

```bash
NKTP_NAMESPACE=nktp
BACKEND_CA_FILE=/path/to/backend-server-ca.crt
BACKEND_CLIENT_CERT_FILE=/path/to/client.crt
BACKEND_CLIENT_KEY_FILE=/path/to/client.key
kubectl create secret generic backend-tls \
  --namespace "$NKTP_NAMESPACE" \
  --from-file="ca.crt=$BACKEND_CA_FILE" \
  --from-file="tls.crt=$BACKEND_CLIENT_CERT_FILE" \
  --from-file="tls.key=$BACKEND_CLIENT_KEY_FILE"
```

2. 将 Secret 挂载到每个网关，同时保留其内部 TLS 卷和挂载。每个网关需要其内部服务器证书用于入站，其 sidecar 需要内部客户端证书以导出到 watchdog。后端凭据应放在主网关 Collector 上，而非其 sidecar。

```yaml
gateway_logs:
  kubernetes:
    podSecurityContext:
      fsGroup: 10001
    volumes:
      - name: internal-tls-server
        secret:
          secretName: nktp-gateway-logs-server-tls
          items:
            - {key: tls.crt, path: tls.crt}
            - {key: tls.key, path: tls.key}
            - {key: ca.crt, path: ca.crt}
      - name: internal-tls-client
        secret:
          secretName: nktp-gateway-logs-client-tls
          items:
            - {key: tls.crt, path: tls.crt}
            - {key: tls.key, path: tls.key}
            - {key: ca.crt, path: ca.crt}
      - name: backend-tls
        projected:
          defaultMode: 0440
          sources:
            - secret:
                name: backend-tls
                items:
                  - key: ca.crt
                    path: ca.crt
                  - key: tls.crt
                    path: tls.crt
                  - key: tls.key
                    path: tls.key
    volumeMounts:
      - name: internal-tls-server
        mountPath: /etc/nktp/internal-tls/server
        readOnly: true
      - name: backend-tls
        mountPath: /etc/nktp/backend-tls
        readOnly: true
```

> **注意：** `fsGroup: 10001` 与 NKTP 默认 Collector 镜像使用的组一致。它让非 root 的 Collector 通过 `0440` 中的组读权限读取 root 拥有的 Secret 文件。若使用自定义 Collector 镜像，请将 `fsGroup` 设为该镜像可用的组。或者省略 `defaultMode` 和 `fsGroup`，使用 Kubernetes 默认的 Secret 模式（`0644`）。

3. 配置导出器使用挂载的证书。

```yaml
gateway_logs:
  ...
  otelcol:
    config:
      exporters:
        otlp:
          endpoint: telemetry.example.com:4317
          tls:
            insecure: false
            ca_file: /etc/nktp/backend-tls/ca.crt
            cert_file: /etc/nktp/backend-tls/tls.crt
            key_file: /etc/nktp/backend-tls/tls.key
            reload_interval: 1m
```

将卷、挂载和导出器设置应用到 `gateway_logs`、`gateway_metrics` 和 `gateway_traces`。使用各网关自己的内部 Secret：`nktp-gateway-<signal>-server-tls` 和 `nktp-gateway-<signal>-client-tls`，其中 `<signal>` 为 `logs`、`metrics` 或 `traces`。保持上述卷名和挂载路径不变。完整的 [values 示例](https://github.com/dsx-ai-factory/nvke/blob/main/docs/nktp/examples/gateway-mtls.values.yaml)配置了全部三个网关。

显式设置 `insecure: false`：开发默认值会禁用 TLS，而 Helm 在合并覆盖配置时会保留嵌套默认值。若要将 NKTP 自身的遥测发送到该后端，还需配置 watchdog 及其 sidecar 的后端导出器。

对于 HTTP 后端，用 `otlp: null` 移除默认 gRPC 导出器，添加 `otlp_http` 并使用 `https://` 端点及相同的证书设置，然后在每个受影响的流水线 `exporters` 列表中将 `otlp` 替换为 `otlp_http`。按需保留默认导出器的队列和重试设置。

对于使用私有 CA 但不要求客户端认证的服务器认证 TLS，仅挂载 `ca.crt` 并仅配置 `ca_file`。

若出现故障，检查网关日志中的 `x509`、`tls`、`handshake`、`unauthorized` 或 `forbidden`。常见原因包括缺少中间 CA、端点主机名不在服务器证书中、客户端证书已过期，或后端不信任客户端签发者。

## 使用 cert-manager 自动化客户端证书

[cert-manager](https://cert-manager.io/docs/configuration/) 不随 NKTP 提供。当现有 `Issuer` 或 `ClusterIssuer` 代表后端信任的 CA 时，它可签发和续期网关客户端证书。创建签发者本身并不会让后端信任该 CA。

1. 确保存在一个 `ClusterIssuer` 或 `Issuer` CR，对应后端信任的证书颁发机构。详情请参阅 [cert-manager](https://cert-manager.io/docs/configuration/) 文档。

2. 在 `nktp` 命名空间中创建 `Certificate` CR。将 `metadata.namespace` 改为你的 Helm 发布版本命名空间。

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: nktp-gateway-client
  namespace: nktp
spec:
  secretName: backend-tls
  issuerRef:
    kind: ClusterIssuer
    name: backend-client-ca
  usages:
    - client auth
  commonName: example.com # 占位符：后端期望的客户端身份。
  privateKey:
    rotationPolicy: Always
```

将 `commonName` 替换为后端映射到客户端的身份，或改为设置其读取的 `dnsNames` 或 `uris`。

cert-manager 在 `backend-tls` 中维护 `tls.crt` 和 `tls.key`。仅当签发者提供时它才写入 `ca.crt`，且该 CA 是客户端证书的 CA，而非后端服务器证书的 CA。

3. 创建一个单独的 Secret，包含签署后端服务器证书的 CA。运行前请修改下面的命名空间和本地证书路径：

```bash
NKTP_NAMESPACE=nktp
BACKEND_CA_FILE=/path/to/backend-server-ca.crt
kubectl create secret generic backend-ca \
  --namespace "$NKTP_NAMESPACE" \
  --from-file="ca.crt=$BACKEND_CA_FILE"
```

4. 在每个网关中，将上一节的 `backend-tls` 卷替换为同时投射两个 Secret 的卷。保留其他卷和挂载；挂载路径和文件名保持不变：

```yaml
      - name: backend-tls
        projected:
          defaultMode: 0440
          sources:
            - secret:
                name: backend-tls
                items:
                  - key: tls.crt
                    path: tls.crt
                  - key: tls.key
                    path: tls.key
            - secret:
                name: backend-ca
                items:
                  - key: ca.crt
                    path: ca.crt
```

在导出器配置中保留 `ca_file`、`cert_file`、`key_file` 和 `reload_interval`。