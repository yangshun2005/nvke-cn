# AGENTS.md — `nktp/`

NVKE 遥测流水线（NKTP）Helm chart。它部署一套 OpenTelemetry 流水线，用于采集集群指标、日志和链路追踪。这里没有 Go 代码，也没有镜像；chart 就是整个组件。

## 先读什么

- [deploy/charts/nktp/README.md](deploy/charts/nktp/README.md) — 概要介绍前置条件、安装、组件和配置模型
- [docs/nktp/architecture/topology.md](../docs/nktp/architecture/topology.md) — Collector 之间如何协作，附有 [docs/nktp/components/](../docs/nktp/components/) 下各组件页面和 [docs/nktp/guides/](../docs/nktp/guides/) 下指南的链接

## 目录结构

```text
deploy/charts/nktp/
  Chart.yaml, Chart.lock   # chart 版本和依赖锁定
  values.yaml              # 每个 Collector 的配置和工作负载
  templates/               # Collector、kube-state-metrics、RBAC、开发后端
  dashboards/              # Grafana 仪表盘：default/ 由开发后端加载，examples/ 仅用于导入
hack/chart-validate.sh     # 渲染并校验 chart；make nktp-chart-lint 会运行它
```

## 命令

在仓库根目录执行：

```bash
make nktp-chart-deps   # 将依赖下载到 deploy/charts/nktp/charts/
make nktp-chart-check  # 对 chart 执行 lint、渲染和校验；合并门禁会运行它
```

两者都需要网络访问。依赖来自 `Chart.yaml` 中指定的 chart 仓库，kubeconform 会下载其 Schema。

## 约束

- chart 版本取自发布标签。`Chart.yaml` 中的 `version:` 是 `0.0.0` 占位符，供本地 `helm lint` 和 `helm template` 使用；不要修改它。`.github/workflows/nktp-chart-publish.yml` 会以 `--version X.Y.Z` 打包 `nktp/vX.Y.Z` 标签，与 controller chart 的打包方式一致。已发布的版本不可变：工作流拒绝用不同内容替换已发布版本，因此对 chart 的任何改动（包括 README 和仪表盘）都需在新标签下发布。
- `Chart.yaml` 和 `Chart.lock` 中的依赖锁定需一起变更，放在单独的 Pull Request 中，并执行 `helm dependency update nktp/deploy/charts/nktp`。两者不一致时 `make nktp-chart-deps` 会失败。
- 切勿提交 `deploy/charts/nktp/charts/`。chart 的 `.gitignore` 会排除下载的归档文件。
- 每个 YAML 文件都带有 Apache-2.0 SPDX 头。在 `templates/` 中它是 Go 模板注释，因此渲染出的清单不会改变：

  ```yaml
  {{- /*
  SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
  SPDX-License-Identifier: Apache-2.0
  */ -}}
  ```

  保留两个裁剪标记。不要用 addlicense 给 `templates/` 添加头：它会写入 `#` 行，Helm 会把这些行复制到每个渲染出的清单中。
- README 会被打包进 chart，其中指向其他文件的链接无法解析。请保持它是自包含的摘要，仅在其最后一节链接到 `docs/nktp/` 下的文档。
- 当改动影响组件行为或 values 时，需在同一个 Pull Request 中更新 `docs/nktp/components/` 下对应的页面，以及 `docs/nktp/guides/` 下展示这些 values 的相关指南。