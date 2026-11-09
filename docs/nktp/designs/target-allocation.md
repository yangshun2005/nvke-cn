# 必需的目标分配

NVKE 遥测流水线（NKTP）中的抓取工作负载需要 Target Allocator。分配决定每个抓取目标归哪个 Collector 所有。

- Watchdog 将每个目标分配给一个副本，避免重复抓取。
- 大目标指标在其各副本间使用相同的归属规则。
- DaemonSet 将目标分配给同一节点上的 Collector。

禁用或移除目标分配必须使 Helm 渲染失败，并给出明确消息。为 Target Allocator 服务账号授予发现权限。Collector 保留抓取其分配端点所需的权限。

Watchdog 通过自身分配的抓取任务监控大目标 Target Allocator。大目标 Collector 抓取应用目标。