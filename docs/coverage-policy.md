# 测试覆盖率

本仓库收集什么、用它做什么，以及——最关键的部分——这个数字**不**代表什么。

## 四条说清策略

每个有测试的模块都会写出覆盖率 profile。CI 校验这些 profile，发布按模块划分的表格，并将 profile 附加到运行记录。**没有任何阈值会拦截合并，也没有计划设置。** 真正会拦截合并的是：缺失、格式错误或为空的 profile，以及没有任何东西在运行的测试包。

## 百分比不等于测试充分性

覆盖率衡量的是测试二进制执行了哪些行。仅此而已。它不知道这些行上是否做了哪怕一个断言。

```go
func TestParse(t *testing.T) {
    _, _ = Parse(input) // Parse 现在 100% "被覆盖"
}
```

这个测试执行了 `Parse` 的每一行，即使 `Parse` 对任何输入都返回垃圾也能通过。一组这样的测试与一组检查每个结果、每条错误路径和每个边界的测试报告相同的数字。两者并不一样好，而百分比无法区分它们。

反过来同样成立。一个表驱动测试从数千个输入空间中挑出四个代表性输入，报告的只是它运行过的那些行的覆盖率，对它没试过的输入只字不提。覆盖率没有"你没想到的用例"这个概念——而 bug 恰恰在那里。

所以：

- **不要用这个数字论证某个改动已被测试。** 说出测试断言了什么。
- **不要用这个数字论证某个改动未被测试。** 指出没有任何东西检查的行为。
- **不要追逐这个数字。** 为提高百分比而写的测试，就是为执行行而写的测试，而这恰恰是已被证明无关紧要的那一个属性。

## 它擅长什么

两件事，都关乎变化而非水平。

**变动。** 一个模块在单个 Pull Request 中掉了十个点，值得一问。答案有时是"删掉了一个大型且测试充分的包"，有时是"测试停止运行了"。两者都值得知道；没有收集，两者都不可见。

**静默未运行的测试。** 这正是收集存在的理由。一个构建标签、一个改名的目录、一个从 `go test` 参数列表中掉落的包——任何一个都会把一套测试带出构建，并带走它的失败。运行仍然通过。没有任何红色。当持有 `_test.go` 文件的包没有产出任何 profile 时，`hack/coverage.sh` 会失败，因此停止运行的测试套件是错误而非缺席。

这里已经发生过。在 controller-test.yml 存在之前，`agent` 是唯一被任何东西跑过 `go test` 的模块，而另外五个模块被编译、lint、vet 和扫描——却从未被测试。`controller/test/kubeadm` 下的两个 `cloudinit` 测试指向一个不存在的路径，每次调用都失败，而 CI 保持绿色，因为控制它们的构建标签在任何地方都没有被设置。

## 如何接线

| 组件 | 负责什么 |
|---|---|
| [`make/coverage.mk`](../make/coverage.mk) | profile 路径、目录和 `-covermode`。被每个收集模块的 Makefile 包含。 |
| 每个模块的 `test` 目标 | 向自己的 `go test` 追加 `$(GO_COVER_FLAGS)`。包参数仍由模块自己决定。 |
| [`hack/coverage.sh`](../hack/coverage.sh) | 哪些模块收集的注册表、各项检查和报告。 |
| [`.github/workflows/coverage.yml`](../.github/workflows/coverage.yml) | 在 CI 中运行它，把表格写入 job summary，上传 profile。 |

Profile 落在仓库根目录的 `coverage/` 下，每个模块一个（`coverage/controller.out`、`coverage/agent.out`、……），已 gitignore。名称是模块路径展平分隔符后的结果。`management-engine` 写两个，因为它的 API 套件和 envtest 套件是分开的调用，无法共享一个输出文件。

### 运行它

```bash
make coverage          # 运行每个收集模块的测试，然后报告
make coverage-report   # 对 coverage/ 中已有的 profile 报告
make coverage-check    # 对照目录树检查注册表；不运行测试
```

`make coverage-check` 是便宜的那个，也是合并门禁中的那个。它不需要运行测试，大约耗时一秒。

读取 profile：

```bash
go tool cover -html=coverage/controller.out        # 在浏览器中
go tool cover -func=coverage/controller.out        # 按函数，输出到 stdout
```

### 哪些模块收集

十三个工作区模块中的十二个。`controller/api` 持有生成的 OpenAPI 类型和生成的客户端，`controller/hack/capd-reboot-controller` 是手动运行的开发辅助工具；两者都没有测试文件。两者都在 `hack/coverage.sh` 中如此声明，如果其中任何一个新增了测试却未被移动，`make coverage-check` 会失败——一个开始测试却不开始收集的模块，正是这里要捕捉的漂移。

## 什么会失败，以及意味着什么

| 失败 | 发生了什么 |
|---|---|
| `no coverage profile in coverage` | 模块的测试没有运行，或运行时没有 `$(GO_COVER_FLAGS)`。 |
| `not a \`mode:\` header` / `malformed block line(s)` | profile 被截断或不是覆盖率 profile。`go tool cover` 会拒绝它，而只检查文件是否存在的检查不会。 |
| `the profile holds 0 statements` | 套件编译了，但没有测量到任何东西。 |
| `holds tests that the default build does not compile` | 一个构建标签排除了某个套件。在 `untested_packages()` 中声明它并说明原因，或去掉标签。 |
| `holds tests and compiles, but reaches no coverage profile` | 模块的 test 目标从未到达那个包。 |
| `waives <dir>, which no longer exists` | 一个过期的豁免，在原因早已消失后仍长期排除某个套件。 |

一个只含 `_test.go` 文件的包——`controller/test/e2e` 本身、`os-image/files/test`、`controller/hack/capd-kindest-node`——豁免于 profile 检查。它没有自己的语句可供覆盖率归属，因此无论运行得多彻底都无法出现在 profile 中。它仍会被编译，出问题时仍会让套件失败。

## 刻意的不选择

**没有阈值，现在或以后。** 阈值把数字变成需要满足的东西，而满足它最便宜的方式就是上面那个无断言的测试。如果将来采用，它需要一个具名负责人和一份书面声明说明它是何事的证据，而这是一份比看起来更难写的文档——本文就是那次尝试，结论是阈值不值得拥有。

**没有外部覆盖率服务。** Codecov 或 Coveralls 会为第三方应用提供趋势图和 Pull Request 评论，代价是仓库访问权限和一个 token。对于本仓库的规模，job summary 和上传的构建产物回答了同样的问题，而之前的 cobertura 转换在消费它的流水线消失时被移除，而非被替换。如果将来想要趋势视图，那才是重新考虑的时刻——不是现在。

**没有 `-coverpkg=./...`。** 把一个包的测试归属到另一个包的语句上，会让集成测试看起来像是它仅仅路过的代码的单元覆盖率，这是对一个本就容易被误读的数字最误导的读法。

## 另见

- [development.md](development.md) — 打开 Pull Request 前要运行的检查
- [go-workspace.md](go-workspace.md) — 收集所覆盖的模块
- [../.github/workflows/README.md](../.github/workflows/README.md) — CI 如何接线