# Management Engine

管理引擎通过声明式 API 和基于 Cluster API（CAPI）约定的协调机制来管理 NVKE 工作负载集群。其架构将用户和运维人员 API 与内部 Kubernetes 资源及生命周期控制器分离。本模块提供内部资源类型、生成的 CRD 以及 controller-manager 运行时。

| 路径 | 用途 |
| --- | --- |
| `apis/v1` | 资源类型与 scheme 注册 |
| `config/crd/bases` | 生成的 CRD |
| `cmd` | 连接配置与 manager 启动 |
| `controllers` | 协调器 |

## 开发

内部 CRD 和生成的清单参见 [API 生成](DEVELOPMENT.md)。仓库级指南见[开发约定](../docs/development.md)和[工作区指南](../docs/go-workspace.md)。以下命令在 `management-engine/` 下运行。

```sh
make build test lint
make generate manifests # after API or RBAC changes
git diff --exit-code -- apis/v1 config/crd/bases config/rbac/role.yaml chart # check generated-file drift
make test-controller # controller integration tests (envtest)
make -C .. chart-crds # after changing MachineImage, ClusterRelease, or NodePoolRelease
make -C .. chart-crds-check # verify the NVKE controller chart's generated CRDs
make chart-check # generate, lint, and render the management-engine chart
```

做漂移检查时，先确保工作区干净再重新生成。`make test` 包含 `test-controller`；单独执行该命令只运行该测试套件。

### Tilt provider

将 `management-engine` 加入 `controller/tilt_config.json` 的 provider 列表，保留你使用的 infrastructure provider。以 CAPD 为例：

```json
{
  "providers": ["capi_core", "bgp-containers", "capd", "management-engine"]
}
```

完成控制器的常规 Kind/CAPI 设置后，在 `management-engine/` 下运行：

```sh
make tilt-up
```

## API 约定

所有类型使用 `internal.nvke.nvidia.com/v1`。未来的公开租户和运维人员 API 应使用独立版本化的契约。不要直接暴露内部结构体。

| 资源 | 作用域 | 定义与示例 |
| --- | --- | --- |
| NVKECluster | 组织命名空间 | [集群配置](apis/v1/nvkecluster_types.go)、[示例](config/samples/internal_v1_nvkecluster.yaml) |
| MachineImage | 集群 | [不可变镜像标识与 NICo 启动契约](apis/v1/machineimage_types.go) |
| ClusterRelease | 集群 | [控制平面镜像与初始化](apis/v1/clusterrelease_types.go) |
| ClusterReleaseChannel | 集群 | [发布版本目录](apis/v1/clusterreleasechannel_types.go)、[示例](config/samples/internal_v1_clusterreleasechannel.yaml) |
| NodePoolRelease | 集群 | [工作节点镜像与初始化](apis/v1/nodepoolrelease_types.go) |

| 关注点 | 契约 |
| --- | --- |
| 租户隔离 | 每个数据存储中每个组织一个命名空间。未来的公开 API 必须从认证中推导组织身份，并强制执行资源和字段授权。 |
| 标识 | 客户端选择的不可变 DNS 子域名（最多 253 个字符）；UID 区分重建实例。Provider 名称可能有更短的限制。 |
| 并发 | 保留不透明的 `resourceVersion` 和字段所有权。Spec 表达期望状态；控制器通过 status 报告进度。 |
| 条件 | 使用 `metav1.Condition`，每个条件的 `observedGeneration` 与 NVKECluster 的 generation 一致。没有聚合的 `status.observedGeneration`。缺失或过期的条件不构成就绪。未知计数不出现；观测到的零计数显式表示。 |
| 引用 | 在组织命名空间内解析租户引用，并通过 UID 固定生命周期所有权。发布版本引用选择运维人员发布的配置。 |
| 节点池 | `spec.nodePools` 在 NVKECluster 中嵌入[工作节点配置](apis/v1/nodepool_types.go)；参见[集群示例](config/samples/internal_v1_nvkecluster.yaml)。 |
| 副本数 | `spec.nodePools[].replicas` 表达期望的工作节点数；零表示零个工作节点。滚动更新策略控制替换。 |
| 删除 | 使用删除时间戳和控制器持有的 finalizer。 |

Schema 校验无法证明引用存在、当前 UID 或调用者权限。命名空间作用域还需要 API 授权和控制器 RBAC。