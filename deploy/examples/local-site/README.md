# local-site

本地站点的输入：由 `nvkectl` 引导，在容器节点上端到端运行 operator 流程。`hack/local-site.sh setup`（`make local-site-setup`）将这些输入复制到 operator 的工作区，`deploy/nvke-deploy-local.sh` 从那里安装；[在一台机器上端到端运行 NVKE](../../../docs/get-started/local-site.md) 是相关页面。

| 文件 | 使用者 |
|---|---|
| `stack-config.json` | `nvke-deploy.sh install`：nico-sim 在节点网络网关上的端点及其种子组织，以及 setup 从检出构建并暂存到节点中的控制器和 profile 镜像，替代两个 `REPLACE_` 值 |
| `releases/{machineimage,clusterrelease,nodepoolrelease}-{amd64,arm64}.yaml` | `nvkectl create <kind> -f`：租户节点启动所用的 `local-site-<arch>` 发布版本集 |
| `tenant-{amd64,arm64}.yaml` | `nvkectl create cluster -f`：基于 nico-sim 种子标识符的单控制平面、单工作节点租户 |
| `nvke-api-clients.yaml` | `kubectl apply -f`，替换两个 `REPLACE_` CIDR 后：准入 NVKE API 的网络 |

站点 spec 不在此处：`nvkectl generate sitespec` 每次运行时从 setup 启动的节点生成它。