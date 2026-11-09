# ExecuteNodePreflight — 节点就绪检查

为 NVKE agent 新增的 `ExecuteNodePreflight` gRPC RPC 设计（2026 年 5 月）。

## 问题

集群配置工作流需要知道节点何时真正准备好执行 kubeadm 和包操作——而不仅仅是 agent 进程何时开始监听。

NVKE 控制器首先以 agent 可达性作为门槛（例如通过 `GetAgentVersion` / NodeAgentReady 条件）。这是必要的但不充分：agent 启动后 cloud-init 可能仍在运行 `runcmd` 步骤（包安装、sysctl、服务重启）。过早继续会导致 kubeadm 预检不稳定失败或节点配置不完整。

我们需要一种远程方式，在单次 agent 调用中断言引导已完成（并可选运行 kubeadm 预检）。

## 方案

`ExecuteNodePreflight` 执行：

1. **可选的 cloud-init 检查** — 在请求中设置 `check_cloud_init_status`。
2. **可选的文件检查** — 调用方在 `checks[]` 中定义的 `file_exists` 断言。
3. **可选的 kubeadm 预检** — 内嵌 `kubeadm init|join phase preflight`。

`ExecuteKubeadmPreflight` 保留为独立 RPC，供仅需 kubeadm 检查的调用方使用。两个 RPC 共享 `runKubeadmPreflight` 辅助函数。

## API

### 请求（`ExecuteNodePreflightRequest`）

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `check_cloud_init_status` | `CheckCloudInitStatus` | 设置后，agent 运行 `cloud-init status` |
| `checks` | `repeated NodePreflightCheck` | 可选的 `file_exists` 检查 |
| `kubeadm_preflight` | `KubeadmPreflightRequest` | 可选的 kubeadm init/join 预检 |

`check_cloud_init_status`、`checks`、`kubeadm_preflight` 至少需设置一项。

```protobuf
message CheckCloudInitStatus {}

message NodePreflightCheck {
  string name = 1;  // optional; defaults derived from check type/path
  oneof check {
    FileExistsPreflightCheck file_exists = 2;
  }
}

message FileExistsPreflightCheck {
  string path = 1;  // must be absolute
}
```

### `check_cloud_init_status` 行为

agent 运行 `cloud-init status` 并解析输出：

| 输出 | Agent `status` | 调用方操作 |
|--------|----------------|---------------|
| 包含 `status: done` | `ready` | 继续 |
| 包含 `status: running` | `notready` | 轮询 / 重试 |
| 包含 `status: error` | `error` | 失败（永久） |

结果名称始终为 `cloud-init-status`。

### 响应（`ExecuteNodePreflightResponse`）

| 字段 | 描述 |
|-------|-------------|
| `status` | 聚合结果（见下文） |
| `results` | 各检查结果（`name`、`passed`、`message`） |
| `kubeadm_preflight` | 请求时返回完整的 `KubeadmPreflightResponse` |
| `error_message` | 状态非 `ready` 时的人类可读摘要 |

**状态值**

| 状态 | 含义 |
|--------|---------|
| `ready` | 所有请求的检查均通过 |
| `notready` | 节点检查暂时性失败（如 cloud-init 仍在运行） |
| `error` | 请求无效、I/O 失败、cloud-init 错误或 kubeadm 预检错误 |

### 错误处理约定

与 `ExecuteKubeadmPreflight` 一致，校验和检查失败返回 **gRPC OK**，并设置 `status = "error"`（或 `"notready"`）和 `error_message`——而非 gRPC 错误状态。这样编排逻辑保留在响应体中，避免 Temporal 对预期的"尚未就绪"结果进行重试循环。

只有意外的 agent 故障才以 RPC 错误形式传播。

## 执行流程

```
Caller (controller activity)
        │
        ▼
ExecuteNodePreflight
        │
        ├─► If check_cloud_init_status set
        │       └─► cloud-init status
        │
        ├─► For each check in checks[]
        │       └─► file_exists → os.Stat(path)
        │
        └─► If kubeadm_preflight set
                └─► runKubeadmPreflight()
        │
        ▼
Aggregate status + error_message
```

**状态聚合规则**

- 初始为 `ready`。
- 任一节点检查失败 → `notready`（检查仍会执行完毕）。
- Kubeadm 预检 `error` → 顶层 `error`（覆盖 `notready`）。

## 安全与路径处理

`file_exists` 路径必须为**绝对路径**。当设置了 `NodeAgentServer.RootPath`（单元测试）时，路径在该根目录下解析，若逃逸则拒绝。生产 agent 将 `RootPath` 留空，直接 stat 真实文件系统。

## 使用示例

**默认（NVKE 控制器对典型节点发送的内容）：**

```json
{
  "check_cloud_init_status": {}
}
```

**旧版哨兵文件**（执行 `touch /run/cloud-init.done` 的配置）：

```json
{
  "checks": [
    {
      "name": "sentinel",
      "file_exists": { "path": "/run/cloud-init.done" }
    }
  ]
}
```

**Cloud-init + kubeadm init 预检：**

```json
{
  "check_cloud_init_status": {},
  "kubeadm_preflight": {
    "config": "<rendered kubeadm ClusterConfiguration YAML>",
    "role": "INIT"
  }
}
```

`grpcurl` 示例见主 [README](../README.md#executenodepreflight-grpcurl)。

## NVKE 控制器集成

控制器从 `controller/internal/temporal/activities/node_preflight.go` 中的 `NodePreflight` Temporal activity 调用此 RPC。

**每个节点的执行顺序**

1. **`GetAgentVersion`** — 确认 agent 进程可达并返回其版本。
2. **版本门槛** — 低于 `v0.1.24` 的 agent 跳过严格预检（不支持 `CheckCloudInitStatus`），将可达性视为充分条件。
3. **`ExecuteNodePreflight`** — 通常带 `check_cloud_init_status: {}`（CAPD/docker-machine 节点可能改用哨兵 `file_exists` 检查；见该 activity 中的 `nodePreflightRequestForNode`）。

轮询、超时，以及 cloud-init 长时间 `status: running` 结果被视为软失败还是硬失败，均由控制器 activity / 工作流层负责，而非 agent。

## 文件

| 文件 | 作用 |
|------|------|
| `pkg/agent/proto/node_agent.proto` | RPC 和消息定义 |
| `internal/agent/preflight.go` | `ExecuteNodePreflight` 及检查实现 |
| `internal/agent/agent.go` | 共享的 `runKubeadmPreflight` |
| `internal/agent/preflight_test.go` | 单元测试 |
| `controller/internal/temporal/activities/node_preflight.go` | 控制器调用方 |

## 范围之外（后续工作）

- `check_cloud_init_status` 和 `file_exists` 之外的额外检查类型
- 轮询/重试策略（属于编排器，而非 agent）
- 弃用 `ExecuteKubeadmPreflight`（暂时保留）