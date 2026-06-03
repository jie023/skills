---
name: "test-quality-pattern-android-clean-architecture"
description: "Use when reviewing Android clean architecture, Kotlin Android layers, domain/use cases, repositories, ViewModels, or dependency boundaries."
---
# Android 整洁架构模式质量审查

## 定位

用于审查 Android 或 Kotlin Multiplatform 项目中的 Clean Architecture 落地质量，重点判断分层边界、依赖方向、业务规则位置、异步数据流、错误传播和测试可替换性是否稳定。

## 检查重点

- 分层架构：`app`、`presentation`、`domain`、`data`、`core`、`feature` 等模块职责是否清晰，是否存在跨层直连。
- 依赖方向：依赖是否只从外层指向内层，`domain` 是否保持纯 Kotlin，是否出现循环依赖或反向依赖。
- Domain 纯净度：Domain model、UseCase、Repository interface 是否不依赖 Android、Room、Retrofit、Compose、Hilt/Koin 等框架细节。
- Repository：接口是否定义在 Domain，实现在 Data；是否泄漏 DTO、Entity、Response、Cursor 等数据层类型。
- UseCase：是否表达单一业务动作，是否承载业务规则，是否避免退化为无意义透传包装。
- ViewModel：是否只编排 UI 状态和用户意图，是否把业务规则、数据源选择、复杂转换下沉到 UseCase 或 Mapper。
- 协程/Flow：是否使用结构化并发，是否避免 `GlobalScope`、阻塞调用、错误吞没、重复收集和生命周期不匹配。
- 错误处理：是否使用统一错误模型或 `Result`/sealed type，是否避免把异常、HTTP 细节或数据库异常直接暴露到 UI。
- 测试可替换性：UseCase、Repository、DataSource、Dispatcher、Clock 等依赖是否可替换，是否能用 fake/mock 做单元测试。

## 怎么检查

### 1. 确认项目结构和模块依赖

- 先枚举 `settings.gradle(.kts)`、根 `build.gradle(.kts)`、各模块 `build.gradle(.kts)`，识别模块边界和依赖声明。
- 检查 `domain` 模块是否依赖 `androidx.*`、`com.android.*`、Room、Retrofit、Compose、Hilt/Koin 注解处理等外层或框架依赖。
- 检查 `presentation` 是否直接依赖 `data`，`data` 是否依赖 `presentation`，模块之间是否存在双向依赖。
- 发现依赖方向问题时，定位到具体 `build.gradle(.kts)` 行号，并说明违反的方向规则。

### 2. 检查包导入和类型泄漏

- 搜索 `domain` 目录下的 `import android.`、`import androidx.`、`import retrofit2.`、`import kotlinx.serialization.`、`@Entity`、`@Dao`、`@Composable`、`LiveData` 等框架或 UI/数据层符号。
- 搜索 `presentation` 或 UI 层是否出现 `Dto`、`Entity`、`Dao`、`ApiService`、`Retrofit`、`RoomDatabase` 等数据层类型。
- 搜索 Repository interface 的返回值和参数，确认对外暴露的是 Domain model、统一错误模型或 Flow/Result，而不是 DTO、Entity、HTTP Response。
- 问题报告必须引用具体文件和行号，不只描述“存在泄漏”。

### 3. 检查 UseCase 质量

- 按 `*UseCase.kt`、`*Interactor.kt` 或项目约定查找业务用例。
- 判断每个 UseCase 是否聚焦一个业务动作，命名是否表达业务意图，如 `GetUserProfileUseCase`、`SubmitOrderUseCase`。
- 检查 UseCase 内是否包含必要的校验、组合、事务边界、错误转换或业务规则；如果只是单行调用 Repository 且无业务语义，标为建议优化或中风险，视项目复杂度判断。
- 检查 UseCase 是否直接访问 DataSource、Dao、ApiService、Android Context、Resource、NavController、Composable 等外层细节。

### 4. 检查 Repository 与 DataSource

- Repository interface 应位于 Domain 或等价内层，Repository implementation 应位于 Data。
- Repository implementation 可以协调 remote/local/cache，但不应把 UI 状态、导航、文案资源或 ViewModel 状态混入。
- DataSource 负责具体数据来源，Mapper 负责 DTO/Entity/Domain 转换；避免在 ViewModel 中做跨层模型转换。
- 检查缓存策略、同步策略和错误转换是否集中且可测试，避免每个调用点重复拼装。

### 5. 检查 ViewModel 和 UI 状态

- ViewModel 应依赖 UseCase 或 Domain 层抽象，不应直接依赖 Retrofit service、Dao、Room database、DataSource 实现。
- UI 状态建议使用不可变 state holder，如 data class、sealed interface，并通过 `StateFlow`、`SharedFlow` 或项目既有状态机制暴露。
- 检查 `viewModelScope.launch` 内是否直接写大量业务逻辑、复杂数据映射或多数据源协调；这些逻辑应下沉到 UseCase/Repository/Mapper。
- 检查 Flow 收集是否和生命周期匹配，Compose 侧是否使用项目约定的生命周期感知收集方式。

### 6. 检查协程、Flow 和线程

- 搜索 `GlobalScope`、`runBlocking`、`Thread.sleep`、`withContext(Dispatchers.IO)`、`Dispatchers.Main`、`flowOn`、`catch`、`stateIn`、`shareIn`。
- `GlobalScope`、UI 主线程阻塞、Repository 内吞异常后返回假成功通常是高风险。
- Dispatcher 应通过注入或统一 provider 管理，便于测试替换；避免在业务代码里散落硬编码 dispatcher。
- Flow 应明确错误传播和取消语义，避免 `catch {}` 空处理、重复 `collect` 导致多次请求、`stateIn` scope 生命周期过长。

