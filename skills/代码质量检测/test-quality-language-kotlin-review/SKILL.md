---
name: "test-quality-language-kotlin-review"
description: "Use when reviewing Kotlin, Android, Ktor, Gradle, coroutines, Flow, suspend APIs, null safety, MockK/JUnit tests, or Kotlin multiplatform code quality."
---
# Kotlin 语言专项质量审查

## 定位

本 skill 用于 Kotlin 代码层质量审查，覆盖 Kotlin/JVM、Android、Ktor 与 Kotlin Multiplatform 场景。审查目标是发现会导致编译失败、运行时崩溃、并发泄漏、资源泄漏、平台不兼容、测试失效或长期维护风险的问题。

## 执行边界

- 只做代码层静态审查；不默认发起 API 请求、不默认运行应用。
- 不执行 Git 命令；如用户要求查看变更范围，先请用户提供文件或明确授权其他方式。
- 不自动修复问题；用户要求修复时再进入代码修改流程。
- 审查结论必须有文件路径、行号、问题等级、原因和建议。
- 中文路径、中文内容和 Markdown 输出必须保持 UTF-8。
- 报告结构优先遵循 `test-quality-report-template`。

## 内置规则覆盖

- Kotlin 基础：null safety、sealed class、data class、extension、scope function、不可变集合和类型推断要服务可读性。
- 协程/Flow：结构化并发、Dispatcher、取消传播、异常处理、重复收集、背压和生命周期绑定必须检查。
- Android/Ktor/KMP：平台 API 泄漏、主线程阻塞、生命周期外收集、Ktor pipeline 错误映射和 KMP source set 依赖要按场景判断。
- Gradle Kotlin DSL：插件版本、source set、compilerOptions、kapt/ksp、detekt/ktlint、测试任务和 CI 覆盖属于质量审查范围。
- 测试：JUnit/MockK、协程测试、Flow 测试、边界和错误路径覆盖都要检查。

## 检查流程

1. 确认审查范围：明确用户给出的文件、模块、目录或问题类型；范围不明时先扫描 Kotlin/Gradle 文件。
2. 按内置规则读取 Kotlin 源码、Gradle 配置和测试，避免依赖外部目录。
3. 建立项目画像：识别 Android、Ktor、Spring、纯 JVM、KMP、Compose、Gradle Kotlin DSL、版本目录等上下文。
4. 搜索风险线索：用 `rg -n` 定位空安全、协程、Flow、资源释放、异常处理、Gradle 和测试关键点。
5. 人工核验上下文：不要只凭关键词报问题；必须查看调用链、生命周期、线程/Dispatcher、异常路径和测试断言。
6. 分级输出：按阻断、高风险、中风险、建议优化分组；无法确认时说明缺失证据和下一步验证方式。
7. 复核报告：确保每条问题都能落到具体文件行号，避免泛泛建议。

## 问题分级

### 阻断

- 代码无法编译、Gradle 配置明显冲突、测试无法启动且影响交付判断。
- 已确认的运行时崩溃、数据错误、安全缺陷、资源泄漏或协程泄漏。
- 硬编码密钥、SQL 拼接注入、不安全反序列化或文件路径越界，可能造成凭证泄露、数据泄露或远程风险。
- Android 主线程阻塞、生命周期外持续收集、Ktor 请求处理路径可导致错误响应或未捕获异常。
- `CancellationException` 被吞掉、结构化并发被破坏，导致取消、超时或失败传播失效。

### 高风险

- 外部输入、平台类型、序列化结果或数据库返回值上使用 `!!`、不安全 cast 或未校验 `lateinit`。
- 生产代码使用 `GlobalScope`、裸 `CoroutineScope()`、`runBlocking`、无边界 `launch/async`。
- `suspend` 函数内部执行阻塞 IO/CPU 工作但未切换 Dispatcher，或暴露 `Deferred` 破坏调用方结构化并发。
- `Flow` 热流作用域、replay/buffer、异常处理、`flowOn`、`shareIn/stateIn` 生命周期配置错误。
- 资源未使用 `use`、未关闭 response/body/channel，`callbackFlow` 未 `awaitClose` 或监听器未注销。
- Ktor 路由缺少输入校验、认证授权边界、`StatusPages` 映射或序列化异常处理。
- Spring/Ktor 外部输入缺少 Bean Validation、DTO 校验、枚举/分页/排序白名单或对象级权限校验。
- MockK/JUnit 测试未覆盖失败分支、协程取消、Flow 错误、无效输入或平台生命周期。

