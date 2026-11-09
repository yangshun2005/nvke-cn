# 自监控遥测 Sidecar

NVKE 遥测流水线（NKTP）中每个 Collector Pod 都运行一个 sidecar Collector。主 Collector 将其[内部遥测](https://opentelemetry.io/docs/collector/internal-telemetry/)发送到 `http://127.0.0.1:4319` 的 sidecar，sidecar 再通过 OTLP/gRPC 导出到 `nktp-watchdog-collector:4317` 的 watchdog。watchdog 的 sidecar 再导出到后端。

sidecar 缓冲并重试主 Collector 的遥测，并对其 trace 采样。

sidecar 的导出器是普通 Collector 导出器，因此若 Collector 之间启用了 mTLS，它可以重新加载轮换后的证书。

## 为何 sidecar 内联

对于集群中的其他服务，我们推荐由 OpenTelemetry Operator 注入 sidecar。但注入对 Operator 管理的 Collector 无效。当 Pod 已有名为 `otc-container` 的容器时，Operator 会[跳过注入](https://github.com/open-telemetry/opentelemetry-operator/blob/ca0b832a4cb071d144d374b453d9560b77a7d8d7/pkg/sidecar/pod.go#L79)，而它给所管理的每个 Collector 的主容器都用这个名字。因此 NKTP 在 Collector 的 Pod spec 中定义每个 sidecar。

## 启动与关闭

sidecar 是原生 sidecar，即带 `restartPolicy: Always` 的 init 容器。Kubernetes 在 sidecar 的启动探针成功后启动主 Collector。关闭时，主 Collector 先停止并刷新到 sidecar。若 sidecar 始终不健康，主 Collector 不会启动。

sidecar 的接收器仅监听 `127.0.0.1`。其健康检查使用端口 13134。

## sidecar 自身的遥测

每个 sidecar 将自身的指标和日志发送到自己的接收器，因此它们与主 Collector 的遥测共用其队列和导出器。这两条流描述同一对 Collector：sidecar 的队列和导出指标解释了主 Collector 数据中的缺口，因此它们一起缓冲、一起投递。

sidecar 不发出 trace（`level: none`）。其指标和日志已记录失败的导出和丢弃的导出，而它自身的 span 会使其需采样的 span 数量增加一倍以上。

## Trace 选择

满足以下任一策略时，sidecar 的 `tail_sampling` 处理器保留该 trace：

- `errors`：某个 span 的状态为 `ERROR`。
- `slow`：单个 span 超过 1 秒。不使用 trace 时长，因为导出批处理会给大多数 trace 增加最多 1 秒。

Collector 的自监控 span 发往其 Pod 内的 sidecar。sidecar 在 trace 的首个 span 到达 5 秒后做出决定。主 Collector 每秒刷新已完成的 span（`schedule_delay: 1000`），但仍在运行的操作尚未发送其 span。被保留 trace 的迟到 span 也会被保留。

当一次导出批处理多个请求时，其 span 会开启一个新 trace，并链接到各请求的 span。被保留的导出 trace 不会保留所链接的 trace。

若某 trace 已被拒绝，其迟到的 ERROR 或慢 span 可能被丢弃。若这影响排障，请增大受影响工作负载 `sidecar.otelcol.config` 中的 `processors.tail_sampling.decision_wait`，并同时评估 `num_traces` 和内存预算。窗口更长而 trace 容量不足，可能因评估前被驱逐而增加丢失。参见[上游调优指南](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/v0.162.0/processor/tailsamplingprocessor/README.md#monitoring-and-tuning)。

## 容量规划

| 所属 sidecar | CPU / 内存请求 | CPU / 内存限制 | 每信号队列 | 保留用于采样的 trace 数 |
| --- | --- | --- | --- | --- |
| Watchdog 和网关 | 20m / 64 MiB | 200m / 256 MiB | 16 MiB | 5,000 |
| 节点 collector、大目标 collector、集群事件读取器 | 10m / 32 MiB | 100m / 128 MiB | 4 MiB | 1,000 |

每个 sidecar 每信号一个队列。Collector 自身指标 30 分钟的数据远低于 1 MiB，因此每个队列大部分空间用于事故期间的错误日志和 span。

三个队列全满时，排队数据在内存中占其序列化大小的 1.2 到 1.4 倍。每个限制覆盖基础进程、三个满队列和保留用于采样的 trace，低于内存限制器在限制 55% 处的拒绝阈值。

队列在内存中，Pod 重启后不保留。自动扩缩器的利用率包含 sidecar 的请求。

## 仪表盘

每个 Collector 的默认仪表盘有一行"自监控遥测 sidecar"，显示 sidecar 对全部三种信号的队列使用、导出失败和拒绝项，以及采样决策、CPU 和内存。NKTP Health 仪表盘有单独的"自监控遥测 sidecar"行。sidecar 以自身的 `job` 上报，例如 `nktp-gateway-logs-sidecar`，任何查询都不会将 sidecar 与其 Collector 混在一起。

## 配置

每个工作负载有一个 `sidecar` 段：

- `<workload>.sidecar.kubernetes`：容器设置，如 `resources`、`env` 和探针。Chart 始终设置 `name`、`restartPolicy`、`image` 和 `args`，并在 `env` 列表省略时添加 `POD_NAME`，对 daemonset sidecar 还添加 `NODE_NAME`。
- `<workload>.sidecar.otelcol.config`：sidecar 的完整 Collector 配置，原样传入。
- `<workload>.sidecar.otelcol.image`：仅用于此 sidecar 的镜像。默认为 `otelcol_image`。

要将 NKTP 自身的遥测发送到其他后端，请设置 watchdog sidecar 的导出器端点，而非 watchdog 的 `service.telemetry`：

```yaml
watchdog:
  sidecar:
    otelcol:
      config:
        exporters:
          otlp:
            endpoint: <backend>:4317
            tls:
              insecure: false
```