---
name: "test-quality-build-go-review"
description: "Use when diagnosing Go build, go vet, module, dependency, import, or compiler failures."
---
# Go 构建质量审查

## 定位

Go 构建、模块依赖、工具链、`go vet`、lint 和测试入口的质量审查。只做诊断、归类和修复建议，不默认运行会改写文件的命令。

## 检查重点

- 模块结构：`go.mod`、`go.sum`、`go.work`、多模块仓库、`replace`、`exclude`、vendor 模式是否清晰。
- Go 版本：`go` 指令、`toolchain` 指令、CI/容器 Go 版本是否一致。
- 构建入口：`go build ./...`、`go test ./...`、Makefile、Taskfile、Dockerfile、CI 脚本是否一致。
- 依赖与校验：缺失模块、checksum 错误、私有模块、代理设置、`GONOSUMDB` / `GOPRIVATE` 相关风险。
- import 与包名：循环依赖、未使用导入、包名不一致、internal 包越界、生成代码缺失。
- 构建约束：`//go:build`、平台文件后缀、CGO、架构/系统差异、embed 文件是否匹配。
- `go vet` / lint：格式化字符串、copylocks、unreachable、shadow、staticcheck/golangci-lint 规则触发点。
- 测试构建：测试包命名、race、短测试/集成测试区分、外部服务依赖和缓存目录。
- 生成代码：protobuf、wire、sqlc、mock、stringer、swagger/openapi 生成结果是否参与构建。

## 怎么检查

1. 先读取 `go.mod`、`go.sum`、`go.work`、Makefile/Taskfile、Dockerfile、CI 配置，确认项目推荐构建入口。
2. 如用户没有明确要求运行命令，只做静态分析；`go mod tidy`、`go get`、`go clean -modcache` 会改状态，必须先确认。
3. 对错误日志先定位第一处根因，区分依赖问题、语法/类型问题、生成代码缺失、平台条件或测试专属问题。
4. 检查 `replace` 时要说明是否为本地路径、临时调试依赖或私有模块映射；不要直接建议删除。
5. 检查 CGO 和平台构建时，确认 `GOOS`、`GOARCH`、`CGO_ENABLED`、C 编译器和系统库依赖。
6. 对 `go vet` / lint 问题，优先判断是否真实 bug；不要建议随意加 `//nolint`。
7. 对测试构建问题，区分生产包构建失败和测试文件构建失败，分别给出最小建议。

## 常用搜索线索

- 模块入口：`go.mod`、`go.sum`、`go.work`、`vendor/modules.txt`。
- 构建脚本：`go build`、`go test`、`go vet`、`golangci-lint`、`staticcheck`、`race`、`cover`。
- 模块配置：`replace `、`exclude `、`require (`、`GOPRIVATE`、`GONOSUMDB`、`GOPROXY`。
- 平台约束：`//go:build`、`// +build`、`_windows.go`、`_linux.go`、`_amd64.go`、`import "C"`。
- 生成代码：`//go:generate`、`.pb.go`、`wire_gen.go`、`sqlc.yaml`、`mockgen`、`stringer`。
- 常见错误：`cannot find module`、`missing go.sum entry`、`import cycle not allowed`、`undefined:`、`build constraints exclude all Go files`。

## 执行规则

- 只做代码层质量审查，不跑 API 请求。
- 不默认执行任何 Git 操作。
- 不自动修复问题，只输出问题和修复建议。
- 不照搬强制调用、自动修复、hooks、后台观察逻辑。
- 输出必须包含文件路径、行号、问题等级、原因、建议。
- 报告输出遵循 `test-quality-report-template`。
- 如需要修改代码，必须等待用户确认后逐项处理。
- 多文件修改必须按安全批量修改规则执行。
- 中文路径和中文内容必须保持 UTF-8。

## 平台兼容

- Codex：优先使用 `rg` / `rg --files` 搜索，`Get-Content -Encoding UTF8` 读取中文路径文件。
- Claude：优先使用 `Glob` / `Grep` / `Read`，复杂任务用可用 Todo 工具跟踪。
- Trae：优先使用 `SearchCodebase` / `Read`，避免未确认范围的批量写入。
- 任一平台输出都必须包含文件路径、行号和最小必要配置片段。

## 输出要求

按阻断、高风险、中风险、建议优化分组。每条问题包含：

- 文件路径和行号
- 问题等级
- 涉及命令或构建范围
- 根因判断
- 风险原因
- 最小修复建议
- 无法确认时的下一步验证建议

## 内置规则覆盖

- 诊断顺序：先确认 `go.mod/go.sum/go.work`，再看 `go build`、`go vet`、`go test`、lint 和 CI 脚本的入口差异。
- 常见错误：`cannot find package/module`、`missing go.sum entry`、`import cycle not allowed`、`undefined`、`build constraints exclude all Go files`。
- 模块问题：`replace`、私有模块、GOPRIVATE/GONOSUMDB、vendor、workspace 和代理配置都可能改变依赖解析。
- 平台问题：`GOOS`、`GOARCH`、`CGO_ENABLED`、`import "C"`、平台后缀文件和 `//go:build` 必须成组判断。
- vet/lint：格式化字符串、copylocks、unreachable、shadow、staticcheck/golangci-lint 问题要先判断是否真实缺陷。
- 停止条件：`go get`、`go mod tidy`、`go clean -modcache`、下载依赖或改代理配置会改变状态，执行前必须确认。
