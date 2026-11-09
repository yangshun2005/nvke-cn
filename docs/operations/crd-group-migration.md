# 将 CR 迁移到 internal.nvke.nvidia.com 组

management-engine 的内部 API 组从 `internal.nke.nvidia.com` 改为 `internal.nvke.nvidia.com`，集群 kind 从 `NKECluster` 改为 `NVKECluster`，这是重命名为 NVKE 的一部分。

**组变更意味着新建 CRD，而非编辑。** 没有转换 webhook，也没有自动迁移：新 CRD 与旧 CRD 并存安装，旧自定义资源原地保留，直到你手动复制过去。未执行此操作的站点升级后，management-engine 将看不到任何集群和发布版本。

全新安装无需此操作。以下步骤适用于已有站点。

## 迁移内容

| Kind（旧 → 新） | 作用域 | Status 子资源 |
|---|---|---|
| `NKECluster` → `NVKECluster` | 命名空间级 | 是 |
| `ClusterRelease` | 集群级 | 是 |
| `NodePoolRelease` | 集群级 | 是 |
| `MachineImage` | 集群级 | 是 |
| `ClusterReleaseChannel` | 集群级 | 否 |

仅集群 kind 重命名。其余四个保留名称，只变更组。

Finalizer `internal.nke.nvidia.com/cluster-cleanup` 变为 `internal.nvke.nvidia.com/cluster-cleanup`。无需手动添加新的 finalizer——控制器会在首次 reconcile 时设置。

## 开始之前

- 备份旧资源。步骤 2 会写一份备份，步骤 6 删除原始资源后，这是唯一的副本。
- 需要 `jq` 和能访问管理集群的 `kubectl`。
- 了解你的 NVKECluster 命名空间。示例使用 `--all-namespaces`。

> **Status 很重要。** `NVKECluster.status.provisioning` 将集群绑定到创建它的 NVKE 控制器集群：`endpoint` 是提供该集群的服务，`id` 是其 UUID。status 缺失时，reconciler 会回退到按名称查找，因此 status 丢失本身不会导致重复供应——但 `id` 也是防止*已删除*的集群被同名新集群静默替换的关键。请复制 status。步骤 4 会执行此操作。

## 操作步骤

### 1. 停止 management-engine

两个控制器通过不同 CRD reconcile 同一批集群会冲突。

`cloud-workflow-engine`（CWE）chart 也受影响：它读取 `machineimages`、`clusterreleases` 和 `nodepoolreleases` CR 来解析发布版本名称，其 RBAC 现在只授予新组。在步骤 3 应用发布版本之前，运行新 chart 的 CWE 看不到**任何发布版本**，无法为新供应解析发布版本。现有租户集群不受影响。因此：

- 在 chart 升级之前或同一维护窗口内执行步骤 1–5，并先安装新 CRD（chart 会执行此操作；不要升级站点的 CWE 后拖延后续步骤）。
- 在步骤 5 验证通过之前，不要通过 chart 升级创建、调整或升级租户集群。

```bash
kubectl -n nke-system scale deployment/management-engine --replicas=0
kubectl -n nke-system rollout status deployment/management-engine --timeout=120s
```

### 2. 导出现有资源

```bash
mkdir -p crd-group-backup && cd crd-group-backup

kubectl get nkeclusters.internal.nke.nvidia.com --all-namespaces -o json > nkeclusters.json
for k in clusterreleases nodepoolreleases machineimages clusterreleasechannels; do
  kubectl get "${k}.internal.nke.nvidia.com" -o json > "${k}.json"
done
```

继续之前确认导出非空：

```bash
for f in *.json; do echo "${f}: $(jq '.items | length' "${f}")"; done
```

### 3. 重写并应用 spec

`jq` 会移除无法重新应用的服务器端 metadata——`uid`、`resourceVersion`、`generation`、`creationTimestamp`、`managedFields`、`selfLink`——以及旧 finalizer，否则它会被带到控制器从未设置它的对象上。它还会移除描述旧组的 `kubectl.kubernetes.io/last-applied-configuration`，并将注解键从旧组前缀重写为新前缀。发布版本 CR 带有 `internal.nke.nvidia.com/machine-init-kubernetes-version` 和 `...-injected`；控制器在新前缀下读取它们，因此保留旧键的副本会静默丢失每个发布版本支持的 Kubernetes 版本信息。

```bash
rewrite() {  # $1 = 输入文件，$2 = 新 apiVersion，$3 = 新 kind（空则保留）
  jq --arg av "$2" --arg kind "$3" '
    .items
    | map(
        .apiVersion = $av
        | (if $kind != "" then .kind = $kind else . end)
        | del(
            .metadata.uid, .metadata.resourceVersion, .metadata.generation,
            .metadata.creationTimestamp, .metadata.managedFields,
            .metadata.selfLink, .metadata.ownerReferences
          )
        | del(.metadata.annotations["kubectl.kubernetes.io/last-applied-configuration"])
        | (if .metadata.annotations then
             .metadata.annotations |= with_entries(
               .key |= sub("^internal\\.nke\\.nvidia\\.com/"; "internal.nvke.nvidia.com/"))
           else . end)
        | (if .metadata.annotations == {} then del(.metadata.annotations) else . end)
        | .metadata.finalizers = (
            (.metadata.finalizers // [])
            | map(select(startswith("internal.nke.nvidia.com/") | not))
          )
        | if (.metadata.finalizers | length) == 0 then del(.metadata.finalizers) else . end
      )
    | {apiVersion: "v1", kind: "List", items: .}
  ' "$1"
}

rewrite nkeclusters.json         internal.nvke.nvidia.com/v1 NVKECluster > new-nvkeclusters.json
for k in clusterreleases nodepoolreleases machineimages clusterreleasechannels; do
  rewrite "${k}.json" internal.nvke.nvidia.com/v1 "" > "new-${k}.json"
done
```

