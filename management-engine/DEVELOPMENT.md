# Management Engine 开发

在 `management-engine/` 目录下使用 GNU Make（macOS 上为 `gmake`）运行命令。
生成类目标会安装固定版本的工具。请修改源文件，而非生成的文件。

## 内部 Kubernetes API

在 `apis/v1` 下编辑 Go 类型和 kubebuilder 标记。新增 kind 时，需将其注册到
`SchemeBuilder`，更新 `PROJECT`，并将其 CRD 添加到
`config/crd/kustomization.yaml`。

```sh
make generate manifests
make test-api
```

该命令会重新生成 deepcopy 方法、CRD、RBAC、`dist/install.yaml` 以及 Helm
chart。请保留 `PROJECT` 元数据和脚手架标记；不要使用 `--force`
覆盖已维护的 chart 设置。

修改 MachineImage、ClusterRelease 或 NodePoolRelease 后，还需刷新
NVKE 控制器 chart 中内嵌的 CRD：

```sh
make -C .. chart-crds
make -C .. chart-crds-check
```

请将生成的变更与其源文件一并审查并提交。