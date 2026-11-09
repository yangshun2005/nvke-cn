# 设计原则

## Collector 配置是单体式的

`values.yaml` 中的每个 OpenTelemetry Collector 配置都被视为一个单体值。Chart 完全按原样渲染它。模板不会在其中添加、删除、合并或更改任何内容。`values.yaml` 中的其他任何值也无法改变它。

这一保证确保 NVKE 遥测流水线（NKTP）不会限制集群运维人员根据自身需求配置 Collector 的能力。

OpenTelemetry Operator 在创建每个 Collector 时仍会执行其正常工作。例如，启用 Target Allocator 时，Operator 会将 `prometheus` 接收器连接到它，与任何安装中的行为一致。