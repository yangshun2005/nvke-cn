

> **[中文化] nvke**
>
> 此项目是 [dsx-ai-factory/nvke](https://github.com/dsx-ai-factory/nvke) 的中文翻译版本。
> - 原项目 Stars: 984
> - 主语言: Go
> - 许可证: Apache-2.0
> - 翻译日期: 2026-11-09
> - 原始 README: [README_en.md](README_en.md)
>
> 说明：下方 README 正文为**原项目文档的中文翻译**，力求忠于原文；本页顶部的「中文快速上手」为**面向中文用户的补充导读**（非原文内容），目的是让你不用读完整篇 README 就能先把项目跑起来。
>
> 如有翻译不准确之处，欢迎提 Issue 或 PR。

---


## 🚀 中文快速上手（补充导读，非原文）

### 这是什么 / 解决什么问题

NVKE（NVIDIA Kubernetes Engine）是一个让你在自己的 NVIDIA GPU 基础设施上运行托管 Kubernetes 服务的项目。它提供管理平面，平台团队安装后可以为客户创建和运维各自独立的 Kubernetes 集群（每个工作负载集群有自己的控制平面、etcd 和节点池）。它解决的是「在自有 GPU 机器上批量、隔离地交付 Kubernetes 集群」这件事，而不是一个 Kubernetes 发行版或 Cluster API 的变体。

### 快速上手（最短路径）

1. **只想评估、没有 GPU 硬件**：按 README 指引阅读 `docs/get-started/quickstart-local.md`，在模拟机器上跑起开发用管理平面和一个工作负载集群，无需 NVIDIA 账号。
2. **克隆仓库**：
   ```bash
   git clone https://github.com/dsx-ai-factory/nvke.git
   cd nvke
   ```
3. **从源码构建**（需要 Go 1.27.3 和 GNU Make 4+）：
   ```bash
   make build
   ```
   构建会在 `bin/` 下产出九个二进制文件（`agent`、`updater`、`controller`、`nvkectl` 等），但**不会**启动管理平面或创建集群。
4. **真实安装**：阅读 `docs/operate/plan-installation.md` 了解准备节点、引导管理集群、安装服务的顺序；引导管理集群见 `docs/operate/bootstrap-management-cluster.md`，安装 NVKE 服务见 `deploy/README.md`。
5. **本地开发环境**（可选）：`docs/getting-started.md` 给出各模块的构建、lint、测试命令，例如 `make -C controller build`。

### 环境要求

- **操作系统**：本地快速上手需 Linux 主机或 VM（amd64 或 arm64）；macOS 用户需通过 `docs/get-started/local-site.md` 在 Colima 虚拟机内运行评估。节点 agent 只在 Linux 上运行。
- **硬件（本地快速上手）**：至少 6 CPU、12 GiB 空闲内存、100 GiB 空闲磁盘，并需要虚拟 TPM 支持。
- **依赖**：Docker；构建需 Go 1.27.3（`docs/getting-started.md` 写 Go 1.26+，以 `go.work` 固定版本为准）和 GNU Make 4.0+；Flox 可选，用于固定完整工具链。
- **真实安装**：需自备机器、网络、节点镜像和镜像仓库，并配置对 NVIDIA Infra Controller (NICo) 的访问——预览阶段 NICo 是唯一兼容的基础设施提供方。
- **网络**：首次构建需能访问公共 Go module proxy；国内环境可能需要更换 Go 模块镜像源。

### 常见坑与注意事项

- **预览状态**：NVKE 目前是 preview，API、配置和行为可能随版本变化，官方不建议用于生产负载。
- **安全默认值**：安装后 API 认证默认关闭、API 走明文 HTTP，在允许调用方接入前需按 `docs/operate/api-authentication.md` 配置认证与受保护访问。
- **管理集群自身限制**：管理集群不会自我升级、扩缩容或替换，也没有管理平面升级和备份/恢复流程；FDE 节点镜像仅供评估，不提供静态数据机密性。
- **macOS 的 Make 版本**：系统自带 GNU Make 3.81 会直接报错，需 `brew install make` 后用 `gmake` 代替 `make`；绕过 `make` 直接用 `go` 构建还可能因工具链版本不匹配而 panic。

---



---

# NVIDIA Kubernetes Engine (NVKE)

[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![Go](https://img.shields.io/badge/Go-1.27.3-00ADD8.svg)](go.work)

**在你自己的 NVIDIA GPU 基础设施上运行托管 Kubernetes 服务。**
NVIDIA Kubernetes® Engine (NVKE) 提供管理平面，平台团队安装它来为客户创建和
运维 Kubernetes 集群。每个工作负载集群都有
自己的 Kubernetes 控制平面、etcd 和节点池。你运维 NVKE 管理
平面；你的客户通过 Kubernetes API 和 `kubectl` 使用他们的集群。

> **预览版：** NVKE 以预览版形式提供。API、配置和行为
> 可能在不同版本之间变化，我们不建议将 NVKE 用于生产
> 工作负载。参见[预览状态和已知限制](docs/overview/preview-status.md)。

[开始使用](#getting-started) · [文档](docs/README.md) ·
[贡献](CONTRIBUTING.md) · [获取帮助](SUPPORT.md)

## 开始使用

| 你的目标 | 从这里开始 | 你将获得 |
|---|---|---|
| **无需 GPU 硬件进行评估** | [本地快速入门](docs/get-started/quickstart-local.md) | 在模拟机器上运行开发管理平面和工作负载集群。无需 NVIDIA 账户。 |
| **安装到你的基础设施** | [规划安装](docs/operate/plan-installation.md) | 准备节点、引导管理集群并安装其服务的前提条件和步骤。 |
| **构建或贡献** | [从源码构建](#build-from-source) | NVKE 二进制文件和贡献者开发流程。 |

本地快速入门运行在 **Linux 主机或虚拟机，amd64 或 arm64** 上，至少需要
**6 个 CPU、12 GiB 空闲内存和 100 GiB 空闲磁盘**。它需要 Docker、
虚拟 TPM 支持以及指南中列出的工具。按照指南完成设置，直至
创建工作负载集群，并确认集群节点报告 `Ready`。
在 macOS 上，使用[本地站点指南](docs/get-started/local-site.md)，它在
Colima 虚拟机内运行评估。

对于真实安装，你需要提供机器、网络、节点镜像和
镜像仓库，并配置对 **NVIDIA Infra Controller (NICo)** 的访问。在预览版中，NICo 是
唯一兼容的基础设施提供者。
[平台和版本参考](docs/overview/supported-platforms.md)列出了
已验证的组合和固定依赖。

## NVKE 的功能

- **创建独立的工作负载集群。** 通过 NVKE
  API 或 `nvkectl` 请求集群；NVKE 通过 NICo 配置机器，并组建具有各自控制平面的
  Kubernetes 集群。
- **管理工作负载集群生命周期。** 创建、扩缩和删除节点池，
  升级 Kubernetes 并替换节点。参见[节点池指南](docs/workload-clusters/node-pools.md)
  了解手动步骤以及预览版在替换和升级方面的限制。
- **建立节点身份。** 节点在获得加入凭证之前，通过 SPIFFE/SPIRE 使用其
  TPM 进行证明。etcd 数据加密可按集群启用。
- **将构建与部署连接起来。** 其配套项目
  [NVIDIA Kubernetes Recipes (NVKR)](https://github.com/dsx-ai-factory/nvkr)
  从固定输入构建节点镜像和 NVKE 组件。NVKE 部署这些
  构建产物；NVKR 负责其构建和发布证据。

预览版需要运维人员显式配置：安装后 API 认证处于关闭状态，
API 使用纯 HTTP。在接纳调用方之前，请配置
[API 认证和受保护访问](docs/operate/api-authentication.md)。管理集群不会升级、扩缩或
替换自身，也没有管理平面升级或备份/恢复
流程。FDE 节点镜像仅供评估，不提供静态数据
机密性。在选择评估或安装路径之前，请阅读[已知限制](docs/overview/preview-status.md)。

## 概述

该图展示了 NVKE 产品系列、它如何位于 NICo 和你的 GPU 机器之上，以及哪些组件相互通信。

![NVKE 产品系列。NVKR 构建并签名构建产物，并将其发布到镜像仓库。运维人员从镜像仓库中的 chart 和镜像安装管理集群的组件、NVKE 管理平面、CAPNICo 和 NVKE 可观测性。管理平面通过 CAPNICo 请求机器，CAPNICo 调用 NICo，并驱动每个工作负载集群。NICo 从节点 OS 镜像配置 NVIDIA GPU 机器，这些机器成为工作负载集群的 Kubernetes 节点，每个节点在集群的 Kubernetes 控制平面下运行 nvke-agent。AICR 生成的加速运行时和 NVKE 可观测性在工作负载集群中是可选的。客户使用 kubectl 访问控制平面。](docs/assets/diagrams/product-family.svg)

NVKE 不是 Kubernetes 发行版，也不是 Cluster API 的变体。它是一个
**自托管的 Kubernetes 集群工厂**：一个一次性的种子集群
搭建起管理集群，其上的管理平面
随后按需配置并生命周期管理许多隔离的工作负载集群。

一次安装包含两种 Kubernetes 集群：

- **管理集群**运行 NVKE 管理平面和基础设施
  控制器。`nvkectl` 使用临时种子集群组建它；安装
  管理平面服务是单独的步骤。
- **工作负载集群**运行客户应用。管理平面通过 NICo、Cluster API 和节点代理
  创建和运维它们。

阅读[什么是 NVKE](docs/overview/what-is-nvke.md) 了解运维人员工作流，
阅读[架构](docs/architecture.md) 了解实现和数据流。

## 从源码构建

使用 **[go.work](go.work) 中固定的 Go 1.27.3** 版本，以及 **GNU Make 4 或更高版本**。首次构建时必须能访问
公共 Go 模块。[Flox](https://flox.dev/) 是可选的，提供
仓库的开发工具；在 macOS 上，它还提供 GNU Make 4。
参见[开发](docs/development.md)了解工具链设置。

```bash
git clone https://github.com/dsx-ai-factory/nvke.git
cd nvke
make build
```

这会构建工作区并将九个二进制文件写入 `bin/`：`agent`、
`updater`、`controller`、`nvkectl`、`capd-reboot-controller`、
`nvke-os-image-check`、`management-engine`、`tenant-api` 和 `operator-api`。构建不会启动
管理平面或创建集群。节点代理运行在 Linux 上。

[工作区指南](docs/go-workspace.md)解释了各模块和输出。
关于本地服务和测试环境，参见[本地开发设置](docs/getting-started.md)。
在提交更改之前，请遵循 [CONTRIBUTING.md](CONTRIBUTING.md) 并运行
`make preflight`；它会为你更改的模块选择检查项。

## 文档

| 任务 | 指南 |
|---|---|
| 准备节点并引导管理集群 | [引导管理集群](docs/operate/bootstrap-management-cluster.md) |
| 安装 NVKE 服务 | [从部署路径安装管理平面](deploy/README.md) |
| 构建节点镜像并提供发布构建产物 | [从 NVKR 获取 NVKE 构建产物](docs/operate/build-node-images.md) |
| 创建和管理客户集群 | [创建工作负载集群](docs/workload-clusters/create.md)、[管理节点池](docs/workload-clusters/node-pools.md) |
| 授予客户访问权限 | [访问工作负载集群](docs/workload-clusters/access.md) |
| 检查集群健康和遥测 | [观测 NVKE 及其集群](docs/operate/observe.md) |

[文档索引](docs/README.md)链接了其余指南和参考。

## 贡献和支持

欢迎贡献、缺陷报告和文档改进。
从 [CONTRIBUTING.md](CONTRIBUTING.md) 开始；在进行实质性工作之前先开一个 Issue，
以便维护者就方案达成一致。

- **问题：** [GitHub Discussions](https://github.com/dsx-ai-factory/nvke/discussions/categories/q-a)。
- **聊天：** DSX AI Factory Slack 中的 `#nvke` 频道。
  [获取邀请](https://inviter.co/dsx-ai-factory)。
- **缺陷和功能请求：** [GitHub Issues](https://github.com/dsx-ai-factory/nvke/issues)。
- **安全漏洞：** 遵循 [SECURITY.md](SECURITY.md) 并私下报告。
- **社区：** [行为准则](CODE_OF_CONDUCT.md)、[治理](GOVERNANCE.md)、
  [维护者](MAINTAINERS.md) 和 [支持](SUPPORT.md)。
- **发布版本：** [发布流程](RELEASE.md)。
- **CI 健康：** [CI 仪表板](https://dsx-ai-factory.github.io/nvke/)
  显示每个工作流最近三十次运行，每小时刷新。

## 商标

Kubernetes® 是 The Linux Foundation 在美国和其他国家的
注册商标。NVIDIA 和 NVIDIA 徽标是 NVIDIA Corporation 在美国和其他国家的
商标和/或注册商标。
其他公司和产品名称可能是其各自关联公司的商标。

## 许可证

Apache License 2.0。参见 [LICENSE](LICENSE) 和 [NOTICE](NOTICE)。

Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.