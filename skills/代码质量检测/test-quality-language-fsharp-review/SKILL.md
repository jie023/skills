---
name: "test-quality-language-fsharp-review"
description: "Use when reviewing F# code quality, functional idioms, type safety, pattern matching, computation expressions, or performance."
---
# F# 语言专项质量审查

## 定位

F# 专项代码质量审查，重点覆盖函数式习惯、类型安全、模式匹配、错误处理、异步、资源释放、安全和性能。

## 适用范围

- 用户要求审查 F#、`.fs`、`.fsx`、Giraffe、Saturn、Fable、Elmish、EF Core F# 代码时使用。
- 涉及领域建模、判别联合、Option/Result、task/async 计算表达式、模式匹配、数据库访问、Web handler 和测试时使用。
- 默认只做静态审查，不执行 `dotnet build`、`dotnet test`、`fantomas` 或外部命令；需要运行时先说明目的并等待确认。

## 检查流程

1. 明确审查范围：优先审查用户指定的 `.fs`、`.fsx`、项目文件、测试文件和相关配置。
2. 识别项目形态：普通 .NET、ASP.NET Core/Giraffe/Saturn、EF Core、Fable/Elmish、脚本或库项目。
3. 读取上下文：从外部输入、handler、service、domain、repository、view/update、测试文件建立调用链。
4. 按分级检查项判断：先安全、错误处理、资源释放和异步阻塞，再查函数式习惯、类型建模和性能。
5. 每个问题必须包含文件路径、行号、证据、风险、建议和验证方式；无法确认时说明需要的上下文。

## 分级检查项

### 阻断

- **注入与路径安全风险**：SQL 字符串拼接、`Process.Start` 使用外部输入、用户可控路径未规范化、视图输出未编码。
  - 怎么检查：搜索 SQL 插值、命令执行、文件路径拼接、HTML/raw 输出，追踪输入是否来自请求、文件或外部系统。
- **不安全反序列化或 CSRF 缺失**：对不可信输入使用 BinaryFormatter、危险 JSON/XML/YAML 反序列化，或 Cookie 会话写接口缺少 CSRF 防护。
  - 怎么检查：搜索反序列化 API、Web handler、表单提交、Cookie 认证配置和 anti-forgery 配置。
- **硬编码密钥和敏感配置**：API key、连接串、token、密码写在源码、脚本或测试夹具里。
  - 怎么检查：搜索 `password`、`secret`、`token`、`connectionString`、`apiKey`、`privateKey`。
- **异常被吞或错误语义丢失**：`with _ -> ()`、`with _ -> None`、库代码用 `failwith` 表达预期失败。
  - 怎么检查：搜索 `try ... with`、`failwith`、`raise`，确认是否返回 `Result`/`Option` 或保留错误上下文。
- **资源未释放**：`IDisposable` 未用 `use`/`use!`，数据库连接、流、HTTP 响应未释放。
  - 怎么检查：搜索 `new`、`Dispose`、`use`、`use!`、文件/连接/DbContext 创建点。
- **阻塞异步导致死锁或线程耗尽**：`.Result`、`.Wait()`、`GetAwaiter().GetResult()` 出现在 async/task 路径。
  - 怎么检查：沿 `task {}`、`async {}`、Web handler 和数据库调用链检查同步阻塞。

### 高风险

- **模式匹配不完整或 catch-all 掩盖新 case**：判别联合新增 case 后 `_` 分支吞掉业务语义。
  - 怎么检查：查 `match` 表达式、`function`、`_ ->`，确认是否显式覆盖重要业务 case。
- **领域建模过度使用基础类型**：用裸 string/int 表示订单号、金额、状态、邮箱等核心概念。
  - 怎么检查：查看 domain record、函数签名、DTO 转换，判断是否应使用单例判别联合或智能构造器。
- **边界输入未校验**：外部输入直接变成 domain 类型或数据库查询条件。
  - 怎么检查：从 handler/API/文件/消息入口追踪到 domain/service，确认是否有 smart constructor 或 Result 校验。
- **过度 mutable/ref**：领域逻辑中大量 `mutable`、`ref`、原地修改集合。
  - 怎么检查：搜索 `mutable`、`ref`、`<-`，确认是否可用不可变数据、fold/map 或返回新值。
- **计算表达式误用**：嵌套 `task { task { } }`、错误使用 `let`/`let!`、异常路径未表达。
  - 怎么检查：读取 `task`、`async`、`result`、`option` 表达式，确认返回类型和错误传播一致。
- **Web/EF/Fable 架构风险**：handler 缺少认证/校验，EF lazy loading 循环导致 N+1，Elmish update 覆盖不完整。
  - 怎么检查：查看 handler pipeline、auth policy、EF Include/AsNoTracking、Fable Msg match 分支。
- **null 未转换为 Option**：C# API、数据库、JSON、环境变量返回 null 后直接进入领域逻辑。
  - 怎么检查：检查 `.NET` 互操作边界是否用 `Option.ofObj`、显式校验或 Result 映射。

### 中风险

