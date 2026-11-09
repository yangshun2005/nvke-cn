# 可观测性

NVKE 的运维面分为三层：管理平面服务、Cluster API 基础设施提供程序、租户集群节点。本文分别介绍各层的可观测性工具。

关于凭证过期、续期/轮换健康状况、kubelet-serving CSR 和审计证据，请先阅读 [OSS alpha 凭证运维](credential-lifecycle.md)。[凭证监控指南](../../controller/docs/credential-monitoring.md)定义了当前的观测项和告警规则，包括除抓取之外所需的评估器和通知接收器。

## 管理平面服务

### Temporal Web UI

Temporal Web UI 是观察 NVKE 运行情况的主要窗口。每次集群开通、升级或节点替换都作为 Temporal 工作流运行。操作停滞或失败时，Temporal UI 会显示当前工作流状态和最后运行的活动。

从管理集群端口转发 Temporal Web UI：

```bash
kubectl port-forward -n temporal svc/temporal-web 8233:8080 \
  --kubeconfig <sitename>.kubeconfig
```

然后在浏览器中打开 http://localhost:8233。工作流按命名空间组织；NVKE 工作流运行在 `nvke-controller` 命名空间。

**关注点：**
- 卡在 `Running` 且近期无活动的工作流——通常是基础设施操作停滞、节点无响应，或等待 NICo 超时
- 处于 `Failed` 的工作流——展开历史记录，查看哪个活动失败及返回的错误
- `Pending` 工作流任务积压——表明控制器 worker 存在健康问题

### 控制器日志

NVKE 控制器输出结构化 JSON 日志。从管理集群获取：

```bash
kubectl logs -n nvke-controller deploy/nvke-controller \
  --kubeconfig <sitename>.kubeconfig -f
```

日志包含工作流和活动标识符，可将 Temporal UI 中的失败关联到控制器日志行。在控制器部署环境中设置 `LOG_LEVEL=debug` 可提高日志详细程度。默认情况下，日志不包含每条数据库查询的值和 PostgreSQL 错误消息；[观测 NVKE 及其集群](../operate/observe.md#read-the-management-plane-logs)介绍了如何包含这些内容。

### 控制器 REST API 健康检查

NVKE 控制器 API 暴露存活和就绪端点：

```bash
curl https://<controller-url>/healthz   # liveness
curl https://<controller-url>/readyz    # readiness
```

`200 OK` 表示 API 服务器正常。`nvkectl get providers` 显示 CAPI 基础设施提供程序是否已配置——当提供程序的 CRD 存在、控制器管理器正在运行且凭证 Secret 存在时，返回 `CONFIGURED`。注意，`CONFIGURED` 不会针对提供程序后端验证凭证；即使状态显示 `CONFIGURED`，单独的连接问题仍可能阻止集群创建。

## 基础设施提供程序

### 基础设施提供程序日志

CAPI 基础设施提供程序协调集群和机器对象，并调用底层基础设施 API 来分配和管理裸金属机器。检查基础设施提供程序控制器管理器的日志：

```bash
kubectl logs -n <provider-namespace> deploy/<provider-controller-manager> \
  --kubeconfig <sitename>.kubeconfig -f
```

如果租户集群卡在开通阶段，提供程序日志会显示基础设施 API 是否返回错误、机器分配是否超时，或机器对象是否在等待节点出现在 Kubernetes 中。

### NICo 站点健康

独立检查 NICo 站点。监控和健康指导参见 [NICo 文档](https://docs.nvidia.com/infra-controller/documentation/home)——NVKE 的开通能力完全取决于 NICo 是否拥有健康的主机以及配置实例类型的可用容量。

## 租户集群

租户集群是标准 Kubernetes 集群。使用集群的管理 kubeconfig（通过 `nvkectl get kubeconfig <name>` 获取）运行 `kubectl`，检查节点和工作负载健康。

关于节点级健康与 NVKE 生命周期 API 的集成，参见[节点生命周期](../tenant-clusters/node-lifecycle.md)和 [NVSentinel](https://github.com/NVIDIA/NVSentinel)。