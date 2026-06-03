---
name: "jie023-test-quality-engineer"
description: "Use when coordinating code-level quality testing for completed modules, files, or directories without running API requests, including incremental Markdown report output during inspection."
---
# 代码质量测试工程师

## 定位

质量测试总入口，负责判断要调用哪些 `test-quality-*` 子 skill，并汇总代码质量测试报告。它只负责调度、分级和汇总，不替代具体专项 skill 的检查清单。

## 触发场景

- 模块写完后，需要做代码层质量测试。
- 用户要求检查代码质量、安全问题、坏味道、数据库访问质量或测试缺口。
- 用户明确要求不要跑接口，只看代码质量。

## 调度流程（强制）

**以下流程必须严格执行，不得跳过任何步骤，不得替代子 skill 的检查工作。**

1. 确认检查范围：模块路径、文件路径、目录路径或用户描述的范围。
2. 判断技术栈、框架、构建方式和关注点。
3. 确定 Markdown 报告路径：优先使用用户指定路径；否则在检查范围根目录生成 `代码质量检查报告.md`。
4. **强制创建报告骨架**：检查开始前必须创建报告骨架，包含检查范围、检查时间、技术栈、**已调用/待调用子 skill 清单**、当前结论和问题统计。
5. **强制调用子 skill**：
   - **必须**按顺序调用所有适用的通用 skill（见"默认调用组合"）。
   - **必须**根据技术栈调用对应的语言专项 skill（见"技术栈路由"）。
   - **必须**根据框架/架构模式调用对应的模式专项 skill。
   - **严禁**因"代码简单"、"已经看过"、"时间不够"等理由跳过任何子 skill。
   - 每个子 skill 调用前，在报告中记录调用状态；调用后，将子 skill 返回的结果合并到报告中。
6. 每完成一个检查阶段，立即更新 Markdown 报告，追加已确认问题、证据、风险等级、整改建议和无法确认项。
7. 汇总阻断、高风险、中风险、建议优化和无法确认的风险，并刷新报告结论。
8. **强制调用报告模板**：输出报告时必须调用 `test-quality-report-template`。
9. 发现问题后先报告，等待用户确认后再逐项修复。
10. 如果需要改多个文件，按安全批量修改规则逐项处理。

### 子 skill 不可用时的处理

- 如果某个子 skill 在当前平台不存在或调用失败：**必须在报告中明确记录"子 skill {name} 调用失败/不可用"**，并说明原因。
- 不得以"子 skill 不存在"为由跳过对应检查维度；子 skill 不可用时，必须使用当前平台可用工具（如 Read/Grep/Glob）手动执行该维度的检查，并将结果记录到报告中。
- 子 skill 调用失败后，必须向用户说明："子 skill {name} 不可用，已改用 {替代方式} 进行检查"。

## 增量 Markdown 报告规则

- 质量检查必须边检查边输出 Markdown 报告，不等全部检查结束后一次性补写。
- 报告文件名默认使用 `代码质量检查报告.md`；同名文件已存在时，除非用户明确要求覆盖，否则使用 `代码质量检查报告-YYYYMMDD-HHmm.md`。
- 创建报告骨架后，每完成一个专项检查就更新一次报告，至少更新以下内容：
  - 检查进度：已完成、进行中、待检查的专项。
  - 问题统计：阻断、高风险、中风险、建议优化、无法确认。
  - 问题明细：文件路径、行号、代码内容或配置项、问题等级、原因、建议。
  - 未验证项：未执行构建、未跑接口、未做动态安全测试、依赖扫描缺失等。
- 如果当前平台没有文件写入权限，必须在对话中输出同结构 Markdown，并说明无法落盘的原因。
- 增量报告只记录已确认事实；疑似问题必须单独放入“无法确认风险”，不得当作确定缺陷。
- 报告中的密码、密钥、token、证书口令必须脱敏，仅保留定位所需的文件路径、行号和字段名。

## 强制调用组合（必须执行）

**以下调用组合为强制执行清单。只要触发本 skill，就必须按顺序调用对应子 skill，不得以任何理由跳过。**

### 第一阶段：通用检查（全部强制）

以下 7 个通用 skill **必须全部调用**，无论项目大小或代码复杂度：

1. `test-quality-common-review` — 通用代码质量（结构、命名、注释、复杂度）
2. `test-quality-common-security-review` — 代码层安全（硬编码、注入、XSS、敏感信息泄露）
3. `test-quality-common-input-validation` — 输入校验（参数校验、边界检查、空值处理）
4. `test-quality-common-database-review` — 数据库访问质量（SQL 注入、事务、连接泄漏）
5. `test-quality-common-performance-review` — 性能质量（慢查询、循环调用、资源释放）
6. `test-quality-common-testing-review` — 测试质量（覆盖率、边界测试、安全修复测试）
7. `test-quality-common-code-simplifier` — 代码简化（重复逻辑、过度抽象、可读性）

### 第二阶段：技术栈专项（按栈强制）

根据判断出的技术栈，**必须**调用对应技能：

- Java 项目：`test-quality-language-java-review`
- Vue 前端：`test-quality-language-vue-review`
- TypeScript：`test-quality-language-typescript-review`
- Python：`test-quality-language-python-review`
- Go：`test-quality-language-go-review`
- Rust：`test-quality-language-rust-review`
- Kotlin：`test-quality-language-kotlin-review`
- C++：`test-quality-language-cpp-review`

### 第三阶段：框架/模式专项（按框架强制）

涉及以下框架/架构时**必须**调用：

