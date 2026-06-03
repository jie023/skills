---
name: "test-quality-pattern-flutter"
description: "Use when reviewing Dart or Flutter null safety, widget architecture, state management, async/error handling, routing, forms, platform adaptation, assets, performance, and tests."
---
# Flutter 模式质量审查

## 定位

用于审查 Dart / Flutter 项目的代码质量、架构模式和可维护性风险。重点判断代码是否符合 Flutter 组件化、Dart null safety、状态管理、异步错误处理、导航、表单、平台适配、资源管理、性能和测试实践。

本 skill 只做代码层质量审查，不默认修改代码、不默认运行构建测试、不默认执行 Git 操作、不发起 API 请求。如用户要求修复，必须按确认的问题逐项处理。

## 适用范围

- Flutter 页面、Widget、Form、路由、主题、资源、平台适配代码。
- Dart model、repository、service、state、controller、provider、bloc、cubit、notifier 等代码。
- Dio/http 网络层、错误模型、异步加载、缓存、本地存储、权限和平台通道相关实现。
- Flutter unit test、widget test、golden test、integration test 的覆盖质量。

## 多平台检查方式

审查时优先使用当前平台可用的代码搜索和读取工具，输出中引用的问题必须带文件路径和行号。

- Codex：优先用 `rg` / `rg --files` 搜索，读取文件可用 `Get-Content -Encoding UTF8`，复杂任务用 `update_plan` 跟踪。
- Claude：优先用 `Glob` 查找文件、`Grep` 搜索内容、`Read` 读取具体文件，复杂任务用可用 Todo 工具跟踪。
- Trae：优先用 `SearchCodebase` 搜索代码、`Read` 读取具体文件，复杂任务用可用任务清单工具跟踪。

建议检查入口：

- `pubspec.yaml`：确认 Flutter SDK、依赖、资源、字体、生成器、测试依赖。
- `lib/**/*.dart`：检查业务代码、页面、状态、路由、服务、模型和平台适配。
- `test/**/*.dart`、`integration_test/**/*.dart`：检查测试覆盖与断言质量。
- `analysis_options.yaml`：确认 lint、严格模式、生成代码排除和团队规则。

## 检查项与怎么检查

### 1. Flutter / Dart Null Safety

- 搜索 `!`、`late`、`as`、`dynamic`、`Map<String, dynamic>`、`firstWhere`、`context!` 等高风险写法。
- 检查 `!` 是否有可靠前置判空；无保证的强制解包按高风险或中风险记录。
- 检查 `late` 是否只用于生命周期内必然初始化的字段，如 `AnimationController`；业务字段用 `late` 延迟报错应记录。
- 检查 nullable model 字段是否通过 `?.`、`??`、pattern matching、early return 或 sealed state 明确处理。
- 检查 JSON 解析是否处理缺字段、类型不匹配、空数组、空对象和后端返回 null。

### 2. Widget 拆分与组件结构

- 搜索大型 `build` 方法、`_buildXxx()` 返回 Widget、深层嵌套 `Column` / `Row` / `Stack`。
- 检查复杂页面是否拆成独立 `StatelessWidget` / `StatefulWidget`，而不是大量私有方法拼接 UI。
- 检查可复用组件是否暴露清晰参数，避免直接读取上层页面状态或全局变量。
- 检查 `const` 是否能传播到静态 Widget、样式、间距和 Icon，减少不必要 rebuild。
- 检查列表项、表单项、弹窗、空状态、错误状态是否有明确组件边界。

### 3. State 管理

- 识别项目使用的状态方案：`setState`、Provider、Riverpod、BLoC/Cubit、GetX、MobX 或自定义方案。
- 检查状态是否不可变，更新是否使用 `copyWith`、sealed class、Freezed 或明确的 value object。
- 检查 `setState` 是否只管理局部 UI 状态；跨页面、异步业务状态不应散落在 Widget 内。
- 检查 Provider/Riverpod/BLoC 的 watch/read/listen/select 使用是否导致过度 rebuild 或读写时机错误。
- 检查 loading、empty、success、failure 是否是互斥状态，避免多个 bool 组合出不可能状态。

### 4. 异步与错误处理

- 搜索 `async`、`await`、`Future`、`Stream`、`catchError`、`try`、`on Exception`。
- 检查 `await` 后是否使用 `mounted` / `context.mounted` 再访问 `BuildContext`、`Navigator`、`ScaffoldMessenger`、`setState`。
- 检查异步请求是否有 loading、成功、失败和取消后的状态处理，避免重复提交和悬挂 loading。
- 检查异常是否按业务异常、网络异常、解析异常、未知异常分类处理，避免吞异常或只 `print`。
- 检查 Stream、Timer、AnimationController、TextEditingController、FocusNode、ScrollController 是否正确释放。

### 5. 路由与导航

- 识别路由方案：Navigator 1.0/2.0、GoRouter、AutoRoute 或自定义路由。
- 检查鉴权路由、重定向、深链、参数解析、未知路由和返回栈是否有一致策略。
- 检查路由参数是否强类型或集中解析，避免页面内散落字符串 key 和强转。
- 检查弹窗、bottom sheet、二级页面返回值是否处理取消和异常路径。
- 检查异步回调中导航前是否验证页面仍挂载。

### 6. 表单与输入校验

