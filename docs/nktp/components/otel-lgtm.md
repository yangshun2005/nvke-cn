# otel-lgtm

[grafana/otel-lgtm](https://github.com/grafana/docker-otel-lgtm) 是一体化可观测性后端（Grafana、Loki、Tempo、Prometheus），支持 OTLP 接入。
NVKE 遥测流水线（NKTP）默认部署它，安装后即可查看遥测数据。
Prometheus 可通过 `lgtm` Service 的 9090 端口访问，例如安装到 `nktp` 命名空间时使用
`http://lgtm.nktp.svc:9090`。Grafana 通过 `lgtm` Service 的 3000 端口提供，匿名管理员访问（无需登录）。

> [!WARNING]
> 此容器仅供试用。它无法支撑大规模负载，生产环境必须禁用：
>
> ```yaml
> otel-lgtm:
>   enabled: false
> ```

## 仪表盘

默认的 **NKTP Health** 仪表盘展示工作负载就绪状态、流水线故障、
队列和遥测流量。它包含组件详情、诊断日志、
内部追踪以及组件仪表盘链接。工作负载就绪状态通过指标网关使用
kube-state-metrics。Collector 和 Target Allocator 自身指标使用 watchdog，不依赖指标网关。Target
Allocator 计数按副本分别统计。禁用 LGTM 会移除其仪表盘。Target Allocator 指标
仍会流向 watchdog 中配置的后端，该后端必须支持增量指标。

导出和队列准入延迟通过 Tempo 的 TraceQL
指标支持使用内部 span。容器资源表显示每个组件最多
三个容器的 requests 和 limits。仪表盘还展示分配器分配
时长和 node-exporter 诊断信息。

每个 Collector 也有自己的仪表盘，布局相同：其流水线按信号汇总、
专属行、队列和资源，以及自身遥测 sidecar。显示副本的面板显示最大的 20 个。
**Collectors** 下拉菜单可在它们之间切换。大目标仪表盘
包含 kube-state-metrics 抓取面板：样本数、样本数增长、
时长、watch 错误和分片覆盖率。

参见[仪表盘 README](https://github.com/dsx-ai-factory/nvke/blob/main/nktp/deploy/charts/nktp/dashboards/README.md) 了解所有
仪表盘以及如何导入到你自己的 Grafana。

## 配置

在 `otel-lgtm` 下配置内置后端：

| 值 | 默认值 | 用途 |
| --- | --- | --- |
| `resources` | `{}` | 容器资源 requests 和 limits。 |
| `nodeSelector` | `{}` | Pod 节点选择。 |
| `tolerations` | `[]` | Pod 污点容忍。 |
| `extraVolumes` | `[]` | 追加到内置卷的 Kubernetes 卷。 |
| `extraVolumeMounts` | `[]` | 追加到内置挂载的容器挂载。 |

可通过 `otel-lgtm.env` 提供额外的容器环境变量。要在
`lgtm` 容器日志中暴露 Loki 和后端 Collector 的进程日志以便排查：

```yaml
otel-lgtm:
  env:
    - name: ENABLE_LOGS_LOKI
      value: "true"
    - name: ENABLE_LOGS_OTELCOL
      value: "true"
```

如需自定义后端配置，在发布命名空间中创建 ConfigMap，
并将其文件挂载到镜像使用的路径。例如：

```yaml
otel-lgtm:
  extraVolumes:
    - name: backend-config
      configMap:
        name: my-lgtm-config
  extraVolumeMounts:
    - name: backend-config
      mountPath: /otel-lgtm/loki-config.yaml
      subPath: loki-config.yaml
      readOnly: true
```

使用唯一的卷名和挂载路径，以免破坏内置仪表盘和存储
挂载。修改通过 `subPath` 挂载的 ConfigMap 后需重启 LGTM；
这些变更不会自动滚动 Deployment 或更新挂载的文件。存储仍为临时。