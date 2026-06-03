---
name: "test-quality-language-arkts-review"
description: "Use when reviewing ArkTS or HarmonyOS code quality, security, testing, V2 state management, navigation, permissions, storage, or build validation."
---
# ArkTS 语言专项质量审查

## 定位

ArkTS / HarmonyOS 专项代码质量审查，重点覆盖 ArkTS 编译约束、HarmonyOS 权限、安全存储、V2 状态管理、Navigation、MVVM、资源引用、测试和依赖安全。

## 适用范围

- 用户要求审查 ArkTS、HarmonyOS、`.ets`、`module.json5`、`oh-package.json5`、ArkUI、Navigation、权限或 V2 状态管理时使用。
- 涉及组件、ViewModel、Model、Service、深链、权限、网络、存储、资源、测试和 HAP 构建配置时使用。
- 默认只做静态审查，不启用 hooks，不默认运行 `hvigorw`、`ohpm`、设备测试或 API 请求；需要运行时先说明目的并等待确认。

## 检查流程

1. 明确范围：优先审查用户指定的 `.ets`、`.ts`、`module.json5`、`oh-package.json5`、`build-profile.json5` 和测试文件。
2. 识别项目结构：确认 entry/module、Ability、ArkUI 组件、MVVM 分层、资源目录、ohosTest 测试目录。
3. 读取上下文：从页面/组件到 ViewModel、Model、Service、权限声明、资源和测试建立调用链。
4. 按分级检查项判断：先编译约束、安全、权限、状态/导航，再看性能、测试和风格。
5. 每个问题必须包含文件路径、行号、证据、风险、建议和验证方式；无法确认时说明缺少的信息。

## 分级检查项

### 阻断

- **ArkTS 编译约束违规**：使用 `any/unknown`、条件类型、infer、交叉类型、mapped type、`typeof` 类型标注、`as const`、spread、动态字段访问、解构、对象字面量方法、`var`、`require` 等 ArkTS 不支持或受限写法。
  - 怎么检查：搜索 TypeScript 高级类型、`...`、`obj["field"]`、`delete`、`for...in`、`var`、`require`、解构语法、对象字面量方法和 `catch (e: any)`。
- **V1 状态管理装饰器**：使用 `@State`、`@Prop`、`@Link`、`@ObjectLink`、`@Observed`、`@Provide`、`@Consume`、`@Watch`、`@Component`。
  - 怎么检查：搜索 V1 装饰器，确认是否应改为 `@ComponentV2`、`@Local`、`@Param`、`@Event`、`@Provider`、`@Consumer`、`@Monitor`、`@ObservedV2`、`@Trace`。
- **权限声明或运行时授权缺失**：调用相机、定位、网络、文件等系统 API 未在 `module.json5` 声明，敏感权限未检查/请求。
  - 怎么检查：对照 API 调用和 `requestPermissions`，确认 reason string、usedScene、运行时请求、拒绝 fallback 和二次进入页面的状态恢复。
- **敏感信息泄露或不安全存储**：API key、token、密码硬编码，敏感凭据存 Preferences 或日志。
  - 怎么检查：搜索密钥字段、Preferences 写入、hilog 输出、网络日志；普通 Preferences 不存敏感凭据，敏感数据应使用 HUKS/Keystore 或加密存储，并区分缓存、配置和长期凭据。
- **不安全导航或深链**：外部 URI/参数未白名单校验就进入 `NavPathStack`，或仍使用 `@ohos.router`。
  - 怎么检查：搜索 deep link、`pushPath/replacePath`、`@ohos.router`，确认路径名、参数类型和权限校验。

### 高风险

- **输入输出校验不足**：用户输入、接口响应、deep link、存储数据未校验就进入 ViewModel/Service/UI。
  - 怎么检查：从页面事件、Form、网络响应和 deep link 追踪到业务逻辑和导航。
- **状态管理不符合 V2 响应式规则**：`@ObservedV2` 类属性缺 `@Trace`，`@Param` 被直接修改，Provider/Consumer key 不一致。
  - 怎么检查：读取组件和模型，确认状态来源、修改点、子组件传参和监听逻辑。
- **MVVM 分层混乱**：`build()` 内写业务逻辑、网络请求或数据持久化，ViewModel 直接处理 UI 细节。
  - 怎么检查：查看 Component、ViewModel、Service 的职责边界和依赖方向。
- **网络安全不足**：非 HTTPS、缺超时/重试、证书校验缺失、请求/响应日志输出敏感数据。
  - 怎么检查：搜索 HTTP client、URL、timeout、retry、hilog 和 interceptor。
- **资源和主题不完整**：新增字符串、颜色、字号硬编码，缺 i18n、深色主题或资源引用。
  - 怎么检查：搜索硬编码文本、颜色、尺寸，确认是否使用 `$r()` 和多资源目录。
- **测试覆盖缺失**：ViewModel、Service、V2 状态、Navigation、权限拒绝、网络失败没有测试。
  - 怎么检查：查 `ohosTest/ets/test`，确认是否有 Hypium/UI 测试、断言和失败路径。

### 中风险

- **文件和组件组织不合理**：一个 `.ets` 多个 `@ComponentV2`，ViewModel/Model 文件过长或职责混杂。
  - 怎么检查：按文件读取组件数量、类数量、行数和目录分层。
