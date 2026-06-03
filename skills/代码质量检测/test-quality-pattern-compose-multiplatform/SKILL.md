---
name: "test-quality-pattern-compose-multiplatform"
description: "Use when reviewing Compose Multiplatform or Jetpack Compose state, navigation, theming, performance, or platform-specific UI."
---
# Compose Multiplatform 模式质量审查

## 定位

用于审查 Compose Multiplatform 与 Jetpack Compose 项目的 UI 架构、状态管理、性能、导航、跨平台实现和可测试性。适用于 KMP 的 `commonMain`、`androidMain`、`iosMain`、`desktopMain`、`wasmJsMain` 等源码，也适用于纯 Android Jetpack Compose 模块。

## 检查重点

- 状态提升：业务状态是否上移到 ViewModel、Presenter 或状态持有者；可组合函数是否保持无状态、可预览、可测试。
- 单向数据流：屏幕是否使用单一 UI State、事件入口是否清晰，避免深层组件直接修改共享状态。
- 重组性能：是否存在在 `@Composable` 中执行重计算、频繁分配对象、无稳定 key 的 Lazy 列表、不可跳过的参数传递。
- `remember` 与 `derivedStateOf`：是否只缓存可由输入决定的 UI 派生值，key 是否完整；滚动状态、筛选结果、昂贵格式化是否避免每次重组重复计算。
- Side-effect API：`LaunchedEffect`、`DisposableEffect`、`SideEffect`、`produceState`、`rememberUpdatedState` 是否按生命周期和 key 使用，避免用 `LaunchedEffect(Unit)` 承载业务初始化。
- Navigation：是否将 `NavController` 限制在导航层；页面组件通过 lambda 或事件导航；路由参数是否类型安全、可序列化、可恢复。
- UI 分层：Screen、Content、Design System、业务组件和平台适配是否分层明确，避免 Composable 直接访问仓储、数据库、网络或平台 API。
- 跨平台实现：`expect/actual` 是否只封装平台差异；公共层是否避免引用 Android/iOS 专属类型；平台实现是否保持行为一致。
- 资源管理：字符串、图片、字体、颜色、尺寸、主题 token 是否使用项目资源体系；避免硬编码文案、颜色和平台路径。
- 可测试性：核心状态转换是否能脱离 UI 测试；Composable 是否有稳定输入、语义标识、Preview 或 UI 测试入口。
- 可访问性与交互：重要组件是否提供语义、点击区域、焦点顺序、键盘/桌面交互和屏幕阅读支持。

## 执行规则

- 只做代码层质量审查，不跑 API 请求。
- 不默认执行任何 Git 操作。
- 不自动修复问题，只输出问题和修复建议。
- 不照搬外部流程、自动修复、hooks、后台观察逻辑。
- 输出必须包含文件路径、行号、问题等级、原因、建议。
- 报告输出遵循 `test-quality-report-template`。
- 如需要修改代码，必须等待用户确认后逐项处理。
- 多文件修改必须按安全批量修改规则执行。
- 中文路径和中文内容必须保持 UTF-8。

## 怎么检查

### 1. 确认范围和技术栈

- 先查找 `build.gradle.kts`、`settings.gradle.kts`、`libs.versions.toml`、`composeApp`、`commonMain`、`androidMain`、`iosMain` 等目录，确认是否为 Compose Multiplatform、Android Compose 或混合项目。
- 记录被审查模块、平台源码集、Compose 版本线索、导航库、依赖注入和状态管理方案。
- 不执行 Git 命令；如需要构建、测试或运行 UI，先说明目的并等待用户确认。

### 2. 多平台工具映射

- Codex：内容搜索优先 `rg`，文件查找优先 `rg --files`，读取文件用 `Get-Content -LiteralPath '<path>' -Encoding UTF8`。
- Claude：文件查找优先 `Glob`，内容搜索优先 `Grep`，读取文件用 `Read`。
- Trae：代码搜索优先 `SearchCodebase`，读取文件用 `Read`。
- 任一平台都必须在报告中引用具体文件路径和行号；搜索结果要能回溯到实际代码片段。

### 3. 建议搜索入口

