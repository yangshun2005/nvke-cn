# 租户 Kubernetes API 访问

## 概述

NVKE 在该集群 kubeconfig 中记录的端点上暴露每个租户集群的 Kubernetes API。客户端直接连接该端点；NVKE Controller 和 `nvkectl` 都不代理 Kubernetes API 流量。

首选端点是控制平面节点共享的私有 kube-vip 地址。如果集群没有控制平面 VIP，NVKE 改用第一个控制平面节点的 IP 地址。该回退方案没有负载均衡，当该节点或其网络路径不可用时，API 将不可用。

NVKE 不创建公共 Kubernetes API 端点。部署方必须提供路由和过滤，使私有端点可从获批网络（如站点 VPC、租户对等连接或 VPN）访问。对等连接、VPN、防火墙和 DNS 配置的归属方因部署而异，必须在启用租户访问前达成一致。

## 主体类别

下表描述 alpha 访问模型。"租户 API 端点"指配置了控制平面 VIP 时使用该 VIP，否则使用第一个控制平面节点 IP。

| 主体 | 认证 | 授权 | 网络路径 | 凭据有效期 |
|---|---|---|---|---|
| 租户运维人员 | 运维人员通过 `nvkectl get kubeconfig` 获取 `nvke:breakglass` kubeconfig，其中包含 X.509 客户端证书和私钥。 | 证书的组织为 `nvke:breakglass`，绑定到 `cluster-admin`。 | 运维人员从获批私有网络直接连接租户 API 端点。 | 一年。续期和获取详情见 [Alpha 运维人员访问](#alpha-operator-access)。 |
| 租户用户 | NVKE alpha 不提供独立的、可单独归属的租户用户凭据。租户运维人员必须在 NVKE 之外建立用户认证。 | 租户运维人员创建所需的 Kubernetes RBAC 绑定。NVKE 不授予默认租户用户角色。 | 用户从获批私有网络直接连接租户 API 端点。 | 由租户选择的认证机制决定。 |
| 租户自动化 | 租户运维人员创建 Kubernetes ServiceAccount 并使用 TokenRequest 令牌，或配置其他 Kubernetes 支持的认证器。NVKE 不分发单独的自动化凭据。 | 租户运维人员将 ServiceAccount 或其他身份绑定到特定于工作负载的 Role 或 ClusterRole。NVKE 不授予默认角色。 | 自动化直接连接租户 API 端点，通常从租户网络或集群内部连接。 | TokenRequest 令牌有时间限制。其请求的生命周期和最大生命周期属于部署策略，而非 NVKE 租户访问设置。 |
| NVKE Controller | NVKE Controller 的 SPIFFE 工作负载身份获取短期 JWT-SVID，可选地通过配置的令牌交换服务交换，然后将得到的 bearer 令牌提交给租户 API 服务器。 | 令牌的 `sub` 声明映射到 Kubernetes 用户。NVKE 将该用户绑定到 `nvke:cwe` ClusterRole 以及生命周期操作所需的命名空间级 Role。 | NVKE Controller 从管理集群通过站点私有网络直接连接租户 API 端点。 | JWT 过期时间由签发身份系统设置。客户端获取新令牌，而非存储长期 bearer 令牌。 |
| Cluster API 控制器 | 每个控制器使用其 SPIFFE 工作负载身份和与 NVKE Controller 相同的 JWT 认证路径。 | 当前 NVKE 清单将 CAPI 管理器身份绑定到 `cluster-admin`。 | 控制器从管理集群直接连接租户 API 端点。 | JWT 过期时间由签发身份系统设置，客户端按需获取替换令牌。 |
| 故障修复系统 | NVKE alpha 未定义通用故障修复凭据。故障修复部署必须使用专用 ServiceAccount 或工作负载身份，而非运维人员 admin kubeconfig。 | 部署方创建与系统执行的修复操作匹配的 RBAC。NVKE 不授予默认故障修复角色。 | 系统从其获批运行时网络直接连接私有租户 API 端点。 | 由所选 ServiceAccount 或工作负载身份机制决定。 |
| Break-glass 运维人员 | 运维人员获取与 alpha 租户运维人员路径相同的 `nvke:breakglass` kubeconfig。对管理集群 Secret 的访问是交付控制点。 | 通过 `nvke:breakglass` 组获得 `cluster-admin`。 | 运维人员必须同时具备管理集群访问权限以获取 kubeconfig，以及到租户 API 端点的私有网络可达性。 | 同样适用一年期证书和续期行为。导出的副本不会自动刷新或获得续期后的证书。在证书过期或集群 CA 和凭据被替换之前，将每个下载的副本视为敏感信息。 |

常规租户运维人员访问与 break-glass 访问之间的重叠是 alpha 限制：两者使用相同的高权限、不可归属凭据。不要将此 kubeconfig 共享给应用或提交到仓库。

## Alpha 运维人员访问

运维人员首先为 `nvkectl` 配置管理集群的 kubeconfig。该管理集群身份必须被允许读取 `nvke-system` 中的租户访问 Secret。设置以下命令使用的三个值。`NVKE_CONTROLLER_URL` 是 NVKE API 的基础地址 `http://<MANAGEMENT_ADDRESS>:30080`，其中 `<MANAGEMENT_ADDRESS>` 是管理集群的 API 虚拟 IP 地址（VIP），或单节点管理集群的节点 IP。

```bash
export SITE_NAME="<SITE_NAME>"
export NVKE_CONTROLLER_URL="http://<MANAGEMENT_ADDRESS>:30080"
export CLUSTER_NAME="<TENANT_CLUSTER_NAME>"

nvkectl config set-site --name "$SITE_NAME" \
  --url "$NVKE_CONTROLLER_URL" \
  --kubeconfig site-controller.kubeconfig

nvkectl get kubeconfig "$CLUSTER_NAME" --output-file tenant.kubeconfig
kubectl --kubeconfig tenant.kubeconfig get nodes
```

`nvkectl get kubeconfig` 从管理集群读取 `nvke-system/<cluster>-admin-kubeconfig`。该 Secret 包含 `nvke:breakglass` 客户端证书。服务器 URL 和集群 CA 来自控制平面节点的 `admin.conf`；客户端密钥则不是。随后生成的 `kubectl` 进程直接连接该 kubeconfig 中的服务器 URL。

获取 kubeconfig 证明对管理集群 Secret 的访问权限；并不证明运维人员的工作站可以路由到租户 API 端点。

`nvke:breakglass` 客户端证书有效期为一年。租户访问在每次协调集群时检查它，数据库重新同步大约每 15 分钟执行一次。当剩余有效期小于 `scheduledMaintenance.certRenewal.thresholdDays`（默认 30）时，它提交名为 `nvke-breakglass` 的证书签名请求并批准。kube-controller-manager 签发证书，租户访问替换 Secret。节点的 `admin.conf` 用于批准请求并复制服务器 URL 和集群 CA。该 `admin.conf` 由单独的 12 小时 kubeadm 证书续期保持有效，该续期不会在恰好有两个控制平面节点的集群上运行；参见[证书续期](../../controller/docs/certificate-renewal.md)。续期更新 Secret，而非运维人员已下载的 kubeconfig 文件。运维人员必须在本地副本中的证书过期前获取刷新的 kubeconfig。

目标选择、强制调用、健康检查、叶子证书验证和失败周期恢复请遵循[租户续期运行手册](certificate-renewal.md)。叶子证书续期不会重新签发已下载的 breakglass kubeconfig。[凭据操作清单](../operations/credential-lifecycle.md)说明了现有 CA 配置文件和延迟吊销/CA 轮换能力。

## 服务身份与租户 OIDC

NVKE 现有的 SPIFFE/OIDC 集成用于服务到集群 API 的访问。它与 alpha 人工运维人员流程相互独立。

对于启用 OIDC 的租户集群，租户 API 服务器使用 Kubernetes 结构化认证配置。它验证配置的签发者、签名、受众和过期时间，并将 JWT `sub` 声明映射到 Kubernetes 用户名。然后 Kubernetes RBAC 授权该用户名。OIDC 发现提供程序发布发现元数据和签名密钥；它不授予权限。

NVKE Controller 发布单独的 `nvke-system/<cluster>-kubeconfig` 供自己使用。在启用 OIDC 的集群上，此 kubeconfig 运行 SPIFFE exec-auth 插件以获取 JWT。它不是 `nvkectl get kubeconfig` 返回的 kubeconfig。

## 授权模型

认证建立 Kubernetes 用户名，并在适用时建立组。Kubernetes RBAC 做出单独的授权决定：

- alpha admin 证书是 `nvke:breakglass` 组中的客户端，该组绑定到 `cluster-admin`。
- NVKE Controller 的 JWT `sub` 映射到 NVKE `nvke:cwe` 绑定中指定的用户。该标识符保留在当前 RBAC 清单中。其 ClusterRole 和命名空间 Role 列举了 NVKE Controller 可执行的生命周期操作。
- CAPI 管理器 JWT 主体当前绑定到 `cluster-admin`。
- 租户用户、租户自动化和故障修复系统不会获得额外的默认 NVKE 角色。租户管理员必须为它们创建显式绑定。

RBAC 授权不会使端点可达，网络可达性也不会授予 Kubernetes 权限。两项检查都必须成功。

## 计划中的人工 OIDC 访问

alpha 之后，租户运维人员和用户预计通过获批身份提供程序认证，而非共享 `nvke:breakglass` kubeconfig。预期流程为：

1. 用户向身份提供程序认证。
2. 客户端获取受众限定、有时间限制的 OIDC JWT。
3. 租户 Kubernetes API 服务器验证签发者、签名、受众、过期时间和声明。
4. API 服务器将 JWT `sub` 声明（最终还包括获批的组声明）映射到 Kubernetes 用户名和组。
5. Kubernetes RBAC 仅授予绑定到该身份的角色。

身份提供程序集成、kubeconfig 或 exec 插件交付、声明约定、默认租户角色、组映射和吊销行为属于未来工作。在这些契约实现之前，不得将其呈现为可用的租户访问。

## 网络路径

### 管理平面到租户集群

NVKE Controller 和 Cluster API 控制器从 Site Controller 集群连接私有租户 API 端点。此路径是集群生命周期和协调所必需的。管理平面部署必须能够路由到端点上的 TCP 端口 6443。

### VPC 内访问

已在具有到租户端点路由的网络上的客户端可以直接连接 TCP 端口 6443。TLS 认证 API 服务器，请求中的凭据认证客户端。

### 租户对等连接或 VPN

直接连接网络之外的运维人员需要获批的私有路径，例如租户对等连接或 VPN。NVKE 记录并配置 Kubernetes API 端点，但本身不建立运维人员的对等连接、VPN、防火墙或 DNS 配置。

### 公共网络

NVKE 不为租户 Kubernetes API 配置公共负载均衡器或公共入口。部署网络必须拒绝来自不受信任和公共网络的访问。这是部署属性，必须在目标环境中验证，而不能仅从 kubeconfig 地址推断。

## 手动验证清单

从获批测试位置对非生产集群运行这些检查。不要将 kubeconfig 内容、客户端密钥或 bearer 令牌粘贴到工单或日志中。

1. 创建自动清理的临时目录，然后获取 alpha 运维人员 kubeconfig。在同一 shell 中运行其余命令，以保持清理 trap 处于活动状态：

   ```bash
   VALIDATION_DIR="$(mktemp -d)" || exit 1
   readonly VALIDATION_DIR
   TENANT_KUBECONFIG="${VALIDATION_DIR}/tenant.kubeconfig"
   TENANT_CA="${VALIDATION_DIR}/tenant-ca.pem"
   VALIDATION_NAMESPACE=
   cleanup_validation() {
     if [ -n "${VALIDATION_NAMESPACE}" ]; then
       kubectl --kubeconfig "$TENANT_KUBECONFIG" delete namespace \
         "$VALIDATION_NAMESPACE" --ignore-not-found --wait=false \
         >/dev/null 2>&1 || true
     fi
     rm -rf -- "${VALIDATION_DIR:?}"
   }
   trap cleanup_validation EXIT

   nvkectl get kubeconfig "$CLUSTER_NAME" --output-file "$TENANT_KUBECONFIG"
   chmod 0600 "$TENANT_KUBECONFIG"
   ```

2. 确认 kubeconfig 指定了预期的 VIP 或回退节点 IP：

   ```bash
   TENANT_API_SERVER="$(kubectl --kubeconfig "$TENANT_KUBECONFIG" \
     config view --minify -o jsonpath='{.clusters[0].cluster.server}')"
   printf '%s\n' "$TENANT_API_SERVER"
   ```

   检查嵌入的客户端证书的准确有效期窗口：

   ```bash
   kubectl --kubeconfig "$TENANT_KUBECONFIG" \
     config view --raw --flatten --minify \
     -o jsonpath='{.users[0].user.client-certificate-data}' \
     | openssl base64 -d -A \
     | openssl x509 -noout -subject -dates
   ```

3. 从获批网络验证已认证的 API 访问和身份：

   ```bash
   kubectl --kubeconfig "$TENANT_KUBECONFIG" get --raw=/readyz
   kubectl --kubeconfig "$TENANT_KUBECONFIG" auth whoami
   kubectl --kubeconfig "$TENANT_KUBECONFIG" auth can-i '*' '*'
   ```

   alpha admin kubeconfig 必须能够访问 API，识别 `nvke:breakglass` 组中的客户端，并报告所有操作均被允许。

4. 验证未认证请求无法列出租户资源。使用租户 CA 但不提供客户端证书或 bearer 令牌：

   ```bash
   kubectl --kubeconfig "$TENANT_KUBECONFIG" \
     config view --raw --flatten --minify \
     -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' \
     | openssl base64 -d -A > "$TENANT_CA"

   curl --silent --show-error --output /dev/null --write-out '%{http_code}\n' \
     --cacert "$TENANT_CA" "$TENANT_API_SERVER/api/v1/nodes"
   ```

   请求不得返回成功的节点列表。取决于 API 服务器的匿名认证配置，拒绝可能是 HTTP 401 或 HTTP 403。当 shell 退出时，清理 trap 会移除两个临时产物。

5. 从具有 SPIRE socket 和租户服务 kubeconfig 访问权限的授权管理集群工作负载环境，验证服务身份和有效 RBAC：

   ```bash
   kubectl --kubeconfig tenant-service.kubeconfig auth whoami
   kubectl --kubeconfig tenant-service.kubeconfig auth can-i --list
   ```

   NVKE Controller 必须仅获得 `nvke:cwe` 中的权限。Cluster API 控制器身份当前获得 `cluster-admin`。受众错误或 `sub` 未绑定的 JWT 必须被拒绝或拒绝授权。

6. 为租户自动化创建测试 ServiceAccount 和窄范围 Role。使用命名 ConfigMap，使允许的请求只需要 `get` 动词，然后确认绑定之外的集群级请求收到 HTTP 403：

   ```bash
   VALIDATION_NAMESPACE="nvke-api-access-validation-$(date +%s)"
   kubectl --kubeconfig "$TENANT_KUBECONFIG" \
     create namespace "$VALIDATION_NAMESPACE"
   kubectl --kubeconfig "$TENANT_KUBECONFIG" -n "$VALIDATION_NAMESPACE" \
     create serviceaccount reader
   kubectl --kubeconfig "$TENANT_KUBECONFIG" -n "$VALIDATION_NAMESPACE" \
     create configmap validation-target --from-literal=result=expected
   kubectl --kubeconfig "$TENANT_KUBECONFIG" -n "$VALIDATION_NAMESPACE" \
     create role configmap-reader --verb=get --resource=configmaps
   kubectl --kubeconfig "$TENANT_KUBECONFIG" -n "$VALIDATION_NAMESPACE" \
     create rolebinding configmap-reader --role=configmap-reader \
     --serviceaccount="$VALIDATION_NAMESPACE:reader"

   TENANT_TOKEN="$(kubectl --kubeconfig "$TENANT_KUBECONFIG" \
     -n "$VALIDATION_NAMESPACE" create token reader --duration=10m)"
   kubectl --server="$TENANT_API_SERVER" --certificate-authority="$TENANT_CA" \
     --token="$TENANT_TOKEN" -n "$VALIDATION_NAMESPACE" \
     get configmap validation-target -o name
   kubectl --server="$TENANT_API_SERVER" --certificate-authority="$TENANT_CA" \
     --token="$TENANT_TOKEN" get nodes

   kubectl --kubeconfig "$TENANT_KUBECONFIG" \
     delete namespace "$VALIDATION_NAMESPACE" --wait=true
   VALIDATION_NAMESPACE=
   unset TENANT_TOKEN
   ```

   命名 ConfigMap 请求必须成功。节点列表请求必须以 HTTP 403 失败。显式删除会移除测试对象；如果 shell 提前退出，退出 trap 也会请求删除命名空间。

7. 从故意未对等、未路由或未通过获批 VPN 连接的网络，确认租户端点不可达。另外验证没有公共 DNS 记录或公共负载均衡器暴露它。

8. 如果集群使用第一个控制平面节点回退，重复端点检查并记录该路径不是高可用。验证负载均衡端点模型时使用启用 VIP 的集群。

人工 OIDC 检查仅在身份提供程序和 RBAC 映射契约实现后才适用。

## 通过堡垒机访问租户 Kubernetes API

当租户 API 端点无法从工作站路由，但可从堡垒机路由时，使用以下两种方法之一。两种方法都：

- 从管理集群获取租户 kubeconfig。
- 需要 SSH 访问堡垒机，但不需要访问租户控制平面节点。
- 需要堡垒机具有到租户 API 端点的网络连接。

NVKE 不创建或配置堡垒机，也不分发用于访问它的 SSH 密钥。

| | 选项 A：在笔记本电脑上使用 `kubectl` | 选项 B