- **类型标注和返回类型不完整**：方法、参数、返回值缺显式类型，Record 取值未处理 `undefined`。
  - 怎么检查：搜索函数签名、Record 访问、catch clause 和对象字面量使用。
- **动画和列表性能风险**：频繁动画修改 width/height/padding/margin，大列表不用 `LazyForEach`，key 不稳定。
  - 怎么检查：搜索 `animation`、`animateTo`、`ForEach`、`LazyForEach`、List key 和 renderGroup。
- **依赖和构建配置风险**：`oh-package.json5` 依赖来源不明、版本漂移，`build-profile.json5` 环境配置不清。
  - 怎么检查：查看依赖版本、registry、build profile、product/module 配置。
- **错误处理和日志不规范**：catch 后只打印或吞异常，hilog 缺上下文或输出敏感字段。
  - 怎么检查：搜索 `try/catch`、`hilog`、`throw new Error`，确认用户提示和内部日志分离。

### 建议优化

- **命名和格式**：变量/函数 camelCase，类/接口 PascalCase，常量 UPPER_SNAKE_CASE，字符串优先双引号并加分号。
- **组件复用**：可抽取通用组件、`@Builder` 片段和明确 `@Param/@Event`。
- **不可变更新**：避免直接修改模型对象，使用新实例或清晰更新方法。
- **测试组织**：测试命名采用 `should_[expected]_when_[condition]`，避免共享可变状态。
- **构建与设备测试验证**：无法静态确认编译或设备行为时，建议验证 `hvigorw assembleHap`、`ohpm install`、Hypium/ohosTest 或目标设备测试命令，但默认不自动运行。

## 常用搜索线索

按当前平台工具执行搜索，结果必须包含文件路径、行号和具体命中内容。

```text
\\bany\\b|\\bunknown\\b|infer|typeof .*:|as const|\\.\\.\\.|\\[\"|\\['|delete |for\\s*\\(.* in |\\bvar\\b|require\\(
\\{\\s*[A-Za-z_][A-Za-z0-9_]*\\s*\\(|catch\\s*\\(.*:\\s*any\\)|object literal
@State|@Prop|@Link|@ObjectLink|@Observed\\b|@Provide|@Consume|@Watch|@Component\\b|@ohos\\.router
@ComponentV2|@Local|@Param|@Event|@Provider|@Consumer|@ObservedV2|@Trace|@Monitor
requestPermissions|checkAccessToken|ohos.permission|module.json5|usedScene|reason
Preferences|huks|HUKS|Keystore|encrypt|decrypt|password|secret|token|apiKey|hilog\\.
NavPathStack|pushPath|replacePath|pop\\(|clear\\(|deepLink|URL\\(
ForEach|LazyForEach|animation|animateTo|renderGroup|width\\(|height\\(|padding\\(|margin\\(
describe\\(|it\\(|expect\\(|@ohos.UiTest|ohosTest
hvigorw|assembleHap|ohpm install|Hypium|hdc
```

## 判断标准

- 阻断：会导致 ArkTS 编译失败、权限/导航/存储安全风险、V1 状态管理违规或敏感信息泄露。
- 高风险：状态同步、MVVM 分层、网络安全、资源主题、关键测试存在明显质量风险。
- 中风险：类型、组件组织、性能、依赖、日志和错误处理有维护隐患。
- 建议优化：命名、格式、组件复用、不可变更新和测试组织可改进。

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
  - 位置：path/to/File.ets:line
  - 证据：命中的关键代码或配置
  - 原因：为什么违反 ArkTS/HarmonyOS 质量要求
  - 建议：如何修改或下一步验证
```

没有发现对应等级问题时写“未发现”。无法确认的问题要说明原因和下一步验证建议。

## 内置规则覆盖

- ArkTS 规范：类型安全、组件状态、生命周期、装饰器、资源引用、模块边界和命名一致性都纳入审查。
- HarmonyOS 安全：权限申请、用户授权、Ability/Want 参数、文件访问、网络请求和敏感信息存储要重点检查。
- 编译限制：spread、对象字面量方法、动态字段访问、解构、`any/unknown`、条件类型、mapped type 等 ArkTS 受限写法要优先排查。
- 状态管理：`@State`、`@Prop`、`@Link`、`@Provide`/`@Consume`、全局状态和异步更新要避免状态错乱。
- UI 与性能：列表 key、重复渲染、资源加载、图片体积、动画、布局嵌套和生命周期内耗时任务要结合设备场景判断。
- 测试：UI 状态、权限分支、Ability 跳转、服务调用、错误态和多设备适配需要可验证覆盖。

## 多平台兼容

- Codex：优先使用 `rg`、`rg --files`、`Get-Content -Encoding UTF8`，复杂任务使用 `update_plan`。
- Claude：优先使用 `Read`、`Grep`、`Glob`，复杂任务使用可用的 Todo 工具。
- Trae：优先使用 `SearchCodebase`、`Read`，复杂任务使用可用的任务清单工具。
- 任一平台都必须遵守：不默认执行 Git、不自动修复、不批量改文件、中文内容保持 UTF-8。