- 查状态入口：`StateFlow`、`MutableStateFlow`、`mutableStateOf`、`rememberSaveable`、`collectAsState`、`collectAsStateWithLifecycle`。
- 查重组风险：`@Composable`、`remember(`、`derivedStateOf`、`LazyColumn`、`LazyRow`、`items(`、`key =`、`@Stable`、`@Immutable`。
- 查副作用：`LaunchedEffect`、`DisposableEffect`、`SideEffect`、`produceState`、`snapshotFlow`、`rememberCoroutineScope`、`rememberUpdatedState`。
- 查导航：`NavHost`、`NavController`、`navigate(`、`composable<`、`toRoute`、`dialog(`、`popBackStack`。
- 查跨平台：`expect fun`、`actual fun`、`commonMain`、`androidMain`、`iosMain`、`desktopMain`、`LocalContext`、`UIKit`、`NS`。
- 查资源：`stringResource`、`painterResource`、`FontFamily`、`MaterialTheme`、`Res.string`、`Res.drawable`、硬编码中文/英文文案、硬编码颜色。
- 查测试：`composeTestRule`、`runComposeUiTest`、`testTag`、`semantics`、`Preview`、状态 reducer 或事件处理单元测试。

### 4. 判断规则

- 状态提升：如果子 Composable 持有业务状态、直接调用仓储/网络/数据库、或多个组件各自维护同一状态，标为中风险或高风险；建议改为单一 UI State + 事件回调。
- 状态收集：Android 端优先检查是否使用生命周期感知收集；跨平台公共层检查是否封装了平台差异，避免直接依赖 Android lifecycle API。
- `remember`：如果缓存依赖缺少 key、缓存了应由 ViewModel 管理的业务状态、或 `remember` 中读取不稳定外部变量，标为中风险。
- `derivedStateOf`：只在输入频繁变化但输出阈值变化较少时建议使用；不要把普通字符串拼接、简单布尔值一律要求改成 `derivedStateOf`。
- 副作用：检查 effect key 是否覆盖真实依赖；长期监听必须用 `DisposableEffect` 清理；回调闭包变化但 effect 不应重启时使用 `rememberUpdatedState`。
- 重组性能：Lazy 列表缺稳定 key、Composable 参数传可变集合、重组中做排序/过滤/格式化/创建对象、Modifier 或 lambda 导致大量无效重组时，按影响定级。
- Navigation：深层 UI 直接持有 `NavController`、字符串拼路由、参数未编码或不可恢复、导航和业务逻辑耦合时，至少标为中风险。
- UI 分层：公共 UI 层依赖平台上下文、业务组件包含数据访问逻辑、Design System 混入页面业务规则时，按影响定级。
- 跨平台：`commonMain` 出现平台专属 API、`expect/actual` 行为不一致、资源或权限逻辑只在单平台处理时，按崩溃或功能缺失风险定级。
- 资源管理：可本地化文本硬编码、图片/字体跨平台路径不一致、主题 token 被绕过、颜色尺寸散落在页面中时，输出建议优化或中风险。
- 可测试性：状态转换只能通过 UI 触发、关键 Composable 缺少语义标识、复杂分支无法 Preview 或 UI 测试覆盖时，按业务重要性定级。

## 输出要求

按阻断、高风险、中风险、建议优化、无法确认的风险分组。无法确认的问题要说明原因和下一步验证建议。每个问题必须包含：

- 文件路径
- 行号
- 问题等级
- 问题描述
- 原因或风险
- 修改建议

建议使用下列 Markdown 表格格式：

```markdown
## 高风险问题

| 文件 | 行号 | 问题等级 | 问题 | 原因/风险 | 建议 |
| ---- | ---- | ---- | ---- | ---- | ---- |
| app/src/commonMain/kotlin/AppScreen.kt | 42 | 高风险 | LazyColumn 未设置稳定 key | 列表增删会导致状态错位和无效重组 | 使用业务唯一 id 作为 `items(..., key = { it.id })` |
```

## 内置规则覆盖

- 状态提升：检查 state hoisting、单向数据流、事件回调命名、UI state 不可变和 ViewModel/Presenter 边界。
- Composable API：检查 slot 参数设计、`Modifier` 是否作为首个可选参数并向根节点传递、默认参数是否稳定。
- 重组性能：检查 `remember`、`rememberSaveable`、`derivedStateOf`、`LaunchedEffect` key、`snapshotFlow`、稳定 key 和大列表重组。
- 副作用：检查 `DisposableEffect` 清理监听器、协程 scope 生命周期、重复请求、导航事件和一次性事件消费。
- 跨平台隔离：检查 `commonMain` 是否泄漏 Android/iOS API，`expect/actual` 边界、平台权限、文件/网络/时间依赖是否可替换。
- 导航与资源：检查导航状态、返回栈、深链、资源引用、字符串国际化、图片尺寸和无障碍语义。
- 主题体系：检查 Material 3 token、dynamic color 使用边界、暗色模式、字体尺寸、颜色对比和组件主题覆盖方式。
- 测试覆盖：检查 preview、snapshot、UI state reducer、Composable 交互、导航和跨平台逻辑是否可测试。
