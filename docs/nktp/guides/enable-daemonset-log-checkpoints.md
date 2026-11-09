# 为节点 Collector 启用日志检查点

节点 Collector 的 `filelog` 和 `journald` 接收器从每个文件和 journal 的末尾开始读取，并将读取位置保存在内存中。当 Collector 重启或其 Pod 被替换时（例如在 chart 升级期间），停机期间写入的行会被跳过。本指南使用 `file_storage` 扩展（默认 Collector 镜像已包含）将读取位置保存到每个节点上的目录中，使替换后的 Collector 从上一个停止的位置继续读取。这利用了接收器自身的[偏移量跟踪](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver#offset-tracking)和[游标跟踪](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/journaldreceiver#cursor-tracking)功能。

[values 示例](https://github.com/dsx-ai-factory/nvke/blob/main/docs/nktp/examples/daemonset-log-checkpoints.values.yaml)包含以下所有配置。

## 启用检查点

按照[入门指南](../getting-started.md#install)中的说明配置 Helm 仓库。在仓库根目录下，构建 chart 依赖，然后使用 values 示例进行安装或升级。将下方的 release 名称和命名空间修改为与你的安装一致：

```console
NKTP_RELEASE=nktp
NKTP_NAMESPACE=nktp
make nktp-chart-deps
helm upgrade --install "$NKTP_RELEASE" ./nktp/deploy/charts/nktp --namespace "$NKTP_NAMESPACE" \
  --values docs/nktp/examples/daemonset-log-checkpoints.values.yaml
```

该配置将主机目录挂载到 Collector 中：

```yaml
daemonset:
  kubernetes:
    volumes:
      # ...default volumes...
      - name: log-checkpoints
        hostPath:
          path: /var/lib/nktp/log-checkpoints
          type: DirectoryOrCreate
    volumeMounts:
      # ...default volume mounts...
      - name: log-checkpoints
        mountPath: /var/lib/otelcol/log-checkpoints
```

Volumes 和 mounts 是列表，Helm 会整体替换，因此示例中重新列出了 `nktp/deploy/charts/nktp/values.yaml` 中 chart 的默认值。与其他示例（例如[启用主机进程指标](enable-host-process-metrics.md)）组合使用时，需手动合并这些列表。

然后添加扩展，并为每个日志接收器设置 `storage`：

```yaml
daemonset:
  otelcol:
    config:
      extensions:
        file_storage/log-checkpoints:
          directory: /var/lib/otelcol/log-checkpoints
          create_directory: true
          timeout: 1s
      receivers:
        filelog:
          storage: file_storage/log-checkpoints
        filelog/host-auth:
          storage: file_storage/log-checkpoints
        filelog/host-audit:
          storage: file_storage/log-checkpoints
        filelog/kube-api-audit:
          storage: file_storage/log-checkpoints
        filelog/targetallocator:
          storage: file_storage/log-checkpoints
        filelog/operator:
          storage: file_storage/log-checkpoints
        journald:
          storage: file_storage/log-checkpoints
      service:
        extensions: [health_check, file_storage/log-checkpoints]
```

`service.extensions` 出于同样的原因重新列出了 `health_check`。为你添加的任何日志接收器都加上 `storage`。每个接收器的检查点以其名称存储，因此重命名接收器会使其从 `start_at` 重新开始。

## 保留内容

主机目录在 Collector 重启、Pod 替换和节点重启后仍然存在。`start_at` 仍适用于没有检查点的文件和 journal，例如新的 Pod 日志文件。

当 kubelet 在 Collector 读取之前轮转日志文件，或节点本身被替换时，日志行仍会丢失。

## 安全与清理

节点 Collector 以 root 运行，因此可以写入主机目录。请将该路径专用于 NVKE 遥测流水线（NKTP）。卸载 NKTP 后，该目录仍保留在每个节点上；不再需要时请从节点上删除。

每个节点有一个目录。在同一节点上运行多个 NKTP release 时，请为每个 release 使用不同的 `hostPath.path`。

## 验证

将下方的节点名称设置为运行 Collector 的节点，然后列出其检查点文件：

```console
NKTP_NODE=your-node-name
kubectl debug "node/$NKTP_NODE" -it --image=docker.io/library/busybox:1.37 \
  -- ls -la /host/var/lib/nktp/log-checkpoints
```

每个日志接收器对应一个文件，以接收器名称命名。