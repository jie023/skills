---
name: "jie023-test-quality-engineer"
description: "Use when coordinating code-level quality testing for completed modules, files, or directories without running API requests."
---
# 代码质量测试工程师

## 定位

质量测试总入口，负责判断要调用哪些 `test-quality-*` 子 skill，并汇总代码质量测试报告。它只负责调度、分级和汇总，不替代具体专项 skill 的检查清单。

## 触发场景

- 模块写完后，需要做代码层质量测试。
- 用户要求检查代码质量、安全问题、坏味道、数据库访问质量或测试缺口。
- 用户明确要求不要跑接口，只看代码质量。

## 调度流程

1. 确认检查范围：模块路径、文件路径、目录路径或用户描述的范围。
2. 判断技术栈、框架、构建方式和关注点。
3. 先调用通用质量 skill，再按语言、框架、模式、构建问题追加专项 skill。
4. 汇总阻断、高风险、中风险、建议优化和无法确认的风险。
5. 输出报告时调用 `test-quality-report-template`。
6. 发现问题后先报告，等待用户确认后再逐项修复。
7. 如果需要改多个文件，按安全批量修改规则逐项处理。

## 默认调用组合

- 通用代码质量：`test-quality-common-review`
- 代码层安全：`test-quality-common-security-review`
- 输入校验：`test-quality-common-input-validation`
- 数据库访问质量：涉及 SQL、Mapper、Repository、Migration 时调用 `test-quality-common-database-review`
- 性能质量：涉及慢查询、缓存、循环调用、渲染或资源释放时调用 `test-quality-common-performance-review`
- 测试质量：涉及测试覆盖、回归、边界、权限或安全修复测试时调用 `test-quality-common-testing-review`
- 代码简化：涉及复杂分支、重复逻辑、过度抽象或可读性问题时调用 `test-quality-common-code-simplifier`
- 语言专项：按技术栈调用 `test-quality-language-*`
- 框架/模式专项：涉及 Spring Boot、Vue、Django、NestJS、Android、Flutter 等架构模式时调用 `test-quality-pattern-*`
- 构建专项：涉及编译、依赖、打包、构建配置或 CI 失败时调用 `test-quality-build-*`
- 报告模板：`test-quality-report-template`

## 技术栈路由

- Java / Spring / MyBatis：`test-quality-language-java-review`，必要时追加 `test-quality-pattern-java`、`test-quality-pattern-springboot`。
- Vue：`test-quality-language-vue-review`，必要时追加 `test-quality-pattern-vue` 和 `test-quality-language-typescript-review`。
- TypeScript / JavaScript：`test-quality-language-typescript-review`。
- Python / Django / FastAPI：`test-quality-language-python-review`，Django 项目追加 `test-quality-pattern-django`。
- Go：`test-quality-language-go-review`。
- Rust：`test-quality-language-rust-review`。
- Kotlin / Android：`test-quality-language-kotlin-review`，Android 架构追加 `test-quality-pattern-android-clean-architecture`。
- C++：`test-quality-language-cpp-review`，需要规范或测试专项时追加 `test-quality-pattern-cpp-standards`、`test-quality-pattern-cpp-testing`。
- PHP / ArkTS / F#：分别调用对应 `test-quality-language-*`。

## 统一规则

- 不默认执行任何 Git 操作。
- 不自动修复问题。
- 不跑 API 请求。
- 不照搬强制调用、自动修复、hooks、后台观察逻辑。
- 输出必须包含文件路径、行号、问题等级、原因、建议。
- 报告输出遵循 `test-quality-report-template`。
- 多文件修改必须按安全批量修改规则执行。
- 中文路径和中文内容必须保持 UTF-8。
- 不要求固定存在 `.claude`、`.codex`、`.trae` 等平台专属目录；按当前平台可用工具执行。

## 内置调度规则覆盖

- 通用审查已下沉到 `test-quality-common-*`：代码结构、安全、输入校验、数据库、性能、测试覆盖和代码简化。
- 语言专项已下沉到 `test-quality-language-*`：Java、Vue、TypeScript、Python、Go、Rust、Kotlin、C++、F#、PHP、ArkTS。
- 框架/模式专项已下沉到 `test-quality-pattern-*`：Java CRUD、Spring Boot、Backend、API Design、Database Migration、Vue、Django、NestJS、Android、Compose、Flutter、C++ 标准和 C++ 测试。
- 构建专项已下沉到 `test-quality-build-*`：通用构建、Node、Java、Kotlin、Go、Rust、C++。
- 报告格式已下沉到 `test-quality-report-template`，总入口只负责选择、组合和汇总，不保留离线归档。
- 不迁入旧资料中的 hooks、后台观察、自动修复、默认 Git、斜杠命令和平台强绑定流程。

## 多平台兼容

- Codex：优先使用 `rg`、`rg --files`、`Get-Content -Encoding UTF8`，复杂任务使用 `update_plan`。
- Claude：优先使用 `Read`、`Grep`、`Glob`，复杂任务使用可用的 Todo 工具。
- Trae：优先使用 `SearchCodebase`、`Read`，复杂任务使用可用的任务清单工具。
- 任一平台都必须遵守：不默认执行 Git、不自动修复、不批量改文件、中文内容保持 UTF-8。
