# 启用自动埋点

NVKE 遥测流水线（NKTP）安装的 OpenTelemetry Operator 可在创建 Pod 时，向 Java、Node.js、Python 和 .NET 容器注入 OpenTelemetry 代理。代理无需改动服务代码或镜像，即可生成该服务的追踪和指标。

`Instrumentation` 资源用于配置代理，包括遥测数据的发送位置。服务所有者通过为 Pod 模板添加注解，指定要使用的 `Instrumentation` 来启用。参见[使用自动埋点](send-telemetry.md#use-auto-instrumentation)。

## 提供 Instrumentation

设置以下值，在发布命名空间中创建名为 `default` 的共享 `Instrumentation`。在端点中，只需将点号后的命名空间（`.nktp`）替换为发布命名空间，保留 `nktp-daemonset-collector` 前缀。

```yaml
opentelemetry-kube-stack:
  instrumentation:
    enabled: true
    name: default
    exporter:
      endpoint: http://nktp-daemonset-collector.nktp:4318
    nodejs:
      env:
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: http://nktp-daemonset-collector.nktp:4317
```

服务所有者以 `nktp/default` 引用它。每个代理将数据发送到其所在节点上的节点采集器。Java、Python 和 .NET 代理通过 HTTP 使用 OTLP，端口为 4318。Node.js 代理通过 gRPC 使用 OTLP，因此其端点端口为 4317。

`opentelemetry-kube-stack.instrumentation` 下的其他值用于设置 `Instrumentation` 的其余部分，例如采样器、传播器和代理镜像。相关设置参见[注入自动埋点](https://opentelemetry.io/docs/platforms/kubernetes/operator/automatic/)。