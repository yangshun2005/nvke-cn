# 资源规格

默认预算按 Pod 计算。在 `<component>.kubernetes.resources` 下配置采集器资源，在 `<component>.kubernetes.targetAllocator.resources` 下配置分配器资源。

| 组件 | CPU / 内存请求 | CPU / 内存限制 |
| --- | --- | --- |
| 节点采集器 | 500m / 1 GiB | 2 / 2 GiB |
| 大目标采集器 | 500m / 2 GiB | 4 / 12 GiB |
| 指标网关 | 250m / 512 MiB | 6 / 4 GiB |
| 日志和追踪网关 | 250m / 512 MiB | 4 / 4 GiB |
| 集群事件读取器 | 250m / 256 MiB | 1 / 512 MiB |
| Watchdog | 100m / 256 MiB | 4 / 1 GiB |
| 节点 Target Allocator | 250m / 2 GiB | 2 / 4 GiB |
| 大目标和 watchdog Target Allocator | 250m / 1 GiB | 1 / 2 GiB |
| 自监控 sidecar、watchdog 和网关 | 20m / 64 MiB | 200m / 256 MiB |
| 自监控 sidecar、其他采集器 | 10m / 32 MiB | 100m / 128 MiB |

节点采集器使用 `768MiB` 的 `GOMEMLIMIT`。网关使用 1.5 GiB，大目标采集器使用 7 GiB，watchdog 使用 192 MiB。运行时内存目标和队列需与容器限制一起调整。
节点采集器队列每种信号保留 64 MiB，包含在途载荷。

HPA 目标相对于请求值。提高限制不会提高扩缩容阈值。为节点容量预留增长和故障恢复空间，并启动足够的抓取副本以吸收较大的初始负载。这些预算不保证特定的序列数量。在负载下验证分配器发现和重新分配，尤其是在重启之后。

关于大抓取响应的池容量规划，参见[大目标指标](../components/large-target-metrics.md#sizing-large-scrapes)；关于组件饱和时队列和内存限制器的行为，参见[背压](../architecture/backpressure.md)。关于 sidecar 规格的选取方式，参见[自监控 sidecar](../architecture/self-telemetry.md#sizing)。