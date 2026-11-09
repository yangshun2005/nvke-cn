# 配置持久化导出器队列

默认情况下，watchdog 和网关将遥测数据排队在内存中，因此重启的 Collector 会丢失已排队的数据。NVKE 遥测流水线（NKTP）不配置持久化队列。本指南介绍如何通过 chart 的 values 设置 OpenTelemetry Collector 配置，将每个导出器的[队列](https://github.com/open-telemetry/opentelemetry-collector/blob/v0.162.0/exporter/exporterhelper/README.md#persistent-queue)保存在 PersistentVolume 上，使用 [`file_storage` 扩展](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/v0.162.0/extension/storage/filestorage)。默认 Collector 镜像已包含该扩展。

队列仅在 Collector 无法连接后端时保存数据。将其存储在磁盘上不会改变重试行为：在 `retry_on_failure.max_elapsed_time`（默认 30 分钟）之后仍然失败的批次会被丢弃。在 `otlp` 导出器上提高该值，以便在更长时间的中断期间保留数据。

## 前提条件

一个能提供卷的 StorageClass。检查集群是否有默认的 StorageClass：

```console
kubectl get storageclass
```

没有默认 StorageClass 时，卷声明会保持 `Pending`，Pod 也会如此。在每个声明模板中设置 `storageClassName`，以使用非默认的 StorageClass。

## 配置队列

从仓库根目录，将下面的发布名称和命名空间改为与你的安装匹配，然后使用 [values 示例](https://github.com/dsx-ai-factory/nvke/blob/main/docs/nktp/examples/persistent-queue.values.yaml)安装或升级，该示例配置了 watchdog 和全部三个网关：

```console
NKTP_RELEASE=nktp
NKTP_NAMESPACE=nktp
make nktp-chart-deps
helm upgrade --install "$NKTP_RELEASE" ./nktp/deploy/charts/nktp --namespace "$NKTP_NAMESPACE" \
  --values docs/nktp/examples/persistent-queue.values.yaml
```

对每个 Collector，它会添加一个声明模板、挂载卷、使其对 Collector 用户可写，并将导出器的队列指向该卷。对于 watchdog：

```yaml
watchdog:
  kubernetes:
    podSecurityContext:
      fsGroup: 10001
      fsGroupChangePolicy: OnRootMismatch
    volumeClaimTemplates:
      - metadata:
          name: queue
        spec:
          accessModes: [ReadWriteOnce]
          # storageClassName: <class>
          resources:
            requests:
              storage: 3Gi
    persistentVolumeClaimRetentionPolicy:
      whenDeleted: Retain
      whenScaled: Retain
    volumeMounts:
      - name: internal-tls-server
        mountPath: /etc/nktp/internal-tls/server
        readOnly: true
      - name: queue
        mountPath: /var/lib/otelcol/queue
  otelcol:
    config:
      extensions:
        file_storage/queue:
          directory: /var/lib/otelcol/queue
      exporters:
        otlp:
          sending_queue:
            storage: file_storage/queue
            queue_size: 536870912
      service:
        extensions: [health_check, file_storage/queue]
```

卷挂载和 `service.extensions` 是列表，Helm 会替换它们，因此示例重新声明了 chart 的内部 TLS 挂载和 `health_check`。与其他示例（例如 [安全网关导出](secure-gateway-export.md)中的示例）组合时，请手动合并列表。Helm 会合并 `sending_queue`，因此其其他默认设置保持不变。

## 设置卷大小

`queue_size` 计算序列化字节数并限制每个队列的大小。chart 的默认值为 watchdog 每个信号 8MiB，每个网关约 0.48 GiB。示例为每个队列设置 512MiB，即分配给每个队列空间的一半。

保持 `queue_size` 不超过每个队列空间的一半。一个满的 512MiB 队列在磁盘上占用 530MiB，在队列排空过程中文件增长到 635MiB。之后文件不会缩小。当卷满时，队列写入会失败并报 `no space left on device`。在 `queue_size` 等于卷大小的测试中，重启的 Collector 花了约六分钟排空，丢失了正在处理的批次，并且未能删除一些已发送的条目，这些条目可能在后续重启后再次发送。每个网关有一个队列并声明 1Gi。watchdog 为其三个信号各保留一个独立队列，并声明 3Gi。同时提高 `queue_size` 和声明大小，以应对更长时间的后端中断。

至少请求提供商的最小卷大小。OCI 块卷的最小值为 50GB。

每个运行过的副本都保留自己的声明。watchdog 在 2 到 16 个副本之间自动扩缩，每个网关在 2 到 128 个之间，因此在最大值时最多有 400 个卷，按 OCI 最小值计算为 20TB。降低 `autoscaler.maxReplicas` 以限制数量。使用 `whenScaled: Retain` 时，已移除副本上排队的遥测数据会在该副本再次扩容时发送。使用 `Delete` 时，数据会被丢弃。

## 更改声明模板

Kubernetes 不允许更改 StatefulSet 的卷声明模板。当它们被添加或更改时，OpenTelemetry Operator 会删除 StatefulSet 并重新创建，因此该 Collector 的每个副本会同时重启。发送方在不可用期间会重试。请在安静时段进行此更改。`whenDeleted: Retain` 会在此过程中保留声明，新副本会按原样复用它们。使用 `Delete` 时，排队的数据会被删除。

## 删除声明

保留的声明在卸载 NKTP 后仍然存在。编辑下面的命名空间，然后在不再需要时删除它们：

```console
NKTP_NAMESPACE=nktp
for collector in watchdog gateway-logs gateway-metrics gateway-traces; do
  kubectl delete pvc --namespace "$NKTP_NAMESPACE" \
    --selector "app.kubernetes.io/instance=$NKTP_NAMESPACE.nktp-$collector"
done
```

## 验证

每个 Collector 的 `otelcol_exporter_queue_size` 指标显示排队的数据。每个副本都有一个以其 Pod 命名的绑定声明。编辑下面的命名空间，然后列出它们：

```console
NKTP_NAMESPACE=nktp
kubectl get pvc --namespace "$NKTP_NAMESPACE" | grep -E "nktp-(watchdog|gateway)-"
```