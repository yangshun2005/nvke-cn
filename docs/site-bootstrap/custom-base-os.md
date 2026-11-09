# 自定义基础操作系统

如果需要在**原生 Ubuntu 镜像**（而非预构建的 NVKE 节点镜像）上引导 tier-0 集群，可以在 cloud-init 配置中添加必要的安装步骤。

开始前，先编写[站点配置文件](configuration.md)，以便 cloud-init 将其放置到初始化节点上。

引导主机还需要在 `PATH` 中提供 `nerdctl`（含 containerd）、`kind`、`helm`、`kubectl` 和 `clusterctl`——参见[安装](installation.md#additional-prerequisites-for-bootstrap)。`nvkectl` 部署的两个镜像来自镜像仓库，分别由 `--nvke-controller-image` 和 `--provisioned-provider-image` 指定，因为新安装的主机没有本地构建可供旁加载。示例只标出工具安装的位置，不提供安装脚本；具体命令取决于你的镜像。

## Cloud-Init 示例

以下是在自定义基础操作系统上引导 tier-0 集群的完整 cloud-init 配置。它会安装 Kubernetes 组件和 `nvke-agent`，并在首次启动时运行引导。它不安装 `nvkectl`：请从源码构建，并在首次启动前将其放到引导主机的 `PATH` 中，例如放入你的镜像。

```yaml
#cloud-config
users:
  - default
  - name: nvke-admin
    gecos: NVKE Admin User
    sudo: ALL=(ALL) NOPASSWD:ALL
    groups: root
    lock_passwd: true
autoinstall:
  version: 1
  identity:
    hostname: <node-name>
    password: $6$...  # hashed password
    username: nvke-admin
  packages:
    - chrony
    - jq
  ntp:
    enabled: true
    ntp_client: chrony
  keyboard:
    layout: us
    toggle: null
    variant: ""
  locale: en_US
  ssh:
    allow-pw: true
    authorized-keys: []
    install-server: true
  late-commands:
    - curtin in-target --target=/target -- sed -i 's|^GRUB_CMDLINE_LINUX=.*|GRUB_CMDLINE_LINUX="ipv6.disable=1 console=ttyS1,115200"|' /etc/default/grub
    - curtin in-target --target=/target -- update-grub
  user-data:
    write_files:
      # nvkectl site spec file (bootstrap host only — omit on other nodes)
      - path: /etc/nvkectl/sitespec.yaml
        owner: root:root
        permissions: '0640'
        content: |
          clusterName: "nvke-site"
          cluster:
            nodes:
              - name: "ctrl-1"
                ip: "10.0.0.10"
              - name: "ctrl-2"
                ip: "10.0.0.11"
              - name: "ctrl-3"
                ip: "10.0.0.12"
            k8sVersion: "1.36.3"
            cniName: "ovn-kubernetes"
            podCidr: "192.168.0.0/16"
            serviceCidr: "10.96.0.0/12"
            nodeAgentPort: "15005"
    runcmd:
      # Expand root LVM volume
      - [sh, -c, "lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv"]
      - [sh, -c, "resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv"]
      - [sh, -c, "mkdir -m 755 /etc/apt/keyrings"]
      - [sh, -c, "sysctl -p"]
      - [sh, -c, "systemctl restart procps"]

      # Install Kubernetes components (replace v1.34 with your target minor version)
      - [sh, -c, "curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg"]
      - [sh, -c, "echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | tee /etc/apt/sources.list.d/kubernetes.list"]
      - [sh, -c, "apt-get update"]
      - [sh, -c, "apt-get install -y kubelet kubeadm kubectl"]
      - [sh, -c, "apt-mark hold kubelet kubeadm kubectl"]

      # Known limitation at preview: the NVKE apt repository is not published, so
      # the key download and apt-get commands below fail. Instead, build the
      # .deb from a checkout (make -C agent deb-amd64) and install it from a
      # local path.
      # Install nvke-agent from the NVKE apt repository
      - [sh, -c, "curl -fsSL -o /etc/apt/keyrings/nke.asc https://packages.nvidia.com/nvke/nvke-apt-public.key"]
      - [sh, -c, "echo 'deb [signed-by=/etc/apt/keyrings/nke.asc] https://packages.nvidia.com/nvke/apt nvke-noble main' | tee /etc/apt/sources.list.d/nke-apt.list"]
      - [sh, -c, "apt-get update"]
      - [sh, -c, "apt-get install -y nvke-agent"]
      - [sh, -c, "echo 'AGENT_ARGS=\"--disable-mtls --secure-server-port 15005\"' > /etc/default/nvke-agent"]
      - [sh, -c, "systemctl restart nvke-agent"]

      # Bootstrap host only: install nerdctl + containerd, kind, helm, kubectl and
      # clusterctl here. nvkectl checks for nerdctl before it starts; a missing kind
      # fails pre-flight, and helm, kubectl or clusterctl fail mid-bootstrap.

      # nvkectl must already be on PATH (bootstrap host only). It is not
      # published as a download and this config does not install it: build it
      # from a checkout with `make -C cli build`, cli/v0.2.0 or later (earlier
      # releases do not take --nvke-controller-image), and put cli/bin/nvkectl
      # in the host's image.

      # Run bootstrap (bootstrap host only). --disable-mtls matches the plaintext
      # agents configured above; this is the development-only mode, not a
      # production path. The two images are named by flag because this host has
      # no local build to side-load; nvkectl pulls them with nerdctl.
      - [sh, -c, "nvkectl bootstrap site -f /etc/nvkectl/sitespec.yaml --nvke-controller-image <controller-image> --provisioned-provider-image <provisioned-provider-image> --disable-mtls --encryption-key-out /root/nvke-site-etcd-encryption-key.json >> /var/log/nvkectl-bootstrap.log 2>&1"]
```

引导过程会将集群的 etcd 加密密钥写入 `--encryption-key-out` 指定的路径。引导主机是集群节点之一，因此该文件位于密钥所保护的磁盘上：运行结束后取出该文件，并从主机上删除。参见[预配置节点](pre-provisioned-nodes.md#step-2-run-bootstrap-on-init-node)。

## 关键组件

- **Kubernetes 安装**：添加 Kubernetes apt 仓库并安装 kubelet、kubeadm 和 kubectl。将包标记为保持，防止自动升级。
- **nvke-agent**：从 NVKE apt 仓库安装；为预配置节点引导配置 `--disable-mtls` 和 `--secure-server-port 15005`。
- **nvkectl**：仅安装在**引导主机**上，连同其容器运行时和 CLI 依赖。运行时使用 `--disable-mtls` 以匹配 agent，并设置两个镜像参数。其他节点上不要安装 `nvkectl`，也不要运行引导命令。

> **明文模式仅用于开发。** 两端都启用 `--disable-mtls` 时，没有 SPIRE，也没有证明。如需带证明的路径——每个节点上的 `spire-agent`、每台主机的 TPM EK 指纹、来自 `NVKECTL_SPIRE_CA_CERT_FILE` 和 `NVKECTL_SPIRE_CA_KEY_FILE` 的 SPIRE CA——请遵循[你的第一个集群，真实站点](../first-cluster.md#real-site)。

## 引导主机与其他节点

| | 引导主机 | 其他节点 |
|---|---|---|
| Kubernetes 包 | ✓ | ✓ |
| nvke-agent | ✓ | ✓ |
| nerdctl、kind、helm、clusterctl | ✓ | ✗ |
| `/etc/nvkectl/sitespec.yaml` | ✓ | ✗ |
| `nvkectl` 二进制 + 引导运行 | ✓ | ✗ |

在其他节点上，省略 `write_files` 部分和最后一行 `runcmd`（引导运行）。

## 自定义

- 用实际的节点名称和 IP 更新 `cluster.nodes`
- 通过密钥管理让引导主机登录镜像仓库（`nerdctl login nvcr.io`）；`nvkectl` 使用 nerdctl 拉取这两个镜像
- 将 Kubernetes 仓库的次版本更新为与站点配置中的 `cluster.k8sVersion` 一致
- 根据你的环境更新主机名、密码和 SSH 授权密钥

> 安装 Kubernetes 包前**确保 `/etc/apt/keyrings` 存在**——上面的 `mkdir` 步骤会创建它。

引导日志位于引导主机的 `/var/log/nvkectl-bootstrap.log`。