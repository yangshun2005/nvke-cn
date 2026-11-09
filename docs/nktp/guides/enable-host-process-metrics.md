# 启用主机进程指标

节点 Collector 默认不采集按进程划分的指标。Node exporter 提供主机指标，kubelet cAdvisor 抓取提供按容器划分的资源用量。本指南在节点 Collector 上启用 `hostmetrics` receiver 的 `process` scraper，采集每个节点上各进程的 CPU、内存和磁盘 I/O。

某些环境需要额外权限，scraper 才能完整读取主机进程。常见问题见[故障排查](#故障排查)。[values 示例](https://github.com/dsx-ai-factory/nvke/blob/main/docs/nktp/examples/host-process-metrics.values.yaml)启用了本指南中的所有权限、挂载和静默选项。

## 启用 scraper

节点 Collector 已把主机的 `/proc` 挂载到 `/hostfs/proc`。在节点 Collector 中添加 receiver 及其独立流水线：

```yaml
daemonset:
  otelcol:
    config:
      receivers:
        hostmetrics:
          root_path: /hostfs
          collection_interval: 1m
          scrapers:
            process:
              resource_attributes:
                process.command_line:
                  enabled: false
      service:
        pipelines:
          metrics/process:
            receivers: [hostmetrics]
            processors: [memory_limiter, resource/node, redaction/basic_v1]
            exporters: [otlp/metrics]
```

使用独立流水线可避免复制默认 `metrics` 流水线的 `receivers` 列表——Helm 会替换而非合并该列表。保持 `process.command_line` 禁用：命令行可能包含凭据。

## 安全

节点 Collector 以 root 身份在每个节点上运行，并挂载了主机的 `/proc`。[故障排查](#故障排查)中的权限会扩大被攻陷 Collector 的可达范围：

- `SYS_PTRACE` 使其能读取以 root 运行的主机进程（如 kubelet 和容器运行时）的环境变量和内存（`/proc/<pid>/environ`、`/proc/<pid>/mem`）。
- `DAC_READ_SEARCH` 将上述能力扩展到所有用户的进程，并绕过所有已挂载主机路径的读权限。它还启用 `open_by_handle_at(2)`（默认 seccomp 配置随后允许该调用），可打开 Collector 有挂载的主机文件系统上的任意文件。
- `Unconfined` AppArmor 配置，或 SELinux 下的 `spc_t`，会为整个容器移除该层防护。

只授予所需权限。仅 `SYS_PTRACE` 即可覆盖以 root 运行的进程的可执行文件路径和磁盘 I/O。

## 基数

每个进程是独立的 resource，以 `process.pid` 为键，默认指标为每个进程产生 7 条序列。内核线程也计为进程，内核在每个 CPU 上运行多个内核线程。

每个新进程都会开启新序列，因此短生命周期进程会增加序列变动。scraper 的 CPU 用量也随进程数增长。

Prometheus OTLP 摄取不会把服务标识之外的 resource 属性转为序列标签，除非显式提升，因此不同进程的序列会冲突。在后端提升 `process.*` 属性，或在 `metrics/process` 流水线中把标识进程的属性复制到数据点属性：

```yaml
daemonset:
  otelcol:
    config:
      processors:
        transform/process:
          metric_statements:
            - context: datapoint
              statements:
                - set(attributes["process.pid"], resource.attributes["process.pid"])
                - set(attributes["process.executable.name"], resource.attributes["process.executable.name"])
      service:
        pipelines:
          metrics/process:
            processors: [memory_limiter, resource/node, transform/process, redaction/basic_v1]
```

减少数据量：

- 用 `include` 或 `exclude` 按 `process.executable.name` 过滤：

  ```yaml
  daemonset:
    otelcol:
      config:
        receivers:
          hostmetrics:
            scrapers:
              process:
                include:
                  match_type: regexp
                  names: ["^kubelet$", "^containerd", "^crio$"]
  ```

- 在 `metrics` 下禁用不需要的指标。
- 设置 `scrape_process_delay` 跳过短生命周期进程。
- 提高 `collection_interval` 以减少发送的样本数。序列数不变。

## 故障排查

scraper 输出能读取的数据，并在每次采集时为其余部分记录一行 `Error scraping metrics`，计入 `otelcol_scraper_errored_metric_points`。CPU 时间和内存无需额外权限。编辑下方命名空间，然后检查错误：

```console
NKTP_NAMESPACE=nktp
kubectl logs --namespace "$NKTP_NAMESPACE" daemonset/nktp-daemonset-collector \
  --container otc-container | grep "Error scraping metrics"
```

授予权限前请阅读[安全](#安全)，或参见[静默错误](#静默错误)以接受缺失数据。

### 读取可执行文件路径时权限被拒绝

```text
error reading process executable for pid <pid>: readlink /hostfs/proc/<pid>/exe: permission denied
```

这些进程的 `process.executable.path` 为空。读取 `/proc/<pid>/exe` 需要通过 ptrace 读检查。Collector 以 root 运行但能力已被丢弃，对其他用户的进程以及持有其缺失能力的进程（如许多主机守护进程）无法通过该检查。添加 `SYS_PTRACE`，同时保留 journald receiver 所需的 `SYS_CHROOT`：

```yaml
daemonset:
  kubernetes:
    securityContext:
      capabilities:
        add: [SYS_CHROOT, SYS_PTRACE]
```

若报 `no such file or directory` 而非 `permission denied`，来自内核线程，它们没有可执行文件。

### 有 SYS_PTRACE 仍权限被拒绝

在启用 AppArmor 的节点上，运行时的默认配置即使有 `SYS_PTRACE` 也可能拒绝这些读取。例如 containerd 和 Docker 的配置只允许同一配置内的 ptrace 读。节点内核或审计日志会显示 `apparmor="DENIED" operation="ptrace"`。若如此，以 unconfined 运行 Collector：

```yaml
daemonset:
  kubernetes:
    securityContext:
      appArmorProfile:
        type: Unconfined
```

### 读取磁盘 I/O 时权限被拒绝

```text
error reading disk usage for process "<name>" (pid <pid>): open /hostfs/proc/<pid>/io: permission denied
```

这些进程的 `process.disk.io` 缺失。读取 `/proc/<pid>/io` 需要 `SYS_PTRACE`。对于非 root 运行的进程，还需要 `DAC_READ_SEARCH`，这是这些权限中最宽的一个：

```yaml
daemonset:
  kubernetes:
    securityContext:
      capabilities:
        add: [SYS_CHROOT, SYS_PTRACE, DAC_READ_SEARCH]
```

### 进程所有者查询失败

```text
error reading username for process "<name>" (pid <pid>): open /etc/passwd: no such file or directory
```

`process.owner` 为空。scraper 通过 Collector 容器自身的 `/etc/passwd` 解析用户 ID，默认镜像没有该文件。将节点的 `/etc/passwd` 只读挂载到 `/etc/passwd`。volumes 和 mounts 是列表，因此从 `nktp/deploy/charts/nktp/values.yaml` 复制默认值并添加：

```yaml
daemonset:
  kubernetes:
    volumes:
      # ...default volumes...
      - name: host-passwd
        hostPath:
          path: /etc/passwd
          type: File
    volumeMounts:
      # ...default volume mounts...
      - name: host-passwd
        mountPath: /etc/passwd
        readOnly: true
```

之后添加到节点的用户需在 Collector 重启后才能解析。容器内进程的用户 ID 定义在其自身镜像中。这些 ID 要么如下失败，要么解析为无关的节点用户：

```text
error reading username for process "<name>" (pid <pid>): user: unknown userid <uid>
```

## 静默错误

每个选项阻止一类错误被记录。数据仍然缺失。

| 选项 | 错误来源 | 适用场景 |
| --- | --- | --- |
| `mute_process_exe_error` | `/proc/<pid>/exe` | 内核线程。也会隐藏缺失 `SYS_PTRACE` 和 AppArmor 拒绝。 |
| `mute_process_io_error` | `/proc/<pid>/io` | 未使用 `SYS_PTRACE` 运行，或对非 root 进程未使用 `DAC_READ_SEARCH`。 |
| `mute_process_user_error` | 所有者查询 | 未挂载 `/etc/passwd`，或容器用户 ID。 |

```yaml
daemonset:
  otelcol:
    config:
      receivers:
        hostmetrics:
          scrapers:
            process:
              mute_process_exe_error: true
              mute_process_io_error: true
              mute_process_user_error: true
```

或禁用无法读取的指标或属性。即使禁用 `process.owner`，scraper 仍会查询所有者，因此保留 `mute_process_user_error`：

```yaml
daemonset:
  otelcol:
    config:
      receivers:
        hostmetrics:
          scrapers:
            process:
              mute_process_user_error: true
              metrics:
                process.disk.io:
                  enabled: false
              resource_attributes:
                process.owner:
                  enabled: false
```