# 多个 OIDC 提供商支持：设计

> **状态：** 集群级提供商已实现，见 [#1325](https://github.com/dsx-ai-factory/nvke/pull/1325)、[#1355](https://github.com/dsx-ai-factory/nvke/pull/1355)、[#1425](https://github.com/dsx-ai-factory/nvke/pull/1425)、[#1426](https://github.com/dsx-ai-factory/nvke/pull/1426)、[#1527](https://github.com/dsx-ai-factory/nvke/pull/1527) 和 [#1628](https://github.com/dsx-ai-factory/nvke/pull/1628)（集群级提供商上的 `discoveryURL`）。本文档写于该实现之前，因此"今天"指当时的 NVKE 控制器。**未来范围：按集群提供商 API** 中的按集群提供商 API 尚未实现。
>
> **命名。** 本文档使用撰写时的名称。Chart 现在默认将设置 ConfigMap 命名为 `nvke-controller-config`，将认证 ConfigMap 命名为 `nvke-auth-config`，两者都位于控制器自己的命名空间中；在迁移到 nvke 名称之前安装的环境固定使用 `nke-controller-config` 和 `nke-auth-config`。本文档展示的顶层 `permissiveMode` 键此后已移至 `apiAuth.permissive`。
>
> **发布范围。** 后续版本需要同时支持集群级和按集群控制 OIDC 提供商。本文档涵盖两者：下文提出的集群级提供商 ConfigMap 先发布，用于初始 OSS 版本；**未来范围：按集群提供商 API** 中描述的按集群提供商 API 在后续版本中发布，在集群级机制之上增加按集群控制，而不是替代它。

## 问题

NVKE 目前支持一个 OIDC 提供商：共享的 `NvkeOIDCAuth` 提供商，通过环境变量设置一次，并推送到每个集群用于管理平面（SPIFFE）认证。这对人员或租户访问来说是错误的工具。想要授予人员集群维护访问权限，或将租户访问权限交给其自己的集群的操作员，将不得不重用 NVKE 自身用于管理集群的同一身份。本设计为操作员提供了一种注册自己的提供商而不触及该身份的方法。

## 目标

- 让操作员注册一个或多个 OIDC 提供商，一次即可，NVKE 将其应用于随后创建的每个集群：无需按集群配置，无需操作员构建编排。
- 将这些提供商与现有的共享 NVKE OIDC 提供商（如果启用）合并为每个集群的单个 Kubernetes `AuthenticationConfiguration`，与今天仅共享的情况相同。
- 保证任何操作员配置的提供商都无法在 Kubernetes、kubeadm、NVIDIA 或 NVKE 保留的命名空间内（`system:`、`kubeadm:`、`nke:`、`nvke:`、`nvidia:`，以及配置了 SPIFFE 信任域时的 `spiffe://<trust domain>/`），或 NVKE 自身服务在令牌交换后持有的确切身份之一，生成最终用户名或组，且操作员无法移除或覆盖。

## 非目标（初始 OSS 版本）

- 将提供商集变更传播到已运行的集群。集群保留其创建时的集群级提供商；后续编辑仅影响随后创建的集群。将变更推广到运行中的集群需要自己的推广和故障模式分析，权衡替换节点与跨控制平面就地编辑 `AuthenticationConfiguration` 文件，类似于现有的棕地 `EnableOIDC` 工作流。这是扩展点：本版本不构建任何与以后重用该推广框架不兼容的内容。
- 按集群提供商覆盖或退出。本版本中集群中的所有集群获得相同的合并提供商集；例如，目前还无法为运行 NVKE 自身组件的自管理集群提供比集群其余部分更严格的控制。参见 **未来范围：按集群提供商 API** 了解后续版本添加此功能。
- 操作员提供的用于用户名/组映射的 CEL 表达式。
- 将集群级提供商添加到已存在的集群。在设置 `oidc.providers` 之前创建的集群，或没有 `nke-controller-config` 的集群，没有固定提供商，之后也不会获得。`EnableOIDC` 推广为现有集群开启共享提供商，保留集群创建时的提供商但不添加任何提供商。
- 在 `helm upgrade` 或控制器重启之外，自行向操作员提供持续验证反馈。本版本在 chart 渲染时和控制器启动时进行验证；其间没有任何东西监视问题。
- 更改 NVKE 自身访问租户集群进行生命周期管理的方式。该访问已在每个集群上使用 kubeadm 签发的 `admin.conf` 客户端证书，独立于任何 OIDC 提供商；共享提供商在启用后在此基础上添加可选的基于 SPIFFE 的路径，它不替代 `admin.conf`。配置了零个 OIDC 提供商的集群，无论是集群级、按集群还是共享，在 NVKE 侧都不受影响。

## 提议的解决方案

我们添加一个 Helm 模板化的 ConfigMap，`nke-system/nke-controller-config`，它将本设计涉及的集群级提供商列表与 `nvke-controller` 其余非机密配置合并在一起，后者目前分散在 chart 从单个反射驱动结构体 `internal/config/config.go` 的 `Config` 设置的环境变量中。该结构体并非 OIDC 专用；共享提供商的设置只是其中的一个切片。仅为该切片划出 ConfigMap 而将其余部分保留为环境变量，意味着操作员和读者要学习两个不同的地方来查找本质上是同一类设置的内容。因此本设计不划出切片：`Config` 中作为普通值到达控制器的每个字段都移入此 ConfigMap，直接镜像结构体自身的形状，而不是为其已命名的任何内容发明第二个名称。只有少数作为 `secretKeyRef` 支持的凭据到达控制器的字段保留环境变量路径，因为该路径正是它们避免以纯文本形式出现的方式；确切边界见 **迁移**。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nke-controller-config
  namespace: nke-system
data:
  config.yaml: |
    fieldManager:
      name: cloud-workflow-engine
      rebootMachineName: cloud-workflow-engine-reboot-machine
      machineHealthIssueName: cloud-workflow-engine-machine-health-issue
    managedByLabelValue: cloud-workflow-engine
    metricsListenAddress: 127.0.0.1:9090
    supportedInfraProviders: ["nico"]
    tracing:
      enabled: false
      endpoint: ""
      insecure: true
      serviceName: cloud-workflow-engine
    certRenewal:
      controllerEnabled: true
      interval: 12h
      thresholdDays: 30
    kubeadmImageRepository: registry.k8s.io
    cniImageRegistry: ""
    installCNIAfterInit: false
    nicoControllersEnabled: true
    profilesDir: ./profiles
    runbooksDir: ""
    tenantAccessEnabled: false
    nodeAgent:
      spireAttestationMethod: tpm
      dsxImds:
        endpoint: http://169.254.169.254
        pluginChecksum: ""
        operatorTenant: ""
      tlsJWT:
        enabled: false
        issuer: ""
        jwksURI: ""
        algorithms: []
        subjectTrustDomain: ""
        dsxIdentityExchangeURL: ""
        subjectTenant: ""
        subjectService: ""
      externalMTLSEnabled: false
    oidc:
      managementOIDCAuth:
        enabled: true
        issuerURL: https://spire.example.com
        discoveryURL: https://spire.example.com/.well-known/openid-configuration
        caBundleSecretRef:
          namespace: nke-system
          name: nke-oidc-ca-bundle
          key: ca.pem
      providers:
        - name: corporate
          issuerURL: https://login.example.com/tenant
          # 可选：当租户 API 服务器无法访问 issuerURL 本身时，
          # 它们从何处获取发现文档。
          discoveryURL: https://login.internal.example.com/tenant/.well-known/openid-configuration
          audiences: ["nvke-tenant-cluster"]
          usernameClaim: sub
          usernamePrefix: "corporate:"
          groupsClaim: groups
          groupsPrefix: "corporate:"
          requiredClaims:
            tenant_id: acme
          caBundle: |
            -----BEGIN CERTIFICATE-----
            ...
            -----END CERTIFICATE-----
    temporal:
      hostPort: localhost:7233
      namespace: cloud-workflow-engine
      taskQueue: cloud-workflow-engine-tasks
      tls:
        caCert: ""
        clientCert: ""
        clientKey: ""
        serverName: ""
    spire:
      spiffeEnabled: false
      spiffeEndpointSocket: unix:///tmp/spire-agent/agent.sock
      spiffeJWTExchangeEndpoint: ""
      serverNamespace: spire-server
      className: spire-mgmt-spire
      tpmPubhashesConfigMapName: tpm-pubhashes
      serverPort: "9091"
      joinTokenTTL: 1h
    teleport:
      enabled: false
      identityFile: ""
      proxyAddress: ""
      environment: non-prod
      managedByLabelValue: nke-cloud-workflow-engine
      tokenNamePrefix: nke-cwe-
    permissiveMode: false
```

八个键 `fieldManager`、`tracing`、`certRenewal`、`nodeAgent`、`temporal`、`spire`、`teleport` 和 `oidc` 将子系统的字段分组在一个名称下，而不是平铺在顶层。`tracing`、`teleport` 和 `nodeAgent` 镜像 `Config` 已有的嵌套结构体（`TracingConfig`、`TeleportConfig` 以及带有自己的 `tlsJWT` 和 `dsxImds` 块的 `NodeAgentConfig`）；`NodeAgentExternalMTLSEnabled` 仍然是扁平的 `Config` 字段，但它共享 `NODE_AGENT_` 前缀，因此也放在 `nodeAgent` 中。其他五个没有结构体支持：其中四个，`Config` 将字段保持扁平并共享名称前缀（`FieldManager*`、`CertRenewal*`、`Temporal*`、`Spire*`/`Spiffe*`），而 `oidc` 将共享提供商和新的集群级列表并排放置。Schema 仍然对它们进行分组，出于同样的可读性原因：`supportedInfraProviders` 旁边的扁平 `providers` 键读起来像"什么的提供商"，而十几个不相关键旁边的扁平 `className` 不如 `spire.className` 表达得多。`Config` 中在三个或更多字段中重复的每个前缀都得到了这种处理；仅由一两个字段共享的前缀（`managedByLabelValue`、`supportedInfraProviders`、`tenantAccessEnabled`、`permissiveMode`）保持扁平，因为分组单个字段没有好处。`oidc.managementOIDCAuth` 一对一替换 `NKE_OIDC_AUTH_*` 环境变量（`Enabled` 变为 `enabled`，等等），包括将其 CA 包保留为 Secret 引用而非内联 PEM，与今天的行为一致，并让它继续通过已经管理该 Secret 的任何机制轮换。`oidc.providers` 是本设计添加的新集群级列表；每个条目为其自己的 CA 包使用内联 PEM 而非 Secret 引用，因为 CA 包是公共证书材料，通过 Secret 间接引用没有什么可保护的，要求操作员为每个提供商管理 Secret 也没有好处。每个条目还可以设置可选的 `discoveryURL`，正如 `oidc.managementOIDCAuth` 已经可以的那样，用于租户 API 服务器无法在其自己的 URL 访问的签发者，例如位于拆分 DNS 后面或管理平面内部的签发者：kube-apiserver 然后从那里获取发现文档，而令牌仍然携带 `issuerURL` 作为其 `iss` 声明。所有其他键，无论是否分组，都是现有的 `Config` 字段，从其 `SCREAMING_SNAKE_CASE` 环境变量重命名为其底层已有的 camelCase Go 字段；它们不需要仅仅因为现在位于同一文档中而看起来相似。

`oidc.providers` 没有按集群的方面。操作员通过同名 chart 值设置一次列表，从那时起它适用于 NVKE 在 Kubernetes 1.34 或更高版本上创建的每个集群（参见 **Kubernetes 版本下限**）。它最多接受 5 个条目：`AuthenticationConfiguration` 总共最多支持 64 个 JWT 认证器，其中少数保留给平台管理的提供商，包括共享提供商。

我们选择 ConfigMap 而非 CRD。CRD 会免费为我们提供准入时 schema 验证，而且 NVKE 已经在其自己的管理集群上管理其他 CRD，因此它不会是新的能力。但它带来了本设计不需要付出的实际成本：Helm 的 CRD 故事很尴尬（单独的 `crds/` 目录只安装从不升级，没有干净的方法在同一发布中应用自定义资源及其自己的 CRD），而且此 schema 以后获得的每个字段都意味着要考虑 CRD 版本控制。ConfigMap 避免了所有这些。它失去了准入时验证，chart 也没有添加 `values.schema.json` 来替代它。相反，控制器的加载器是严格的：它拒绝未知或拼写错误的键、类型错误的值或验证失败的文档，并拒绝启动而不是在部分配置上运行。在 Helm 中重述这些规则会留下两个需要保持同步的 schema，这就是 chart 的 `nke-auth-config` 模板已经给出的不这样做的原因。CI 检查渲染 chart 并通过同一加载器运行结果，因此破坏文档的 chart 更改在合并前就会失败。权衡是操作员的格式错误值在控制器重启时而非 `helm upgrade` 时出现，这就是 `nke-auth-config` 今天的行为方式。

这是一扇双向门，而不是单向门：NVKE 如何解析、验证或应用此配置，都不依赖于它位于 ConfigMap 而非 CRD 或未来用于托管 Kubernetes 的某些操作员 API 中。如果其中某个后来被证明是更好的长期归宿，以后移动它是廉价的。

控制器不通过 Kubernetes API 获取此 ConfigMap。它将其作为卷挂载，与 chart 今天已经为 API bearer-token 认证挂载 `nke-auth-config` 的方式相同。这免费为我们带来两件事，完全匹配该现有先例：

- **没有启动顺序风险。** Kubernetes 在其引用的 ConfigMap 实际存在之前不会启动控制器的容器，因此 `client.Get` 会提出的"它到了吗"问题永远不会出现；kubelet 阻塞挂载，而不是控制器自己的代码，直到它存在。`nke-auth-config` 自己的模板已经带有此注释："缺失的 ConfigMap 使 Pod 处于 `ContainerCreating`...而不是在未配置的情况下启动它。"
- **没有新的 RBAC 授权。** Kubelet 使用自己的凭据获取 ConfigMap，而不是 Pod 的 ServiceAccount 令牌。

第二点不扩展到共享提供商的 CA 包 Secret，控制器仍然通过 API 实时获取，与今天的方式相同，因为它在 ConfigMap 内按名称引用而不是直接挂载。

控制器在启动时读取挂载的文件一次，与 REST 服务器、Temporal 注册和 kubeadm 活动今天已经依赖的一次性读取相同（参见迁移），因此后续 `helm upgrade` 只在控制器重启并再次读取文件后生效。chart 已经为此提供了机制，同样匹配 `nke-auth-config`：一个 `configmap.reloader.stakater.com/reload` 注解，使 [Stakater Reloader](https://github.com/stakater/Reloader) 在命名 ConfigMap 的内容更改时重启 Deployment。`nke-controller-config` 获得相同的注解，原因相同。

具体来说，`deployment.yaml` 获得一个卷、一个挂载、挂载文件的路径和一个重载注解，遵循 `nke-auth-config` 已经使用的确切形状：

```yaml
metadata:
  annotations:
    # 现有的 $reloadConfigMaps 列表在 nke-auth-config 旁边增加 nke-controller-config。
    configmap.reloader.stakater.com/reload: "nke-auth-config,nke-controller-config"
spec:
  template:
    spec:
      containers:
        - env:
            - name: CONTROLLER_CONFIG_PATH
              value: /etc/nke/controller-config/config.yaml
          volumeMounts:
            - name: nke-controller-config
              mountPath: /etc/nke/controller-config
              readOnly: true
      volumes:
        # 非可选：缺失的 ConfigMap 使 Pod 处于 ContainerCreating，
        # 其名称出现在事件中，而不是在未配置的情况下启动它。
        - name: nke-controller-config
          configMap:
            name: nke-controller-config
```

NVKE 将 `oidc.providers` 合并到今天已经携带共享提供商的同一 `AuthenticationConfiguration` 中：每个控制平面节点上的同一文件、同一位置。一个复杂情况：磁盘上的文件可能随时更改，按操作员自己的时间表，与任何特定集群的生命周期无关。因此，当创建集群时，NVKE 将加载文件中的提供商固定到新集群的数据库行上，在创建它的同一插入中，之后没有任何东西更新它们。每个控制平面节点，即使是很久以后加入集群的节点，都从该行上固定的提供商构建其 `AuthenticationConfiguration`，而不是从该较晚时刻文件所说的内容。否则同一 VIP 后面的两个控制平面节点可能强制执行不同的认证规则。只有集群级列表被固定：共享提供商保持今天的行为，每当 NVKE 渲染文件时从实时配置读取。`EnableOIDC` 推广从相同的输入构建，因此以后开启共享提供商保留集群的固定提供商。

操作员配置的每个提供商（不是共享的）都携带操作员无法移除的 CEL 规则：拒绝任何以 `system:`、`kubeadm:`、`nke:`、`nvke:`、`nvidia:`（`controllerconfig.ReservedIdentityPrefixes()` 持有的列表，它是权威的）或 SPIFFE 信任域（`spiffe://<trust domain>/`）开头的用户名或组，以及任何与 NVKE 自己的交换身份之一完全匹配的用户名或组。这针对 *最终* 映射值运行，在加前缀之后，因此它也捕获烦人的边缘情况：空前缀加上 `system:masters` 的声明，或像 `sys` + `tem:masters` 这样的前缀