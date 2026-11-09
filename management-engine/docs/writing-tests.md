# 编写控制器测试

这些测试中，终态指某个 fixture 输入已稳定下来的状态，包括 pending 和 failure 状态。Envtest 用例是该状态的快照，不是工作流脚本。由此得出四条规则。

- 使用现有的 create、update 或 delete 用例集。绝不为该 provider 特有的流程新增用例集。
- 仅当稳定后的 generation 不同时才分组用例。
- 步骤只等待用例集不变量，即当前 condition 的 `observedGeneration`、期望的 generation，或删除。NVKECluster 用例使用 `Synced` condition。Golden 断言每个字段和 condition。
- 时序、watch 和跨控制器交互用端到端测试。单个状态机状态用 envtest 用例。

## Fixture 与 Golden

Golden 状态存放位置及重新生成方式有两条规则。

- Kubernetes 和 NICo 的 golden 放在一起。`expected_objects.yaml` 覆盖 API-server 状态，`expected_nico.yaml` 覆盖 HTTP 请求和外部资源。
- `make test-update` 重新生成 golden。提交前先读生成的 diff，因为那里出现意外变更通常意味着行为变了。

## CAPNICo Generation 不变量

每个用例集稳定在已知的 generation，因此意外的 generation 是信号而非噪声。

- `NicoCluster` create 用例停留在 generation 1，因为协调只写 status。
- 已供应的 `NicoMachine` create 用例在 CAPNICo 写入 `spec.providerID` 后达到 generation 2。
- 未供应的 `NicoMachine` create 用例停留在 generation 1。
- 已供应的 `NicoMachine` spec update 达到 generation 3。

## 扩展 Fake NICo 后端

`internal/fake` 的 `Server` 是 NICo API 的自包含替身。测试用生产客户端对接 `Server.Handler()`，因此无需真实 NICo 部署。包规则是扩展这个 HTTP 接口，而不是用注入的 Go 客户端绕过序列化。三项任务覆盖几乎所有扩展。

添加端点：在 `Handler()` 中注册路由，沿用现有的“方法与路径到 handler”模式。例如 `GET /v2/org/{org}/nico/instance/{instanceID}` 映射到 `s.getInstance`。handler 写成 `*Server` 的方法：锁 `s.mu`，读取或修改状态，解锁，然后用 `writeJSON` 或 `writeError` 响应。

添加可种子状态：在 `SeedTenant`、`SeedInstanceType`、`SeedInstance`、`SeedMachine`、`SeedSite` 和 `SeedVPC` 旁添加 `Seed<Thing>(org string, thing nicosdk.<Thing>)` 方法。每个方法都加锁并克隆输入。`SeedTenant` 将租户存到 `s.tenants[org]`。其他 seed 方法将资源存到以 `resourceKey(org, id)` 为键的 map 中。在 `Server` 中添加对应的 map 字段。若 fixture 应从 YAML 种子化，在 `serverDump` 中添加 slice，并在 `SeedFromYAML` 中添加 case。

添加 golden 状态：扩展 `Server.Dump()`。它将每个资源 map 以及去重后的写请求序列化为 YAML，`expected_nico.yaml` 即与之比较。读轮询被有意排除，以免 golden 抖动。新资源类型也需在 `Dump()` 中添加 case，否则永远不会出现在 golden 中。