### 中风险

- 可变状态过多，`var`、可变集合、共享 mutable state 或 data class 可变字段导致并发/等值语义风险。
- sealed 层级未通过 `when` 穷尽处理，新增状态时容易漏分支。
- 过宽的 `catch (Exception)`、吞异常、丢失 cause、日志无上下文或业务异常映射不清。
- Android `viewModelScope/lifecycleScope`、Compose effect、`repeatOnLifecycle` 使用不当但未形成已确认崩溃。
- Gradle 插件、Kotlin、AGP、Ktor、coroutines、serialization、jvmTarget 版本不一致或依赖范围不合理。
- 测试使用真实延时、顺序依赖、共享 mock 状态，或协程测试未使用 `runTest`、`coEvery/coVerify`。

### 建议优化

- 命名、包结构、函数长度、嵌套层级、魔法值、重复逻辑、scope function 滥用。
- 可用 data class、sealed interface/class、extension function、`val`、`when`、`require/check` 提升表达力。
- 补充 ktlint/detekt、版本目录、测试命名、边界样例和回归测试。
- 让 public API 的 nullable、suspend、Flow、异常契约更清晰。

## 怎么检查

### Null Safety

- 查 `!!`、`as`、`lateinit`、平台类型、`Map/List` 取值、JSON/数据库/Intent/Bundle 入参。
- 判断 nullable 是否来自外部边界；优先建议 `?.`、`?:`、`requireNotNull`、显式校验或领域异常。
- 关注 `List<T?>`、`List<T>?`、`Result<T?>` 这类嵌套 nullable 的语义是否清晰。

### Coroutines 与 suspend

- 查 `GlobalScope`、`CoroutineScope(`、`launch`、`async`、`runBlocking`、`withContext`、`Dispatchers`。
- 核验是否结构化并发：子任务是否在调用方 scope 内，异常和取消是否能正确传播。
- `suspend` 函数不应隐藏阻塞调用；阻塞 IO 使用 `withContext(Dispatchers.IO)`，CPU 密集工作使用合适 Dispatcher。
- 不要吞掉 `CancellationException`；捕获异常后需要恢复取消或重新抛出。

### Flow

- 查 `flow {}`、`callbackFlow`、`channelFlow`、`collect`、`launchIn`、`stateIn`、`shareIn`、`catch`、`retry`。
- 区分冷流和热流，核验热流 scope 生命周期、replay/buffer 策略、背压和取消路径。
- Android 收集应关注 `repeatOnLifecycle`、`collectAsStateWithLifecycle`、Compose effect key。
- `catch` 只捕获上游异常；不要把取消异常转成普通失败状态。

### Android

- 查 `viewModelScope`、`lifecycleScope`、`Dispatchers.Main`、`LiveData`、`StateFlow`、Compose side effects。
- 主线程不得执行阻塞 IO、数据库重活或长 CPU 任务。
- 生命周期对象、Context、View、Adapter、listener、receiver、callback 必须有清理路径。
- Compose 中关注重组稳定性、remember key、副作用取消、状态提升和 Flow 生命周期收集。

### Ktor

- 查 `routing`、`authenticate`、`receive`、`respond`、`StatusPages`、`ContentNegotiation`、client request。
- 请求 DTO 必须校验；错误响应要稳定映射，不泄露内部异常。
- `suspend` route 内避免阻塞；数据库、HTTP client、文件 IO 要有超时、关闭和异常处理。
- Ktor client response/body、stream、channel 使用后必须释放。

### Spring / 安全输入边界

- Spring Controller、Ktor route、消息消费者和定时任务的外部输入必须有 `@Valid`、Bean Validation、DTO 校验或显式白名单。
- 检查 `JdbcTemplate`、Exposed、JPA、MyBatis、字符串模板 SQL 和动态排序字段是否参数化或白名单。
- 检查 kotlinx serialization、Jackson、反射、文件路径、URL 和命令参数是否限制来源、大小、格式、协议和根目录。
- 检查配置、测试夹具、默认值和前端共享代码中是否硬编码 token、密码、私钥、连接串或内部地址。

### sealed、data class 与 Kotlin 习惯用法

