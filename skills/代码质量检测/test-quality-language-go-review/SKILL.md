---
name: "test-quality-language-go-review"
description: "Use when reviewing Go code quality, error handling, context cancellation, goroutine/channel safety, Gin/GORM/SQL risks, race conditions, tests, go vet, or idiomatic Go patterns."
---
# Go 语言专项质量审查

## 定位

用于 Go 项目的代码质量审查，重点覆盖错误处理、context 传播与取消、goroutine/channel 生命周期、资源释放、nil pointer、interface 设计、SQL 注入、并发竞态、测试覆盖和 `go vet`。

本 skill 只做代码层审查和本地质量检查，不默认执行 Git 操作，不跑 API 请求，不自动修复代码。审查必须输出明确的文件路径、行号、等级、原因、检查方式和修复建议。

## 内置规则覆盖

- Go 规范：错误必须显式处理，命名简洁，包职责清晰，interface 尽量由使用方定义，避免过度抽象。
- 并发安全：goroutine 生命周期、context 取消、channel 关闭、WaitGroup、errgroup、map/slice 共享写入和 race 风险必须检查。
- 资源释放：HTTP body、文件、数据库 rows、timer/ticker、锁和事务都要有关闭、释放或回滚路径。
- Web/数据库：Gin/GORM/SQL 的参数绑定、权限校验、超时、事务、N+1 和 SQL 注入要结合调用链判断。
- 测试与工具：`go test`、`go vet`、race、table-driven tests、mock/fake、边界和错误路径覆盖是审查依据。

## 执行流程

1. 确认审查范围：优先使用用户指定文件；未指定时搜索 `go.mod`、`go.work`、`*.go`、`*_test.go`，排除 `vendor/`、生成文件和构建产物。
2. 按内置规则读取当前项目代码、配置和测试，再结合调用链判断。
3. 搜索风险线索：使用带行号的搜索结果定位候选代码，再读取上下文确认，不能只凭关键字下结论。
4. 分级判定：按阻断、高风险、中风险、建议优化归类；安全、数据损坏、死锁、泄漏、panic、竞态、编译/测试失败优先。
5. 可运行检查：在用户允许、环境可用且不会访问生产数据时，可运行 `go test ./...`、`go vet ./...`；并发改动或共享状态风险高时建议或运行 `go test -race ./...`。无法运行时说明原因和替代复核方式。
6. 输出报告：遵循 `test-quality-report-template` 的问题优先风格，先列问题，再列验证结果和残余风险。

## 阻断检查项

| 检查项 | 怎么检查 | 常用线索 | 判定标准 |
| --- | --- | --- | --- |
| SQL 注入 | 查 SQL 拼接、动态表名、动态排序、GORM Raw/Exec/Where/Order/Table 入参，确认是否使用占位符和白名单 | `fmt.Sprintf(`、`+ "SELECT`、`Raw(`、`Exec(`、`Where(`、`Order(`、`Table(` | 用户输入进入 SQL 结构、字段、排序或条件且无占位符/白名单 |
| 硬编码凭证或敏感配置 | 查 token、密码、连接串、私钥、AK/SK、TLS 配置和默认环境变量 | `password`、`secret`、`token`、`apiKey`、`InsecureSkipVerify` | 真实凭证写入代码，或生产 TLS 校验被关闭 |
| 命令注入或路径穿越 | 查 `exec.Command`、`os/exec`、文件上传下载、路径拼接和归一化边界 | `exec.Command`、`os/exec`、`filepath.Join`、`Clean(`、`Open(` | 外部输入进入命令参数、shell 或文件路径且无白名单/根目录校验 |
| goroutine/channel 泄漏或死锁 | 查 `go func`、channel 创建、`select`、循环、ticker/timer、WaitGroup/errgroup，确认退出路径、取消信号、send/receive 配对 | `go func`、`make(chan`、`select {`、`ctx.Done()`、`time.Tick`、`WaitGroup`、`errgroup` | goroutine 无退出条件、阻塞发送/接收不可解除、ticker 未停止、WaitGroup 计数不平衡 |
| 并发竞态 | 查 goroutine 内共享变量、map/slice 写入、全局状态、闭包捕获循环变量、锁和 atomic 使用 | `go func`、`map[`、`append(`、`sync.Mutex`、`atomic.`、`var ` | 多 goroutine 读写共享状态且无锁、channel 串行化或 atomic 保证 |
| nil pointer panic | 查指针返回值、类型断言、接口 nil、map/slice/channel 初始化、错误分支后的继续执行 | `return nil, nil`、`.(*`、`.(type)`、`interface{}`、`any`、`nil` | 可达路径上未判空直接解引用，或 typed nil interface 导致误判 |
| 编译、vet 或测试失败 | 运行或要求运行 `go test ./...`、`go vet ./...`，查看失败包和具体行 | `go test ./...`、`go vet ./...` | 当前变更导致编译失败、vet 明确报错、核心测试失败 |