- **函数过长或嵌套过深**：函数超过约 40 行，嵌套超过 3 层，pipe 链过长难读。
  - 怎么检查：带行号读取函数边界，确认是否可提取命名函数或中间绑定。
- **OOP 风格过重**：简单业务用 class 层层包装，模块/函数/record/DU 更适合。
  - 怎么检查：查看 class 使用是否只是持有函数和数据，是否引入可变状态和继承复杂度。
- **`obj`、强制向下转型或装箱过多**：`:?>`、`box`、`unbox`、`obj` 破坏类型安全。
  - 怎么检查：搜索 `:?>`、`:?`、`box`、`unbox`、`obj`，确认是否可用泛型或判别联合。
- **Seq 在热点路径重复计算**：lazy sequence 多次枚举、数据库/文件流被重复遍历。
  - 怎么检查：搜索 `Seq.`、多次消费同一序列、循环内 Seq 处理，判断是否应 materialize。
- **命令式循环或字符串拼接性能问题**：可读性更适合 `List.map/fold/choose` 的代码使用复杂循环，或循环中反复 `+` 拼接字符串。
  - 怎么检查：搜索 `for`、`while`、`StringBuilder`、`String.concat`、字符串 `+`，结合数据量判断是否应调整。
- **命名和组织不一致**：函数/值、类型/模块/DU case 命名与 F# 惯例或项目风格冲突。
  - 怎么检查：对照同模块命名，检查 camelCase/PascalCase、模块聚合和 open 顺序。

### 建议优化

- **缺少 `[<RequireQualifiedAccess>]`**：容易与常见 case 名称冲突的 DU 或模块可加限定访问。
- **管道链可读性优化**：长 pipe 可拆为命名中间值，特别是混合校验、转换、副作用时。
- **测试覆盖补充**：为 Result/Option、模式匹配分支、边界输入、异步失败和 EF 查询补测试。
- **格式和 import 清理**：建议使用项目现有 formatter/fantomas 配置，移除未使用 open。

## 常用搜索线索

按当前平台工具执行搜索，结果必须包含文件路径、行号和具体命中内容。

```text
failwith|raise|with _ ->|try|Result|Option|None|Some
\\.Result|\\.Wait\\(|GetAwaiter\\(\\)\\.GetResult
mutable|ref|<-|Array\\.set|ResizeArray
match |function|_ ->|:?|:?>|box|unbox|obj
task \\{|async \\{|let!|do!|return!
Process\\.Start|BinaryFormatter|Deserialize|Antiforgery|ValidateAntiForgeryToken|Path\\.GetFullPath|File\\.|Directory\\.
SELECT|ExecuteSql|FromSql|SqlCommand|string interpolation
null|Option\\.ofObj|isNull|Unchecked\\.defaultof|Seq\\.|List\\.|Array\\.|StringBuilder|String\\.concat
```

## 判断标准

- 阻断：可能造成注入、敏感信息泄露、资源泄漏、异步死锁、错误被吞或关键主流程不可用。
- 高风险：类型系统未表达关键业务约束，模式匹配/异步/架构缺陷可能引入严重业务错误。
- 中风险：当前可运行，但函数式习惯、性能、组织或类型安全存在明显维护风险。
- 建议优化：格式、命名、测试、模块组织和 F# 惯用法可进一步提升。

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
- 不要求固定存在 `.claude`、`.codex`、`.trae` 等平台专属目录；按当前平台可用工具执行。

## 输出要求

按阻断、高风险、中风险、建议优化分组。每个问题使用以下格式：

```text
- [等级] 问题标题
  - 位置：path/to/File.fs:line
  - 证据：命中的关键代码或调用链
  - 原因：为什么违反 F# 质量要求
  - 建议：如何修改或下一步验证
```

没有发现对应等级问题时写“未发现”。无法确认的问题要说明原因和下一步验证建议。

## 内置规则覆盖

- F# 规范：函数式组合、不可变数据、判别联合、Option/Result、模式匹配完整性和管道可读性都纳入审查。
- 错误处理：避免异常控制流，优先显式 Result/Option；外部边界要保留错误上下文。
- 异步与资源：Async/Task 互操作、取消、异常传播、IDisposable/use 释放和并发共享状态要重点检查。
- .NET 互操作：null、可空引用、C# API、序列化、反射和集合转换要检查边界处理。
- 函数式风格：优先 Option/Result、map/fold/choose、不可变集合和 `String.concat`/`StringBuilder`，避免命令式循环和循环字符串拼接扩大维护或性能风险。
- 测试：纯函数、属性测试、边界输入、错误分支和异步流程需要可验证覆盖。

## 多平台兼容

- Codex：优先使用 `rg`、`rg --files`、`Get-Content -Encoding UTF8`，复杂任务使用 `update_plan`。
- Claude：优先使用 `Read`、`Grep`、`Glob`，复杂任务使用可用的 Todo 工具。
- Trae：优先使用 `SearchCodebase`、`Read`，复杂任务使用可用的任务清单工具。
- 任一平台都必须遵守：不默认执行 Git、不自动修复、不批量改文件、中文内容保持 UTF-8。