- 搜索 `Form`、`TextFormField`、`TextEditingController`、`validator`、`autovalidateMode`。
- 检查表单是否使用 `GlobalKey<FormState>` 或等价机制统一校验。
- 检查必填、长度、格式、范围、前后空格、跨字段依赖和服务端错误回填。
- 检查提交按钮是否防重复点击，提交中是否禁用输入或展示进度。
- 检查 controller、focus node 是否在 `dispose` 中释放。

### 7. 平台适配与权限

- 检查是否区分 Android、iOS、Web、Desktop 的能力差异，避免无保护地调用平台专属 API。
- 搜索 `Platform.isXxx`、`kIsWeb`、`defaultTargetPlatform`、`MethodChannel`、权限库调用。
- 检查权限申请、拒绝、永久拒绝、系统设置跳转、降级 UI 是否完整。
- 检查安全区域、键盘遮挡、横竖屏、暗色模式、字体缩放和本地化是否被考虑。
- 检查平台通道是否有超时、异常捕获、返回值类型校验和 fallback。

### 8. 资源、主题与本地化

- 检查 `pubspec.yaml` 中 assets、fonts 声明是否与代码引用一致。
- 搜索硬编码颜色、字号、文案、路径、魔法数字，判断是否应迁移到主题、常量或本地化资源。
- 检查图片资源是否按分辨率、格式和体积合理组织，避免运行时加载不存在资源。
- 检查国际化项目是否使用 arb/generated l10n，避免页面中直接写固定文案。
- 检查主题是否覆盖亮暗色、错误色、输入框、按钮、文本层级和无障碍对比度。

### 9. 性能与渲染

- 搜索 `ListView(`、`GridView(`、`SingleChildScrollView`、`shrinkWrap: true`、`IntrinsicHeight`、`Opacity`、`BackdropFilter`。
- 检查长列表是否使用 builder/sliver，避免一次性构建大量子节点。
- 检查 rebuild 范围是否过大，状态监听是否能用 `select`、拆分 Widget 或局部 builder 缩小范围。
- 检查图片是否有缓存、尺寸约束、占位和错误态，避免大图直接进内存。
- 检查昂贵计算、JSON 解析、排序过滤是否放在 build 中；必要时移到 isolate、memo 或状态层。

### 10. 测试

- 检查业务逻辑是否有 unit test，状态管理是否覆盖 loading/success/failure/边界输入。
- 检查关键 Widget 是否有 widget test，覆盖空状态、错误状态、表单校验、点击和导航。
- 检查 repository/service 是否使用 fake 或 mock 隔离网络、数据库、平台通道。
- 检查黄金图或视觉回归测试是否覆盖核心 UI 状态；没有时记录剩余风险而非臆断问题。
- 检查 integration test 是否覆盖关键用户流程，尤其登录、表单提交、列表刷新和错误恢复。

## 风险等级建议

- 阻断：会导致生产崩溃、数据丢失、权限绕过、支付/登录等核心流程不可用的问题。
- 高风险：明显运行时异常、重要状态错乱、严重内存泄漏、关键流程无错误处理或核心测试缺失。
- 中风险：局部可维护性差、边界处理不足、平台适配不完整、异常信息不可诊断。
- 建议优化：不阻断交付，但可提升性能、可读性、一致性或测试质量的问题。

## 输出要求

报告必须遵循 `test-quality-report-template`，按阻断、高风险、中风险、建议优化、无法确认的风险分组。每个确认问题必须包含：

- 文件路径
- 行号
- 问题等级
- 问题描述
- 原因 / 风险
- 建议

建议使用表格输出：

```markdown
## 高风险问题

| 文件 | 行号 | 问题等级 | 问题 | 原因 / 风险 | 建议 |
| ---- | ---- | ---- | ---- | ---- | ---- |
| lib/example/page.dart | 42 | 高风险 | await 后直接使用 context | 页面销毁后继续导航可能触发运行时异常 | 导航前检查 mounted 或 context.mounted |
```

没有发现问题时也必须输出完整报告，写明检查范围、使用 skill、无法确认项和剩余风险，不得只回复“没有问题”。无法确认的问题要说明缺少的信息和下一步验证建议。

## 内置规则覆盖

- Null safety：检查可空链路、`late`、强制解包、默认值、JSON 解析和模型转换失败处理。
- Widget 结构：检查 build 方法过重、重复组件、状态与 UI 混杂、`const` 使用、key 稳定性和 dispose 清理。
- 状态管理：检查 Provider/Riverpod/BLoC/GetX 等状态边界、异步状态、错误状态、刷新和页面销毁后的更新。
- 网络专项：检查 Dio interceptor、鉴权 token 注入、refresh token 单次重试保护、401 防循环、超时、取消请求和错误归一。
- 路由鉴权：检查 GoRouter `redirect`、`refreshListenable`、深链、登录跳转、返回栈、参数校验和开放跳转。
- 表单与平台能力：检查校验触发、防重复提交、权限申请、平台差异、文件/相机/定位异常和用户拒绝路径。
- 全局错误：检查 `FlutterError.onError`、`PlatformDispatcher.onError`、`ErrorWidget.builder`、Crashlytics/日志脱敏和上报降级。
- 资源与性能：检查图片尺寸、缓存、主题、国际化、列表虚拟化、重绘边界、动画控制器和启动性能。
- 测试覆盖：检查 widget test、golden test、状态单测、网络 mock、路由守卫和平台通道 mock。