阻断问题通常要求修复后再合入或发布。

## 高风险检查项

| 检查项 | 怎么检查 | 常用线索 | 判定标准 |
| --- | --- | --- | --- |
| error 被忽略或吞掉 | 查 `_ =`、多返回值只取数据、`err` 被覆盖、日志后继续成功返回；确认调用方是否能感知失败 | `_ =`、`, _ :=`、`if err != nil`、`return nil`、`panic(` | I/O、DB、事务、序列化、锁、Close/Flush/Commit 等关键错误未处理 |
| 错误语义丢失 | 查 `fmt.Errorf`、`errors.Is/As`、sentinel error、自定义错误；确认是否用 `%w` 包装底层错误 | `fmt.Errorf(`、`errors.Is`、`errors.As`、`errors.New` | 跨层返回错误但丢失根因，导致调用方无法分类处理或定位问题 |
| context 未传播或未取消 | 查 handler/service/repository/HTTP/DB 调用链是否带 `context.Context`，WithCancel/WithTimeout 是否 `defer cancel()` | `context.Background()`、`context.TODO()`、`WithTimeout`、`QueryContext`、`ExecContext`、`NewRequestWithContext`、`WithContext` | 请求链路、DB/HTTP、goroutine 或长耗时任务未响应取消/超时 |
| defer/resource close 不可靠 | 查文件、HTTP body、rows、tx、ticker、timer、writer/flush 的关闭和错误处理 | `defer .*Close()`、`Rows()`、`Query(`、`Begin(`、`Commit()`、`Rollback()`、`Flush()` | 资源可能泄漏；写入、flush、commit、close 错误被忽略导致数据丢失 |
| Gin/GORM 输入校验缺失 | 查 handler bind、validator tag、DTO 边界、分页/排序参数、GORM 条件 | `ShouldBind`、`BindJSON`、`validate:`、`gorm`、`Where(` | 外部输入未校验即进入业务、SQL、文件路径或命令参数 |
| TLS 或 HTTP 客户端不安全 | 查 `http.Client`、Transport、TLS 配置、超时和重定向策略 | `http.Client`、`Transport`、`TLSClientConfig`、`InsecureSkipVerify`、`Timeout` | 关闭证书校验、无超时或重定向/代理策略未限制 |

高风险问题会造成线上故障、安全风险、数据不一致或难以排查的错误。

## 中风险检查项

| 检查项 | 怎么检查 | 常用线索 | 判定标准 |
| --- | --- | --- | --- |
| interface 设计不当 | 查接口定义位置、方法数量、实现侧提前抽象、`interface{}`/`any` 滥用、类型断言 | `type .* interface`、`interface{}`、`any`、`.( ` | 接口过大、由实现方定义、调用方仍需类型断言，增加耦合和 panic 风险 |
| 测试覆盖不足 | 查 `*_test.go`、表驱动测试、错误路径、边界值、并发路径、mock 断言 | `_test.go`、`t.Run`、`assert.`、`require.`、`wantErr` | 只有 happy path，缺少失败分支、非法输入、nil、超时、并发和事务回滚测试 |
| context 使用不规范 | 查 context 是否放入 struct、是否作为第一个参数、是否滥用 `context.Background()` 切断调用链 | `ctx context.Context`、`context.Background()`、`context.TODO()` | 可维护性和取消语义受损，但暂未形成明确泄漏或故障 |
| 锁和 channel 复杂度过高 | 查嵌套锁、锁内 I/O、channel 关闭责任、多生产者关闭、select default 忙等 | `Lock()`、`Unlock()`、`close(`、`default:` | 代码可运行但维护风险高，后续改动易引入死锁或竞态 |
| Go 习惯用法偏离 | 查包命名、导出符号注释、函数长度、深层嵌套、魔法值、重复代码 | `package `、导出函数、超过 50 行函数 | 影响可读性、扩展性或静态工具提示，但未直接造成故障 |

中风险问题需要在当前迭代或近期重构中处理，并补充测试。

## 建议优化检查项

| 检查项 | 怎么检查 | 常用线索 | 判定标准 |
| --- | --- | --- | --- |
| gofmt/goimports | 检查格式、import 分组和未使用 import；必要时建议运行格式化工具 | `gofmt`、`goimports` | 格式不统一但不影响行为 |
| 表驱动测试改进 | 查重复测试用例是否可合并为表驱动，是否覆盖边界名称 | `tests := []struct`、`t.Run` | 测试可读性和扩展性可提升 |
| 错误信息可观测性 | 查日志和错误上下文是否包含关键业务标识但不泄露敏感信息 | `log.`、`slog.`、`fmt.Errorf` | 排障信息不足或错误文案不统一 |
| 性能小优化 | 查切片预分配、字符串拼接、无意义 goroutine、重复正则编译 | `make([]`、`append(`、`strings.Builder`、`regexp.MustCompile` | 对性能有帮助但不属于当前主要风险 |
| sync.Pool 使用边界 | 高频临时对象分配可考虑 `sync.Pool`，但要检查对象重置、生命周期和并发安全 | `sync.Pool`、`Get()`、`Put()` | 只在明确高频分配和可安全复用对象时建议，避免过早优化 |

