# deploy

将 NVKE 管理平面安装到 `nvkectl bootstrap site` 留下的站点集群上：Postgres、Temporal、SPIRE、CAPNICo、NKTP 和 NVKE 控制器。一个场景：NVKE 搭配 CAPNICo，运行在 NICo 站点上。

两个脚本：`check-site.sh` 报告站点还缺什么，`nvke-deploy.sh` 安装部署路径。`nvke-deploy-local.sh` 封装第二个脚本，用于[在一台机器上端到端运行 NVKE](../docs/get-started/local-site.md) 的本地站点：之前准备实验用 StorageClass，从环境读取输入，之后打 SPIRE exec-auth 补丁。运行前请先阅读它们。

## 输入

| 输入 | 来源 | 说明 |
|---|---|---|
| 站点 spec | `nvkectl generate sitespec`，即 bootstrap 运行时使用的文件 | bootstrap 构建了什么，以及栈必须与之一致的内容：站点名称、控制平面 VIP 和 SPIRE 信任域。安装过程读取它，不做任何修改。[docs/sitespec.md](../cli/docs/sitespec.md) |
| 栈配置 `stack-config.json` | 复制并填写 [`examples/stack-config.json`](examples/stack-config.json) | 仅栈需要的内容：NICo API，以及可选的镜像覆盖和镜像仓库。参见[栈配置](#the-stack-config) |
| 发布版本描述 | `nvkr release export --reference <repo@digest>` | 使用哪些已发布的构建产物，每个以 OCI 引用和摘要指定：`nvkr.dev/release-description/v1` JSON。它只描述发布版本，不做任何验证；请先对同一引用运行 `nvkr release verify`。可选；不提供时，chart 来自本检出目录和公共仓库，镜像由栈配置指定 |
| 部署路径 [`nico/`](nico/) | 本检出目录 | 安装什么以及按什么顺序。`nvke-deploy.yaml` 列出各项；每个条目目录包含其 chart 版本锁定（`release.yaml`）、values 和 overlay，或其清单文件。[deploy-repository.md](../docs/site-bootstrap/deploy-repository.md) |

安装过程从 `NVKECTL_NICO_TOKEN`、`NVKECTL_NODE_USER`、`NVKECTL_NODE_PASSWORD` 和 `NVKECTL_DATABASE_PASSWORD` 读取 NICo token、节点登录信息和数据库密码，从 `NVKECTL_SPIRE_CA_CERT_FILE` 和 `NVKECTL_SPIRE_CA_KEY_FILE` 指定的文件读取 SPIRE CA，站点 spec 中 `spire.upstreamCA: true` 时必需。渲染过程将它们写入 Secret（`nvke-database`、`spire-upstream-ca`、模板条目的 Secret），chart 按名称读取；没有 values 文件携带凭据。节点的 CPU 架构不在任何文件中：两个脚本都从集群的 `kubernetes.io/arch` 节点标签读取，执行器将其传给渲染过程，渲染过程据此选择发布版本构建产物和 SPIRE TPM 证明者校验和。`nvkectl` 读取部署路径，为站点并针对发布版本渲染到 `$XDG_STATE_HOME/nvke/renders`（`~/.local/state/nvke/renders`）下的私有临时目录；脚本退出时删除该目录。

## 运行

[在一台机器上端到端运行 NVKE](../docs/get-started/local-site.md) 按顺序给出了安装步骤及每条命令的作用。`nvke-deploy-local.sh` 是入口点；它使用环境中的输入调用 `nvke-deploy.sh`。

```bash
clusterctl init --kubeconfig local-site.kubeconfig --core cluster-api:<v> --bootstrap kubeadm:<v> --control-plane kubeadm:<v> --wait-providers
"$NVKE_REPO"/deploy/nvke-deploy-local.sh install
kubectl --kubeconfig local-site.kubeconfig apply -f nvke-api-clients.yaml   # the NetworkPolicy admitting your client networks
nvkectl config set-site --name <site> --url http://<ADDRESS>:30080 --kubeconfig local-site.kubeconfig
nvkectl create machineimage -f machineimage.yaml && nvkectl create clusterrelease -f clusterrelease.yaml && nvkectl create nodepoolrelease -f nodepoolrelease.yaml
```

下面两个脚本是 `check-site.sh` 和 `nvke-deploy.sh` 的参考文档，`nvke-deploy-local.sh` 封装了它们。

## 栈配置

一个 JSON 对象，仅包含 [`examples/stack-config.json`](examples/stack-config.json) 中的键，不得有其他键：未知、重复或大小写不同的键会被拒绝，仍为 `REPLACE_*` 占位符的值也会被拒绝。复制示例、填写，并与站点 spec 放在一起；它不含机密。`nvkectl inspect stack-config stack-config.json`（内部命令；两个脚本都会运行）以 `<key>: <problem>` 格式一次性报告所有问题，并同时检查 `NVKECTL_*` 凭据。

| 键 | 必需 | 值 |
|---|---|---|
| `nico.endpoint` | 是 | CAPNICo 调用的 NICo API 基础 URL，`http` 或 `https`，不含用户名或密码 |
| `nico.orgID` | 是 | NICo 组织 ID |
| `nico.apiName` | 是 | NICo API 路径中的 API 名称，例如 `forge` |
| `images.controller` | 否 | 覆盖控制器镜像，格式为 `repository:tag` 或 `repository@sha256:<digest>`；默认为发布版本的 `cwe` 构建产物，无发布版本时默认为 overlay 中锁定的镜像 |
| `images.profiles` | 否 | 覆盖 profile 内容镜像；默认为发布版本的 `controller-profiles` 构建产物。无发布版本时必需：`deploy/nico` 启用了该镜像但未指定名称 |
| `mirror` | 否 | 外部镜像的拉取仓库，替代其上游仓库，格式为 `host[:port][/prefix]`，不含 scheme |

## check-site.sh

每条发现一行，PASS、WARN、SKIP 或 FAIL 并附修复建议，任何 FAIL 时以非零状态退出。它不做任何修改。开头是一行 `INFO  inputs:`，以绝对路径列出配对的站点 spec 和栈配置，以及站点名称、NICo 端点和组织，以便在检查前发现错误的文件。全部通过时，它打印剩余的命令：Cluster API 尚未安装时的 `clusterctl init`，以及 `nvke-deploy.sh install`，附带给定路径。

| 发现 | 检查内容 |
|---|---|
| tools | `PATH` 上的 `kubectl`、`helm`、`clusterctl`、`curl`、`python3`；`helm` 版本为 `make/tool-versions.mk` 中锁定的版本；`nvkectl`（`NVKECTL`，否则 `PATH`） |
| checkout | `deploy/nico/nvke-deploy.yaml` 以及控制器和 NKTP chart，执行器从目录树安装它们 |
| Cluster API | `make/tool-versions.mk` 中的 `CAPI_VERSION`；`clusterctl` 为该版本；尚未安装、已按该版本安装且带有 kubeadm provider，或 FAIL：其他版本、仅摘要镜像、或缺少 kubeadm provider 的核心组件 |
| site spec | `nvkectl validate sitespec` 通过 |
| stack config | `nvkectl inspect stack-config` 接受该文件并找到 `NVKECTL_*` 凭据：严格 JSON、无占位符、所有必需值已设置且格式正确、所有变量已设置。FAIL 携带第一个 `<key>: <problem>` |
| release | `--release` 可解析并列出其引用和声明的资格；`--nvkr-run-dir` 存在 |
| cluster | API 以 `--kubeconfig` 响应；节点数为奇数；每个节点都有 `kubernetes.io/arch` 且架构一致；无控制平面 `NoSchedule` 污点；恰好一个默认 StorageClass |
| egress | 固定端点列表在 10 秒内响应：`ghcr.io`、`github.com`、NKTP chart 索引、`quay.io`、`registry.k8s.io`（WARN） |
| disk | 状态目录下至少 10 GB 可用空间（WARN） |

`--diagnostics` 配合 `--kubeconfig` 时，改为将节点、Pod、事件、helm release 和控制器日志写入 `NVKE_STATE_DIR/diagnostics` 并以 0 退出；在步骤失败后运行。

## nvke-deploy.sh

```bash
deploy/nvke-deploy.sh install --deploy-path DIR [--sitespec FILE --stack-config FILE [--arch linux/ARCH]] [--release FILE] [--nvkr-run-dir DIR] \
  [--config bootstrap-config.json] [--values-for ITEM=FILE]... [--kubeconfig PATH] [--timeout 15m] [--dry-run] [-- helm args]
```

执行器：遍历 `nvke-deploy.yaml`，按列表顺序应用各项。使用 `--sitespec` 和 `--stack-config`，或 `--release` 时，它先为站点并针对发布版本将部署路径渲染到 `NVKE_STATE_DIR` 下的私有临时目录，从那里安装，退出时删除，不留下任何渲染后的凭据。`--sitespec` 和 `--stack-config` 必须同时提供，`--release` 需要两者。渲染前，它记录与检查相同的 `inputs:` 行，从集群读取节点架构并作为 `--arch` 传给渲染过程；在执行器上指定 `--arch linux/amd64` 或 `--arch linux/arm64` 可替代读取。无节点、节点缺少 `kubernetes.io/arch` 标签、两种架构或 API 不可达都会中止安装；API 不可达时的 `--dry-run` 按本机架构渲染并说明。`--dry-run` 改为打印每个 kubectl 和 helm 的 argv。e2e 流水线用它安装 `controller/deploy/capd-e2e` 和 `capnico-e2e`，`controller/hack/ci/kind-stack-from-release.sh` 通过该列表的步骤 1 到 4 从发布版本在 kind 节点上启动 `deploy/nico`。

| 类型 | 行为 |
|---|---|
| `manifest` | 条目目录中的每个 `.yaml`，按名称顺序执行 `kubectl apply --server-side`；渲染出的 `kustomization.yaml` 通过 `kubectl kustomize` 应用 |
| `template` | 由 `nvkectl render` 为站点渲染；使用 `--config` 时，由 `deploy/lib/render-template.py` 针对该 BootstrapConfig JSON 渲染到临时目录，然后像 manifest 条目一样应用 |
| `helm` | 一个 chart、一个 release，`helm upgrade --install --wait`；`release.yaml` 指定 chart、版本、来源、release 名称、命名空间、values 文件、`setFile` 条目和 `requires` 前置条件 |

chart 来自 `chart.path`（检出目录，先执行 `helm dependency build`，或渲染放入 `charts/` 的归档），否则来自 `chart.repository` 的 `chart.version`。Values 按顺序应用：条目自身的文件，然后渲染添加的文件（`site-values.yaml`，以及发布版本携带该条目镜像时的 `release-images.yaml`），最后是每个 `--values-for`。渲染树中的 `images.txt` 列出在应用任何内容前要导入节点 containerd 的 OCI 布局；`CTR` 指定导入命令（`sudo ctr`，或 kind 节点的 `docker exec -i <node> ctr`）。

## 环境变量

| 变量 | 默认值 | 用途 |
|---|---|---|
| `NVKECTL` | `PATH` 上的 `nvkectl` | 读取部署路径并渲染的二进制文件。`make -C cli build` 生成 `cli/bin/nvkectl` |
| `NVKE_STATE_DIR` | `$XDG_STATE_HOME/nvke` | 渲染临时目录和诊断。绝不用共享临时目录 |
| `CTR` | `ctr` | `images.txt` 的 containerd 导入命令 |
| `PHASE_TIMEOUT`、`--timeout` | `15m` | 每个 release 的 `helm --wait` 超时 |
| `REQUIRE_TIMEOUT` | `180s` | `kubectl wait` 和 rollout 超时 |
| `CAPI_VERSION` | `make/tool-versions.mk` | `check-site.sh` 期望的 Cluster API 版本，并据此打印 `clusterctl init` |
| `CONTROLLER_NAMESPACE` | `nvke-controller` | `check-site.sh --diagnostics` 读取控制器日志的位置 |

## 测试

```bash
make -C deploy test   # builds nvkectl, renders deploy/nico for a site and against a fake Run, runs check-site.sh against stub tools and the executor in --dry-run
```