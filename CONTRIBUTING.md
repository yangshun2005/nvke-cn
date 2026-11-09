# 为 NVKE 做贡献

如果你有兴趣为 NVKE 做贡献，你的贡献将分为三类：
1. 你想报告 bug、功能请求或文档问题
    - 提交一个 [Issue](https://github.com/dsx-ai-factory/nvke/issues/new/choose)，描述你遇到的问题或你希望看到的改动。
    - 使用相关问题时，请使用 [GitHub Discussions Q&A](https://github.com/dsx-ai-factory/nvke/discussions/categories/q-a)，而不是 Issue。
    - 维护者按 [docs/issue-triage.md](docs/issue-triage.md) 所述对 Issue 进行分诊。工程团队审查 bug，产品团队审查功能请求，维护者审查文档请求。Issue 中报告的影响是分诊的依据，而不是分诊者设置的 `priority/P0` 到 `priority/P2` 标签。如果你认为某个 Issue 需要尽快处理，请在 Issue 下评论以通知团队。
2. 你想提出一个新功能并实现它
    - 发布你打算实现的功能，我们将讨论设计和实现方案。
    - 一旦我们确认方案可行，就按下面的[代码贡献](#代码贡献)指南开始实现。
3. 你想为某个未解决的 Issue 实现功能或修复 bug
    - 遵循下面的[代码贡献](#代码贡献)指南。
    - 如果你需要某个 Issue 的更多背景信息，请提问，我们会提供。

## 搭建开发环境

### 前置条件

- Go —— 下限是 `go.work` 中的 `go` 指令，而不是任何单个模块的 `go.mod`，本页刻意不重复具体版本号。从工作区读取：`awk '/^go /{print $2; exit}' go.work`。更新的 Go 也可以：`make` 会把 `GOTOOLCHAIN` 固定为同一个值，因为 `strcase` 随每个 Go 版本发布生成的 Unicode 表，当这些表与标准库的 `unicode.Version` 不匹配时会在启动时 panic。固定版本可确保在它没有对应表的 Go 版本上两者保持一致。你可以自行设置 `GOTOOLCHAIN` 来覆盖。
- GNU Make 4.0+（`make --version`）—— ⚠️ **macOS 自带 3.81，无法运行根目录的 `Makefile`。** 运行 `brew install make`，它会以 `gmake` 安装，然后把下文所有 `make` 都读作 `gmake`。在 shell 中设置 `alias make=gmake` 是最不容易出错的方式。各模块的 Makefile 两个版本都能用。
- [Flox](https://flox.dev/)，可选，用于固定工具链。环境位于仓库根目录，覆盖所有模块，包括 GNU Make。
- Docker（用于本地集成测试）
- 一个运行中的 Temporal 服务器（用于 controller 开发）

### 克隆并构建

```bash
git clone https://github.com/dsx-ai-factory/nvke.git
cd nvke
```

要贡献代码，请改为克隆你的 Fork。参见[从 Fork 工作](#从-fork-工作)。

**一条命令构建所有公开构建的内容：**

```bash
make build   # 构建 agent、controller、nvkectl 和辅助二进制文件
```

本仓库是一个 Go 工作区，`go.work` 把**每个模块**串联在一起。`make build` 会编译所有模块，因此你无需从各模块自己的目录构建。`controller/` 和 `controller/nico-sim` 过去因存在内部依赖而在工作区之外；#76 移除了该依赖，它们已加入工作区。

⚠️ **修改任何 `go.mod` 或 `go.sum` 后，请运行 `make work-sync`，并将 `go.work` 和 `go.work.sum` 随改动一起提交。** 没有东西会自动更新它们，工作区过期会导致 CI 失败。参见 [`docs/go-workspace.md`](docs/go-workspace.md)。

从根目录操作单个模块，这样你就不必记住自己在哪个目录：

```bash
make -C controller build test lint
make -C agent build test lint
```

每个模块都支持同样的四个动词 —— `build`、`test`、`lint`、`fmt` —— `make -C <module> help` 会列出其余命令。

### 运行测试

```bash
make test                      # 每个模块自己的测试套件
make -C controller test        # 单个模块
make -C controller test-integration   # 需要运行中的 CWE 部署
```

`make test-integration` 是集成测试套件的正确入口。它位于 `integration_test` 构建标签之后，因此在该目录直接运行 `go test` 会报告 "build constraints exclude all Go files" 并且什么都不运行。

在 macOS 上，agent 测试套件无需特殊处理：`agent/Makefile` 把 `TMPDIR` 指向 `/private/tmp`，因为默认值位于 `/var` 下，而 `/var` 是一个符号链接，会被 agent 自身的路径检查拒绝。

### 打开 Pull Request 之前

```bash
make preflight
```

它会对你实际改动的模块运行 lint 和测试，并在你的改动涉及工作区和许可证检查时加上这些检查。它未覆盖的内容参见 [`docs/development.md`](docs/development.md)。

### 本地开发等同于 CI

相同的工具、相同的版本，在本地和 CI 中运行。

**是什么。** 工具版本位于 `make/versions.mk`，工作流读取该文件而不是重复写版本号。`make show-versions` 会打印版本集合。仓库根目录的 Flox 环境固定了相同的版本，如果你不想用 Flox，每个目标都会从你 `PATH` 上的工具运行。

**为什么。** "在我机器上能跑"会浪费一次 CI 往返，去了解本地运行本可以告诉你的事情。这仅在你的工具链匹配时成立：比固定版本旧的 golangci-lint 会静默漏掉 CI 报告的问题，因此请对比 `make show-versions` 与 `make help` 显示你实际在运行的版本。

这并非处处成立，`docs/development.md` 列出了例外。最大的是 `govulncheck`，CI 用与工作区要求不同的 Go 运行它。

## 代码贡献

### Issue 优先的工作流

**提交 PR 之前先开 Issue**，适用于任何非平凡的改动（新功能、破坏性变更、重大重构）。这可以避免重复劳动，并让维护者在你投入时间编码之前对方案给出早期反馈。

对于小的 bug 修复和文档改进，没有前置 Issue 的 PR 也可以。

### 你的第一个 Issue

1. 阅读项目的 [README.md](https://github.com/dsx-ai-factory/nvke/blob/main/README.md)，了解如何搭建开发环境。
2. 找一个要处理的 Issue。最好的方式是查找 [good first issue](https://github.com/dsx-ai-factory/nvke/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22) 或 [help wanted](https://github.com/dsx-ai-factory/nvke/issues?q=is%3Aissue+is%3Aopen+label%3A%22help+wanted%22) 标签。
3. 在 Issue 下评论说明你要处理它。
4. 编码！确保更新单元测试和文档。
5. 完成后，[创建你的 Pull Request](https://github.com/dsx-ai-factory/nvke/compare)，在描述中用 `Closes #<N>` 引用该 Issue。
6. 确认 CI 通过所有[状态检查](https://help.github.com/articles/about-status-checks/)，如有需要则修复。
7. 等待维护者审查并处理反馈。
8. 批准后，项目维护者会合并你的 Pull Request。

记住，如果有任何不确定的地方，请随时在 Issue 下评论请求澄清！

### 管理 PR 标签

每个 PR 必须标明是否为破坏性变更。这用于突出用户升级时应当了解的改动。

对 NVKE 而言，"破坏性"变更是指以非向后兼容方式修改公共 API 的变更。向后兼容的变更（例如添加新的可选参数）不是破坏性的。在 PR 标题的类型后加 `!`，或添加 `BREAKING CHANGE:` 页脚来标记破坏性变更（参见[提交信息格式](#提交信息格式)）；没有对应的标签。

应用一个变更类别标签：

| 标签 | 何时使用 |
|-------|------------|
| `feature` | 新能力，或对现有功能的改进 |
| `bug` | 回归或缺陷修复 |
| `documentation` | 仅文档改动 |

### 资格认证结果

由 [Running one](docs/qualification/overview.md#running-one) 中所述的某个 `qualify-e2e` 标签启动的 Qualify e2e 运行，会在运行页面的 `plan` 卡片下方以 `results` 卡片显示测试结果。该卡片每个流一行，列出每个流的规格，并附上从该流的 JUnit 报告读取的失败信息。试运行没有 `results` 卡片。

### 资深开发者

一旦你熟悉了代码，可以查看我们[项目看板](https://github.com/orgs/dsx-ai-factory/projects)中下个版本的优先 Issue。

查看未分配的 Issue，找一个你愿意贡献的。从上面的_第 3 步_开始，在 Issue 下评论让别人知道你在处理它。如果你对 Issue 的实现有任何疑问，请在 Issue 中提问，而不是在 PR 中。

### 从 Fork 工作

把你的分支推送到自己的 Fork，并从那里打开 Pull Request，而不是从 `dsx-ai-factory/nvke` 上的分支打开。即使你对仓库有写权限，这也适用。

```bash
gh repo fork dsx-ai-factory/nvke --clone
```

这会创建 Fork，将其克隆为 `origin`，并把本仓库添加为 `upstream`。在已有的本仓库克隆中，`gh repo fork --remote` 做同样的事：你的 Fork 成为 `origin`，旧的 `origin` 成为 `upstream`。

### 分支命名

分支应遵循 `<type>/<short-description>` 模式：

| 前缀 | 何时使用 |
|--------|-------------|
| `feat/` | 新功能 |
| `fix/` | Bug 修复 |
| `docs/` | 仅文档 |
| `refactor/` | 不改变行为的代码重构 |
| `test/` | 添加或修复测试 |
| `chore/` | 维护、依赖更新、CI |

示例：`feat/add-retry-logic`、`fix/null-pointer-in-parser`

### 提交信息格式

NVKE 使用 [Conventional Commits](https://www.conventionalcommits.org/)：

```
<type>(<scope>): <short description>

[optional body]

[optional footer: Closes #<issue>]
```

类型：`feat`、`fix`、`docs`、`style`、`refactor`、`perf`、`test`、`build`、`ci`、`chore`、`revert`。类型和范围使用小写。

示例：
```
feat(api): add retry logic with exponential backoff
fix(parser): handle null pointer when input is empty
docs: update quickstart with Docker instructions
```

破坏性变更必须在页脚包含 `BREAKING CHANGE:`，或在类型后加 `!`：
```
feat!: remove deprecated v1 API endpoints

BREAKING CHANGE: The /v1/foo endpoint has been removed. Use /v2/foo instead.
```

**被检查的是 Pull Request 标题。** 本仓库使用 squash 合并，因此标题会成为 `main` 上的唯一提交主题。`pr-title` 检查会报告它，`main` 上的规则会拒绝不匹配的主题。修复被拒绝的标题只需编辑它，无需推送。没有任何东西检查你分支上的提交，因为 squash 会丢弃它们。

### 签署你的提交（DCO）

我们要求所有贡献者对其提交进行"签署"（sign-off）。这证明贡献是你的原创作品，或者你有权以相同的许可证或兼容许可证提交它。

任何包含未签署提交的贡献将不被接受。

要签署提交，在提交改动时使用 `--signoff`（或 `-s`）选项：

```bash
git commit -s -m "fix(controller): handle an empty bundle list"
```

这会在你的提交信息后追加以下内容，名称和邮箱与你的 Git 配置一致：

```
Signed-off-by: Jane Developer <jane@example.com>
```

这适用于每位贡献者，包括 NVIDIA 组织成员。已验证或已签名的提交没有豁免。

#### 开发者原创声明

签署即[开发者原创声明](https://developercertificate.org/)（DCO），与 Linux 内核和 Kubernetes 使用的机制相同。其全文：

```text
Developer Certificate of Origin
Version 1.1

Copyright (C) 2004, 2006 The Linux Foundation and its contributors.

Everyone is permitted to copy and distribute verbatim copies of this
license document, but changing it is not allowed.


Developer's Certificate of Origin 1.1

By making a contribution to this project, I certify that:

(a) The contribution was created in whole or in part by me and I
    have the right to submit it under the open source license
    indicated in the file; or

(b) The contribution is based upon previous work that, to the best
    of my knowledge, is covered under an appropriate open source
    license and I have the right under that license to submit that
    work with modifications, whether created in whole or in part
    by me, under the same open source license (unless I am
    permitted to submit under a different license), as indicated
    in the file; or

(c) The contribution was provided directly to me by some other
    person who certified (a), (b) or (c) and I have not modified
    it.

(d) I understand and agree that this project and the contribution
    are public and that a record of the contribution (including all
    personal information I submit with it, including my sign-off) is
    maintained indefinitely and may be redistributed consistent with
    this project or the open source license(s) involved.
```

如果你忘了 `-s`，最简单的修复方法是把它加到分支上的每个提交：

```bash
git rebase --keep-base --signoff main
git push --force-with-lease
```

`--keep-base` 很重要：没有它，变基会悄悄把你的分支移到你本地 `main` 指向的位置，而在 Fork 上这通常是过期的。

也允许不重写历史的补救方式，但它不只是一个空的 `-s` 提交。应用会为每个未签署提交查找一句确切的文本，每句都指明该提交的完整 SHA：

```
I, Jane Developer <jane@example.com>, hereby add my Signed-off-by to this commit: 3f53ada...

Signed-off-by: Jane Developer <jane@example.com>
```

失败的检查会打印要复制的确切文本，因此从那里取，而不是手动输入。注意应用本身建议不要为此使用 `--allow-empty`，因为如果有人变基分支，空提交会被丢弃。

配置位于 [`.github/dco.yml`](.github/dco.yml)。

### 已验证签名

上面的签署是提交信息中的一行文本。`main` 还要求每个提交都有加密签名。规则集在合并框中报告为 "Commits must have verified signatures."，并在分支上每个提交都带有签名之前阻止合并。处于该状态的打开的 Pull Request 会获得 `needs-signed-commits` 标签。

使用注册在你 GitHub 账户上的 GPG 或 SSH 密钥签名，邮箱地址需已由该账户验证。GitHub 的[提交签名验证](https://docs.github.com/en/authentication/managing-commit-signature-verification)指南涵盖两种密钥类型。然后默认对此克隆中的每个提交签名：

```bash
git config commit.gpgsign true
```

要为分支上已有的提交签名，带签名重放它们：

```bash
git rebase --keep-base --force-rebase --gpg-sign main
git push --force-with-lease
```

`--force-rebase` 很重要：没有它，变基会保持已位于基线上的提交原样，即未签名。`--gpg-sign` 也适用于 SSH 密钥；该选项比 SSH 签名更早。对于单个提交，`git commit --amend --no-edit -S` 后跟同样的推送即可。

## AI 辅助贡献

我们欢迎在 AI 编码助手（Claude、Copilot、Cursor 等）帮助下编写的贡献。使用 AI 辅助时：

- **你负责**在提交前审查并理解所有 AI 生成的代码。不要提交你无法在审查中解释或辩护的代码。
- **验证正确性：** AI 生成的代码可能包含细微的 bug 或安全问题。运行测试套件并仔细审查 diff。
- **不要用 AI 生成敏感内容：** 不要用 AI 生成安全策略、法律文本或许可证头 —— 这些需要人工判断和法律审查。
- **在 PR 中披露**改动中是否有相当一部分由 AI 生成，以便审查者校准审查深度。说明请自己写。
- **不要标注 agent 归属：** 不要在提交中添加命名 AI 助手的 `Co-Authored-By` 尾注，也不要在 PR 描述中添加 "Generated with …" 之类的工具生成页脚。你是改动的作者；请关闭工具的默认归属。

## 行为准则

所有贡献者都应遵守我们的[行为准则](CODE_OF_CONDUCT.md)。

## 许可证头

所有新的源文件必须包含 SPDX 许可证标识符作为第一行：

```
# SPDX-License-Identifier: Apache-2.0
```

根据你的语言调整注释风格（`//`、`/*`、`--` 等）。这让许可证扫描器无需启发式匹配即可识别许可证。

## 归属

部分内容改编自 https://github.com/NVIDIA-GitHub-Management/PLC-OSS-Template/blob/main/CONTRIBUTING.md