- data class 用于纯数据承载；避免把可变集合、业务副作用或资源句柄放进 data class。
- sealed class/interface 用于有限状态；`when` 应穷尽，跨模块暴露时关注 binary/source 兼容。
- 优先 `val`、表达式式 `when`、扩展函数和清晰的 scope function；避免为了“简洁”牺牲可读性。

### 异常处理与资源释放

- 异常边界要区分参数错误、业务错误、外部依赖错误、取消和系统错误。
- 不要空 catch、只打印堆栈、丢失 cause 或把所有异常映射成同一结果。
- 文件、流、游标、response、channel、subscription、listener 使用 `use`、`close`、`cancel`、`awaitClose` 或生命周期清理。

### Gradle 与多平台兼容

- 查看 `settings.gradle.kts`、`build.gradle.kts`、`gradle/libs.versions.toml`、`sourceSets`。
- 校验 Kotlin、AGP、Ktor、coroutines、serialization、JUnit、MockK、Compose compiler 版本是否匹配。
- KMP 中 `commonMain` 不应直接依赖 JVM/Android API；平台能力通过 `expect/actual`、接口或依赖注入隔离。
- Android、JVM、iOS source set 的依赖范围要准确，避免把测试依赖或平台实现泄漏到 common 层。

### 测试

- 单元测试优先覆盖成功、失败、空值、异常、取消、超时、无效输入和边界集合。
- 协程测试使用 `runTest`、测试 Dispatcher/MainDispatcherRule；避免真实 `Thread.sleep` 或固定延时。
- MockK 协程调用使用 `coEvery/coVerify`；Flow 可用 Turbine 或等价方式断言顺序、完成、错误和取消。
- Android 关注 ViewModel/Compose/lifecycle 测试，Ktor 关注 `testApplication`、路由校验和错误响应。

## 搜索线索

优先使用带行号搜索，命中后读取上下文再判断：

```powershell
rg -n "!!|lateinit|\\bas\\b|GlobalScope|CoroutineScope\\(|runBlocking|launch\\(|async\\(|withContext|Dispatchers\\.|CancellationException" <path>
rg -n "flow\\s*\\{|callbackFlow|channelFlow|collect\\(|launchIn|stateIn|shareIn|flowOn|catch\\s*\\{|retry\\s*\\{" <path>
rg -n "use\\s*\\{|close\\s*\\(|awaitClose|add.*Listener|remove.*Listener|registerReceiver|unregisterReceiver" <path>
rg -n "routing\\s*\\{|authenticate\\s*\\{|receive<|respond\\(|StatusPages|ContentNegotiation|HttpClient" <path>
rg -n "@Valid|@Validated|NotBlank|NotNull|Size\\(|Pattern\\(|JdbcTemplate|createQuery|nativeQuery|exec\\(|ProcessBuilder|decodeFromString|readText\\(|Path\\(" <path>
rg -n "password|secret|token|apiKey|privateKey|connectionString|jdbc:" <path>
rg -n "viewModelScope|lifecycleScope|repeatOnLifecycle|collectAsStateWithLifecycle|LaunchedEffect|DisposableEffect|remember" <path>
rg -n "runTest|coEvery|coVerify|mockk|Turbine|testApplication|Thread\\.sleep|delay\\(" <path>
rg -n "plugins\\s*\\{|kotlin\\(|jvmTarget|sourceSets|commonMain|androidMain|libs\\.versions\\.toml" <path>
```

文件发现优先：

```powershell
rg --files <path> -g "*.kt" -g "*.kts" -g "libs.versions.toml"
```

## 输出格式

```markdown
## Kotlin 质量审查报告

### 阻断
- [阻断] 文件路径:行号 - 问题标题
  原因：说明为什么会失败、崩溃、泄漏或破坏交付。
  建议：给出可执行修复方向。
  验证：说明应补充或运行的测试。

### 高风险
- [高风险] 文件路径:行号 - 问题标题
  原因：
  建议：
  验证：

### 中风险
- [中风险] 文件路径:行号 - 问题标题
  原因：
  建议：
  验证：

### 建议优化
- [建议优化] 文件路径:行号 - 问题标题
  原因：
  建议：
  验证：

### 未确认项
- 文件路径:行号 - 缺少什么证据，以及下一步如何确认。
```

若没有发现问题，明确说明“未发现阻断/高风险/中风险问题”，并列出未运行测试或未覆盖范围的残余风险。