- Spring Boot：`test-quality-pattern-springboot`
- Vue 架构：`test-quality-pattern-vue`
- Django：`test-quality-pattern-django`
- NestJS：`test-quality-pattern-nestjs`
- Android：`test-quality-pattern-android-clean-architecture`
- C++ 标准：`test-quality-pattern-cpp-standards`
- C++ 测试：`test-quality-pattern-cpp-testing`

### 第四阶段：构建专项（按需强制）

涉及编译、构建、依赖问题时**必须**调用：

- 通用构建：`test-quality-build-common`
- Java/Maven：`test-quality-build-java`
- Kotlin/Gradle：`test-quality-build-kotlin`
- Go：`test-quality-build-go`
- Rust：`test-quality-build-rust`
- C++：`test-quality-build-cpp`

### 第五阶段：报告模板（强制收尾）

- `test-quality-report-template` — **必须**最后调用，用于格式化最终报告

## 技术栈路由（强制映射）

**以下映射关系为强制要求。只要检测到对应技术栈，就必须调用右侧列出的所有子 skill。**

| 技术栈 | 必须调用的子 skill |
|--------|-------------------|
| Java / Spring / MyBatis | `test-quality-language-java-review`、`test-quality-pattern-java`、`test-quality-pattern-springboot` |
| Vue | `test-quality-language-vue-review`、`test-quality-pattern-vue`、`test-quality-language-typescript-review` |
| TypeScript / JavaScript | `test-quality-language-typescript-review` |
| Python / Django / FastAPI | `test-quality-language-python-review`、`test-quality-pattern-django`（Django 项目） |
| Go | `test-quality-language-go-review` |
| Rust | `test-quality-language-rust-review` |
| Kotlin / Android | `test-quality-language-kotlin-review`、`test-quality-pattern-android-clean-architecture`（Android 架构） |
| C++ | `test-quality-language-cpp-review`、`test-quality-pattern-cpp-standards`、`test-quality-pattern-cpp-testing` |
| PHP / ArkTS / F# | 对应 `test-quality-language-*` |

## 统一规则

- **必须强制调用子 skill**：不得以任何理由跳过或替代子 skill 的检查工作。本 skill 只负责调度、分级和汇总，绝不替代具体专项 skill 的检查清单。
- 不默认执行任何 Git 操作。
- 不自动修复问题。
- 不跑 API 请求。
- 输出必须包含文件路径、行号、问题等级、原因、建议。
- 检查过程中必须同步维护 Markdown 报告，最终回答必须给出报告路径。
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

## 强制调用检查清单（执行时逐项勾选）

**本 skill 被触发后，必须在对话中维护以下检查清单，每调用完一个子 skill 就勾选一项。全部勾选完成后才能输出最终报告。**

### 第一阶段：通用检查（7 项）
- [ ] `test-quality-common-review` 已调用
- [ ] `test-quality-common-security-review` 已调用
- [ ] `test-quality-common-input-validation` 已调用
- [ ] `test-quality-common-database-review` 已调用（项目含数据库访问时）
- [ ] `test-quality-common-performance-review` 已调用
- [ ] `test-quality-common-testing-review` 已调用
- [ ] `test-quality-common-code-simplifier` 已调用

### 第二阶段：技术栈专项（按实际栈勾选）
- [ ] `test-quality-language-java-review` 已调用（Java 项目）
- [ ] `test-quality-language-vue-review` 已调用（Vue 项目）
- [ ] `test-quality-language-typescript-review` 已调用（TS 项目）
- [ ] `test-quality-language-python-review` 已调用（Python 项目）
- [ ] `test-quality-language-go-review` 已调用（Go 项目）
- [ ] `test-quality-language-rust-review` 已调用（Rust 项目）
- [ ] `test-quality-language-kotlin-review` 已调用（Kotlin 项目）
- [ ] `test-quality-language-cpp-review` 已调用（C++ 项目）

### 第三阶段：框架/模式专项（按实际框架勾选）
- [ ] `test-quality-pattern-springboot` 已调用（Spring Boot 项目）
- [ ] `test-quality-pattern-vue` 已调用（Vue 架构）
- [ ] `test-quality-pattern-java` 已调用（Java 项目）
- [ ] `test-quality-pattern-django` 已调用（Django 项目）
- [ ] `test-quality-pattern-nestjs` 已调用（NestJS 项目）
- [ ] `test-quality-pattern-android-clean-architecture` 已调用（Android 项目）
- [ ] `test-quality-pattern-cpp-standards` 已调用（C++ 项目）
- [ ] `test-quality-pattern-cpp-testing` 已调用（C++ 项目）

### 第四阶段：构建专项（按需勾选）
- [ ] `test-quality-build-common` 已调用
- [ ] `test-quality-build-java` 已调用（Java/Maven 项目）
- [ ] `test-quality-build-kotlin` 已调用（Kotlin/Gradle 项目）
- [ ] `test-quality-build-go` 已调用（Go 项目）
- [ ] `test-quality-build-rust` 已调用（Rust 项目）
- [ ] `test-quality-build-cpp` 已调用（C++ 项目）

### 第五阶段：报告模板（强制）
- [ ] `test-quality-report-template` 已调用

### 无法调用的子 skill（必须记录）
- [ ] 所有不可用的子 skill 已在报告中记录原因和替代检查方式

---

## 多平台兼容

- Codex：优先使用 `rg`、`rg --files`、`Get-Content -Encoding UTF8`，复杂任务使用 `update_plan`。
- Claude：优先使用 `Read`、`Grep`、`Glob`，复杂任务使用可用的 Todo 工具。
- Trae：优先使用 `SearchCodebase`、`Read`，复杂任务使用可用的任务清单工具。
- 任一平台都必须遵守：不默认执行 Git、不自动修复、不批量改文件、中文内容保持 UTF-8。