先应用发布版本：集群在 reconcile 时解析其发布版本通道，通道尚不存在会导致 reconcile 失败，之后还需手动清除。

```bash
for k in machineimages clusterreleases nodepoolreleases clusterreleasechannels; do
  kubectl apply -f "new-${k}.json"
done
kubectl apply -f new-nvkeclusters.json
```

### 4. 恢复 status 子资源

`kubectl apply` 不会写入 status 子资源。从备份中逐个 patch。

```bash
restore_status() {  # $1 = 备份文件，$2 = 新 plural.group，$3 = 命名空间级时填 "ns"
  jq -r '.items[] | select(.status != null)
         | [.metadata.namespace // "", .metadata.name, (.status | tojson)] | @tsv' "$1" \
  | while IFS=$'\t' read -r ns name status; do
      if [ "$3" = "ns" ]; then
        kubectl -n "$ns" patch "$2" "$name" --subresource=status \
          --type=merge -p "{\"status\":${status}}"
      else
        kubectl patch "$2" "$name" --subresource=status \
          --type=merge -p "{\"status\":${status}}"
      fi
    done
}

restore_status nkeclusters.json      nvkeclusters.internal.nvke.nvidia.com     ns
for k in clusterreleases nodepoolreleases machineimages; do
  restore_status "${k}.json" "${k}.internal.nvke.nvidia.com"
done
```

`ClusterReleaseChannel` 没有 status 子资源，跳过。

在 management-engine 仍缩容为零时（步骤 1）执行此操作。merge patch 无法移除运行中控制器已写入的字段，因此在控制器启动后恢复 status 可能留下不完整的 status。

### 5. 删除前验证

数量匹配，且每个集群保留了其供应标识：

```bash
kubectl get nvkeclusters.internal.nvke.nvidia.com --all-namespaces
kubectl get nvkeclusters.internal.nvke.nvidia.com --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.provisioning.id}{"\n"}{end}'
```

每一行必须携带与 `nkeclusters.json` 中对应行相同的 UUID。原本有 `id` 的集群出现空 `id`，说明步骤 4 未生效——继续之前修复，不要事后补救。

发布版本注解已在新前缀下迁移，旧前缀下没有（两条命令都应列出每个发布版本及其版本注解，第二条应无输出）：

```bash
kubectl get clusterreleases.internal.nvke.nvidia.com,nodepoolreleases.internal.nvke.nvidia.com \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations}{"\n"}{end}'
kubectl get clusterreleases.internal.nvke.nvidia.com,nodepoolreleases.internal.nvke.nvidia.com,machineimages.internal.nvke.nvidia.com \
  -o json | grep -F 'internal.nke.nvidia.com'
```

恢复控制器并让其 reconcile：

```bash
kubectl -n nke-system scale deployment/management-engine --replicas=1
kubectl -n nke-system rollout status deployment/management-engine --timeout=300s
kubectl -n nke-system logs deployment/management-engine --tail=100
```

每个 `NVKECluster` 应达到 `Ready` 并获取新 finalizer：

```bash
kubectl get nvkeclusters.internal.nvke.nvidia.com --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.finalizers}{"\n"}{end}'
```

**任何租户集群都不应重启、重新供应或改变状态。** 如有发生，停止操作并从 `crd-group-backup` 恢复，不要让 reconcile 继续运行。

### 6. 删除旧资源

仅在步骤 5 验证通过后执行。旧控制器已不存在，没有东西会移除旧 finalizer——先剥离它们，否则删除会永久挂起。

```bash
kubectl get nkeclusters.internal.nke.nvidia.com --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.namespace}{" "}{.metadata.name}{"\n"}{end}' \
| while read -r ns name; do
    kubectl -n "$ns" patch nkeclusters.internal.nke.nvidia.com "$name" \
      --type=merge -p '{"metadata":{"finalizers":null}}'
  done
```

删除旧 CRD 会同时删除其自定义资源：

```bash
for c in nkeclusters clusterreleases nodepoolreleases machineimages clusterreleasechannels; do
  kubectl delete crd "${c}.internal.nke.nvidia.com" --ignore-not-found
done
```

chart 将这些 CRD 标记为 `helm.sh/resource-policy: keep`，因此 Helm 升级不会替你移除它们。此步骤是刻意的手动操作。

在新组上完成完整供应周期之前，保留 `crd-group-backup`。

## 回滚

步骤 6 之前，回滚方式：将 management-engine 缩容为零，删除新 CR，重新安装之前的 chart，再扩容。旧 CR 从未被触碰。

步骤 6 之后，旧 CRD 已删除，回滚意味着重新安装之前的 chart，并按相同步骤反向重放 `crd-group-backup`。这就是步骤 5 在步骤 6 删除之前进行验证的原因。