### 7. 检查错误处理和结果建模

- 确认跨层错误是否转换为领域可理解的错误模型，如 `AppError`、`DomainError`、sealed result。
- Data 层可捕获网络、数据库、序列化异常，但向 Domain/Presentation 暴露时应避免泄漏 Retrofit、Room、SQLDelight 等实现细节。
- ViewModel 应把领域错误映射成 UI state，不应在 UI 层散落异常类型判断或 HTTP code 分支。

### 8. 检查测试可替换性

- 检查构造函数注入是否覆盖 Repository、UseCase、DataSource、Dispatcher、Clock、IdGenerator 等外部依赖。
- 确认 UseCase 和 ViewModel 是否能在 JVM 单元测试中运行，不依赖 Android framework 或真实网络/数据库。
- 检查测试中是否需要启动完整 DI 容器、真实数据库或真实网络才能验证业务规则；如需要，说明可替换性不足。
- 优先建议 fake repository、fake data source、test dispatcher、Turbine 或项目既有 Flow 测试工具。

## 平台工具映射

- Codex：优先用 `rg --files` 枚举文件，用 `rg -n "关键词"` 搜索内容，用 `Get-Content -LiteralPath '路径' -Encoding UTF8` 读取中文或 Windows 路径文件。
- Claude：优先用 `Glob` 枚举文件，用 `Grep` 搜索内容，用 `Read` 读取目标文件。
- Trae：优先用 `SearchCodebase` 搜索代码和结构，用 `Read` 读取目标文件。
- 所有平台都必须在搜索结果和报告中保留文件路径、具体代码内容或命中上下文、行号。

## 建议搜索清单

- 模块依赖：`settings.gradle`、`include(`、`implementation(project(`、`api(project(`、`ksp(`、`kapt(`。
- Domain 纯净度：`import android.`、`import androidx.`、`@Entity`、`@Dao`、`@Composable`、`Context`、`Resources`、`LiveData`。
- 跨层泄漏：`Dto`、`Entity`、`Response<`、`ApiService`、`Dao`、`RoomDatabase`、`Retrofit`。
- 协程风险：`GlobalScope`、`runBlocking`、`Thread.sleep`、`Dispatchers.`、`catch {`、`collect {`、`stateIn(`、`shareIn(`。
- 测试可替换性：`@Inject constructor`、`private val` 构造注入、`Dispatchers.IO`、`Clock.System`、`System.currentTimeMillis`、真实网络或数据库初始化。

## 执行规则

- 只做代码层质量审查，不跑 API 请求。
- 不默认执行任何 Git 操作。
- 不自动修复问题，只输出问题和修复建议。
- 不照搬强制调用、自动修复、hooks、后台观察逻辑。
- 输出必须包含文件路径、行号、问题等级、原因、建议；能引用命中代码时，应包含最小必要代码片段。
- 报告输出遵循 `test-quality-report-template`。
- 如需要修改代码，必须等待用户确认后逐项处理。
- 多文件修改必须按安全批量修改规则执行。
- 中文路径和中文内容必须保持 UTF-8。

## 输出要求

按阻断、高风险、中风险、建议优化分组。无法确认的问题要说明原因和下一步验证建议。

每个问题使用以下字段：

- 文件路径：相对项目根目录或用户提供的绝对路径。
- 行号：具体行号；多处同类问题可列出代表性行号和搜索范围。
- 问题等级：阻断、高风险、中风险、建议优化。
- 问题：一句话说明违反的架构规则。
- 原因：说明为什么这会破坏 Clean Architecture、测试性、可维护性或运行稳定性。
- 建议：给出可执行修改方向，例如移动接口、引入 UseCase、增加 Mapper、注入 Dispatcher、改为统一错误模型。

示例格式：

```markdown
### 高风险

- 文件路径：`feature/user/presentation/UserViewModel.kt`
- 行号：42
- 问题等级：高风险
- 问题：ViewModel 直接依赖 `UserApiService`，绕过了 UseCase 和 Repository 抽象。
- 原因：Presentation 层绑定网络实现细节，业务流程无法在 JVM 单元测试中替换，数据源策略也会散落到 UI 编排代码。
- 建议：在 Domain 定义 `UserRepository` 和对应 UseCase，Data 层实现网络访问，ViewModel 只依赖 UseCase。
```

## 内置规则覆盖

- 依赖方向：Presentation 只能依赖 Domain，Data 只能实现 Domain 抽象，禁止 UI 层直接依赖 Retrofit、Room、SQLDelight 或平台实现。
- Domain 纯净度：UseCase、实体和值对象不应依赖 Android SDK、数据库 DTO、网络模型、DI 注解或 UI 状态类型。
- Repository 边界：Domain 定义接口，Data 提供实现；检查缓存、远程数据、本地数据、错误映射和离线策略是否集中在 Data 层。
- UseCase：每个 UseCase 表达单一业务动作，输入输出明确，避免 ViewModel 承载业务规则、分页策略或数据源选择。
- ViewModel：只做状态编排和事件分发，检查 `StateFlow`/`SharedFlow`、取消、加载/错误状态、一次性事件和配置变更恢复。
- 协程与 Flow：检查 Dispatcher 注入、结构化并发、异常传播、`flowOn` 位置、冷/热流语义和测试可控性。
- 数据层专项：Room DAO、SQLDelight query、Ktor/Retrofit client、分页和 mapper 必须留在 Data 层，不向 Domain/Presentation 泄漏实现类型。
- DI 与构建：检查 Hilt/Koin scope、测试替换模块、Gradle convention plugin、模块依赖和公共依赖版本是否统一。
- 测试替换性：Repository、UseCase、Dispatcher、Clock、网络和数据库依赖应能在 JVM 测试中替换，覆盖成功、空数据、异常和取消场景。