建议优化不得压过阻断和高风险问题。

## 常用搜索线索

Codex 优先用 `rg` 和 `rg --files`；Claude 优先 `Grep`、`Glob`、`Read`；Trae 优先 `SearchCodebase`、`Read`。所有搜索结果都要带文件路径、行号和匹配内容。

| 目标 | Codex 示例 | 无 `rg` 平台替代 |
| --- | --- | --- |
| Go 文件清单 | `rg --files -g "*.go" -g "!vendor/**"` | 使用平台文件搜索筛选 `.go` |
| error 忽略 | `rg -n "(_\\s*=|,\\s*_\\s*:=|panic\\(|fmt\\.Errorf\\()" -g "*.go"` | 搜索 `_ =`、`, _ :=`、`panic(`、`fmt.Errorf(` |
| context | `rg -n "(context\\.Background\\(|context\\.TODO\\(|WithTimeout|WithCancel|QueryContext|ExecContext|NewRequestWithContext|WithContext)" -g "*.go"` | 搜索 context 创建、传递和 DB/HTTP context API |
| goroutine/channel | `rg -n "(go func|make\\(chan|select \\{|ctx\\.Done\\(|time\\.Tick|WaitGroup|errgroup)" -g "*.go"` | 搜索 goroutine、channel、select、ticker、WaitGroup |
| 资源释放 | `rg -n "(defer .*Close\\(|Close\\(\\)|Rows\\(|Query\\(|Begin\\(|Commit\\(|Rollback\\(|Flush\\()" -g "*.go"` | 搜索 Close、Rows、tx、Flush、defer |
| nil/interface | `rg -n "(interface\\{\\}|\\bany\\b|\\.\\(|return nil, nil|== nil|!= nil)" -g "*.go"` | 搜索 interface、any、类型断言、nil 判断 |
| SQL 风险 | `rg -n "(fmt\\.Sprintf\\(|Raw\\(|Exec\\(|Query\\(|Where\\(|Order\\(|Table\\(|\\+.*SELECT|SELECT.*\\+)" -g "*.go"` | 搜索 SQL API、字符串拼接、动态排序 |
| 安全风险 | `rg -n "(password|secret|token|apiKey|InsecureSkipVerify|exec\\.Command|os/exec|filepath\\.Join|filepath\\.Clean|http\\.Client|TLSClientConfig)" -g "*.go"` | 搜索凭证、TLS、命令执行、路径和 HTTP 客户端配置 |
| 并发竞态 | `rg -n "(go func|sync\\.|atomic\\.|map\\[|append\\(|var .*=)" -g "*.go"` | 搜索共享变量、map、append、锁、atomic |
| 测试 | `rg --files -g "*_test.go"` 与 `rg -n "(t\\.Run|wantErr|assert\\.|require\\.|t\\.Parallel)" -g "*_test.go"` | 搜索测试文件和表驱动关键字 |
| vet/race | `go vet ./...`、`go test -race ./...` | 无法运行时在报告中说明原因 |

## 输出格式

报告必须按等级分组，问题优先，证据具体。没有发现某等级问题时写“未发现明确问题”。

```markdown
## 阻断
- [阻断] 标题
  - 位置：path/to/file.go:123
  - 问题：具体说明可达风险。
  - 依据：说明代码证据、Go 规则或内置规则依据。
  - 怎么检查：说明搜索线索、上下文确认或命令结果。
  - 建议：给出可执行修复方向和需要补充的测试。

## 高风险
...

## 中风险
...

## 建议优化
...

## 验证
- 已执行：`go test ./...` / `go vet ./...` / `go test -race ./...`
- 未执行：说明原因、影响和建议下一步。
```

如果问题依赖运行时条件、外部服务或未提供上下文，必须标注“无法确认”，并说明还需要的代码、配置、测试或日志。

## 多平台兼容

- Codex：搜索优先 `rg`、`rg --files`；读取中文文件显式 UTF-8；修改前后复核关键片段；不执行任何 Git 操作。
- Claude：搜索优先 `Grep`、`Glob`，读取用 `Read`；输出同样保留文件路径、行号和匹配片段。
- Trae：搜索优先 `SearchCodebase`，读取用 `Read`；如工具不支持复杂正则，拆成多个简单关键词搜索。
- Windows 路径使用正斜杠；中文路径和中文内容必须保持 UTF-8。
- 如果用户只要求审查，不直接改代码；如需要修改代码，先征得用户确认并按安全批量修改规则逐项处理。
