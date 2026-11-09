# 使用 NVKE Agent 的升级流程

节点上升级流程的编排由调用方负责，Node Agent 提供分组操作以辅助该过程。

## 流程
1. 先升级主控制平面节点
2. 升级其他控制平面节点
3. 升级工作节点

## 控制平面节点

务必遵循流程，在尝试更新任何工作节点之前先更新所有控制平面节点。在继续处理集群中的下一个节点之前，必须确认每个节点已成功完成升级步骤。

### 控制平面主节点

1. UpdateKubeadmPackage({NewVersion: "v1.29.9"})
2. DrainNode({NodeName: <name-of-node-being-upgraded>}) *始终在主节点上执行
3. UpdateKubeletPackage({NewVersion:" v1.29.9"})
4. UncordonNode({NodeName: <name-of-node-being-upgraded>}) *始终在主节点上执行

对集群中每个其他控制平面节点重复步骤 1-4，注意 DrainNode 和 UncordonNode 调用必须在控制平面主节点上执行！

## 工作节点

在控制平面节点成功升级之前，不应升级工作节点！
建议逐个升级工作节点，以确保集群中运行的工作负载不会因容量不足而受影响。

升级工作节点的步骤与控制平面节点相同：
1. UpdateKubeadmPackage({NewVersion: "v1.29.9"})
2. DrainNode({NodeName: <name-of-node-being-upgraded>}) *始终在主节点上执行
3. UpdateKubeletPackage({NewVersion:" v1.29.9"})
4. UncordonNode({NodeName: <name-of-node-being-upgraded>}) *始终在主节点上执行

## 监控升级进度

Kubeadm 和 Kubelet 的升级过程是后台进程。可以通过以下几种方式监控这些操作的进度：
1. 观察 NodeStatus 对象中的 "UpgradePhase" 和 "UpgradeStatus" 字段
2. 观察 "GetNodes" 响应，等待节点以正确版本进入 Ready 状态