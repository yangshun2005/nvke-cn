# 配置 kubelet 指标

每个 DaemonSet 采集器每 30 秒通过 HTTPS 抓取所在节点上的 kubelet。采集器直接连接节点主机 IP 的 10250 端口。这些抓取不使用 ServiceMonitor 或 Target Allocator。采集器使用投射的服务账号令牌进行认证，并具有只读的 `nodes/metrics` 权限。

抓取任务位于 `nktp/deploy/charts/nktp/values.yaml` 中 `daemonset.otelcol.config` 的 `prometheus/kubelet` receiver 内。默认有两个任务：

- `kubelet` 抓取 `/metrics`。
- `kubelet-cadvisor` 抓取 `/metrics/cadvisor`。其 `metric_relabel_configs` [复制自 kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/blob/4713c624771a48e99f16dec9eb55ec8204f06830/charts/kube-prometheus-stack/values.yaml#L1952)。这些规则会丢弃用处不大、始终为零、与 kube-state-metrics 重叠、来自主机网络接口或来自无 Pod 的 cgroup 的样本。

Chart 会完全按原样渲染此配置。要更改抓取，请更改配置。Helm 会整体替换 values 文件中的列表，因此请将 `nktp/deploy/charts/nktp/values.yaml` 中完整的 `scrape_configs` 列表复制到你的 values 文件，并修改你的副本。

## 更改抓取

- 要停止抓取 cAdvisor，请移除 `kubelet-cadvisor` 任务。
- 要更改 cAdvisor 过滤器，请编辑该任务的 `metric_relabel_configs`。
- 要抓取 `/metrics/probes` 或 `/metrics/resource`，请复制 `kubelet` 任务。然后为副本指定新的 `job_name`（例如 `kubelet-probes`），并将 `metrics_path` 和 `metrics_path` relabel 规则的 `replacement` 都设置为新路径。

## 验证 kubelet 证书

签署 kubelet 服务证书的证书颁发机构，以及这些证书中的主题备用名称，因 Kubernetes 发行版而异。因此，证书验证默认禁用。每个 kubelet 证书必须覆盖采集器使用的节点 IP。

要启用验证，请先在 NVKE 遥测流水线（NKTP）发布命名空间中创建一个包含 kubelet 服务 CA 的 ConfigMap。运行前请编辑下面的命名空间和本地证书路径：

```console
NKTP_NAMESPACE=nktp
KUBELET_CA_FILE=/path/to/kubelet-serving-ca.crt
kubectl create configmap kubelet-serving-ca \
  --namespace "$NKTP_NAMESPACE" \
  --from-file="ca.crt=$KUBELET_CA_FILE"
```

然后将其挂载到采集器中。将此卷添加到 `daemonset.kubernetes.volumes`，将此挂载添加到 `daemonset.kubernetes.volumeMounts`。这些是列表，因此请从 `nktp/deploy/charts/nktp/values.yaml` 复制默认条目，并将以下内容添加到其中：

```yaml
daemonset:
  kubernetes:
    volumes:
      # ...default volumes...
      - name: kubelet-serving-ca
        configMap:
          name: kubelet-serving-ca
          # If the CA is under a key other than ca.crt, map it to ca.crt:
          # items:
          #   - key: <key>
          #     path: ca.crt
    volumeMounts:
      # ...default volume mounts...
      - name: kubelet-serving-ca
        mountPath: /etc/nktp/kubelet-serving-ca
        readOnly: true
```

最后，在每个 kubelet 抓取任务中设置 `tls_config`：

```yaml
tls_config:
  ca_file: /etc/nktp/kubelet-serving-ca/ca.crt
  insecure_skip_verify: false
```

## 与 kube-prometheus-stack 配合使用

当 kube-prometheus-stack 与 NKTP 一起安装时，只有在 NKTP 抓取了你需要的 kubelet 端点且其消费方已迁移后，才禁用其 kubelet ServiceMonitor。这可以防止重复抓取 kubelet `/metrics`：

```yaml
kubelet:
  serviceMonitor:
    enabled: false
```

作为尽力而为的兜底措施，DaemonSet Target Allocator 会排除带有常规 `app` 标签 `kube-prometheus-stack-kubelet` 和 `prometheus-operator-kubelet` 的 ServiceMonitor。这可以防止它分配由其他系统管理的 kubelet 抓取。ServiceMonitor 的名称和标签可配置，因此此兜底措施无法找到所有 kubelet ServiceMonitor。