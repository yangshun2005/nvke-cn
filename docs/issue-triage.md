# Issue 分诊

分诊是维护者对 Issue 做出的第一个决定：它是否可处理、归属何处、有多紧急。决定以标签形式记录，任何人都能看到哪些在等待、哪些已决定。

## 标签

| 标签 | 含义 | 由谁设置 |
|---|---|---|
| `needs-triage` | 尚无维护者设置优先级。报告者无需做任何事。 | Issue 表单，以及自动化对任何无优先级的开放 Issue |
| `priority/P0` | 数据丢失、集群或节点不可用、安全问题，或无变通方案。30 个日历日内修复。 | 分诊者 |
| `priority/P1` | 影响重大但有变通方案。183 个日历日内修复。 | 分诊者 |
| `priority/P2` | 其他所有情况。不承诺修复。 | 分诊者 |
| `needs-info` | 轮到报告者：缺少细节，无法分诊。 | 分诊者 |

设置优先级即完成分诊。存在 `priority/*` 标签时，自动化会移除 `needs-triage`；最后一个优先级标签被移除时，又会加回。其他操作不会清除它。带 `needs-info` 的 Issue 在获得优先级前会一直保留 `needs-triage`。

在 P0 和 P1 之间犹豫时，选 P1。功能请求和文档请求也要定优先级；多数为 P2。

其他 NVIDIA 仓库使用相同的标签和含义，因此优先级在各仓库间读法一致。

## 谁来做分诊

拥有仓库 Triage 角色或更高权限的人均可设置这些标签。[MAINTAINERS.md](../MAINTAINERS.md) 中的维护者和领域审查者应当参与；参见 [GOVERNANCE.md](../GOVERNANCE.md)。工程团队分诊 bug，产品团队分诊功能请求，维护者分诊文档请求，如 [CONTRIBUTING.md](../CONTRIBUTING.md) 所述。

## 检查清单

对每个带 `needs-triage` 的 Issue：

1. 确认在范围内且非重复。重复的关闭并附上原 Issue 链接；超出范围的，用一句话说明原因后关闭。
2. 检查类型标签。表单会设置 `bug`、`feature` 或 `documentation`；未通过表单创建的 Issue 可能没有。
3. 检查领域标签。自动化会尽量根据表单的 Component 字段设置一个。报告者猜错就改正，缺失就补上。
4. 阅读报告者填写的影响和讨论串，然后设置一个 `priority/*` 标签。影响下拉框是证据，不是优先级。
5. 若暂时无法分诊，添加 `needs-info` 并说明缺什么，而不是设置优先级。
6. 指派处理人，或留给贡献者并添加 `help wanted` 或 `good first issue`。

## 节奏

每个新 Issue 应在七个日历日内获得优先级。各领域每周清扫一次队列；项目负责人清扫未路由队列。

- 所有等待中：[`is:open is:issue label:needs-triage`](https://github.com/dsx-ai-factory/nvke/issues?q=is%3Aopen+is%3Aissue+label%3Aneeds-triage)
- 单个领域，例如 NVKE controller：[`is:open is:issue label:needs-triage label:area/controller`](https://github.com/dsx-ai-factory/nvke/issues?q=is%3Aopen+is%3Aissue+label%3Aneeds-triage+label%3Aarea%2Fcontroller)
- 未路由：[`is:open is:issue label:needs-triage -label:area/controller -label:area/agent -label:area/cli -label:area/chart -label:area/os-image -label:area/ci-build -label:area/docs -label:area/deps -label:area/nktp -label:area/generated`](https://github.com/dsx-ai-factory/nvke/issues?q=is%3Aopen+is%3Aissue+label%3Aneeds-triage+-label%3Aarea%2Fcontroller+-label%3Aarea%2Fagent+-label%3Aarea%2Fcli+-label%3Aarea%2Fchart+-label%3Aarea%2Fos-image+-label%3Aarea%2Fci-build+-label%3Aarea%2Fdocs+-label%3Aarea%2Fdeps+-label%3Aarea%2Fnktp+-label%3Aarea%2Fgenerated)
- 等待报告者：[`is:open is:issue label:needs-info`](https://github.com/dsx-ai-factory/nvke/issues?q=is%3Aopen+is%3Aissue+label%3Aneeds-info)

## 给报告者

你无需对 `needs-triage` 做任何事。若要争取更高优先级，请评论说明影响：什么坏了、影响多少集群或节点、是否有变通方案。分诊者会在更改已设优先级前阅读讨论串。

## 自动化做什么

[`issue-triage-check.yml`](../.github/workflows/issue-triage-check.yml) 在 Issue 事件触发时及每小时运行一次：

- 为任何无 `priority/*` 标签的开放 Issue 添加 `needs-triage`，设置后移除；
- 对新 Issue，将表单的 Component 字段映射为 `area/*` 标签，仅一次，且绝不恢复分诊者已移除的领域标签。

Pull Request 不携带这些标签。它们由基于路径的标签器路由，并通过 CODEOWNERS 审查。