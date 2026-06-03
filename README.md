# Skills

个人 Agent Skills 仓库，用于沉淀李杰的全局协作规则、Java 后端开发规范、产品/数据库/接口文档模板、P3C 规则、安全规则以及多语言代码质量审查能力。

本仓库以中文规则和 UTF-8 文件为主。每个技能目录通常包含：

- `SKILL.md`：技能主体说明，定义触发场景、执行流程、检查清单或模板。
- `agents/openai.yaml`：OpenAI Agent 入口配置，提供展示名称、简短说明和默认提示。

## 目录结构

```text
.
├── README.md
├── 全局个人规则.md
└── skills/
    ├── 产品设计/
    ├── 接口文档/
    ├── 数据库设计/
    ├── java开发/
    ├── java接口检测/
    ├── 阿里云p3c/
    ├── 安全规则/
    └── 代码质量检测/
```

## 全局规则

[全局个人规则.md](./全局个人规则.md) 是本仓库的基础协作约束，核心规则包括：

- 默认使用中文沟通，代码、命令、API 名称和专有名词可按原文保留。
- 文件编码强制使用 UTF-8，涉及中文路径或中文文件时启用 `windows-utf8-safe-editing`。
- Java 代码遵循个人 Java 规范、中文注释、P3C 风格和 `@author 李杰`。
- 超过 3 个步骤的任务需要使用任务清单工具进行规划和状态更新。
- 无明确要求时不要执行 Git 操作；用户明确要求 Git 时使用 `git-commit-helper`。
- 删除数据、修改生产配置、强制推送、安全敏感操作必须先确认。

## 技能索引

### 产品设计

| 技能 | 用途 |
| --- | --- |
| `jie023-product-designer` | 创建 PRD、产品功能设计、用户流程、线框与需求规格。 |
| `requirement-doc-template` | 生成结构化需求文档模板，覆盖概述、需求、流程、数据、接口和验收标准。 |

### 接口文档

| 技能 | 用途 |
| --- | --- |
| `jie023-api-doc-writer` | 根据 Java Controller 或 REST 接口代码生成前端可用的 API 文档。 |

### 数据库设计

| 技能 | 用途 |
| --- | --- |
| `jie023-database-designer` | 设计 MySQL 表结构、SQL DDL、索引和规范化数据库模型。 |
| `database-design-template` | 输出数据库设计文档模板，描述表结构、字段、关系和建表 SQL。 |

### Java 开发

| 技能 | 用途 |
| --- | --- |
| `jie023-java-backend-architect` | 生成 Java 后端 CRUD 代码，覆盖 DO、VO、DAO、Service、Controller。 |
| `java-personal-standards` | 个人 Java 编码规范，强调中文注释、JavaDoc、作者、P3C 和项目工具复用。 |
| `java-common-standards` | Java 通用编码规范，覆盖 Hutool、Lombok 和共享约定。 |
| `java-do` | 根据 SQL 表结构生成或审查 Java DO 对象。 |
| `java-vo` | 生成或审查 Java VO 对象、字段分类和 DO 到 VO 转换。 |
| `java-dao` | 生成或审查 DAO、MyBatis XML、Mapper 方法和数据访问模式。 |
| `java-service` | 生成或审查 Service 接口、ServiceImpl、事务和业务逻辑层。 |
| `java-controller` | 生成或审查 Controller、REST 映射、参数校验和响应处理。 |
| `java-validation-annotations` | 为 Java 字段或 VO 添加 JSR-380 校验注解。 |

### Java 接口检测与测试

| 技能 | 用途 |
| --- | --- |
| `jie023-test-api-engineer` | 统筹 API 优先的测试方案，覆盖功能、安全、性能、数据质量和权限。 |
| `test-api-basic-function` | 验证 API 基础功能，覆盖正向、反向和边界用例。 |
| `test-api-security` | 测试 API 权限、认证、授权、注入和敏感数据暴露。 |
| `test-api-performance` | 测试 API 响应时间、并发、压力和性能回归。 |
| `test-api-data-quality` | 测试 API 数据一致性、完整性、迁移和修复结果。 |
| `test-function` | 测试应用功能或业务逻辑，覆盖 UI 流程和功能行为。 |
| `test-ui` | 使用浏览器或 Playwright 检查 UI 布局、渲染、响应式和交互。 |

### 阿里云 P3C

| 技能 | 用途 |
| --- | --- |
| `test-p3c-code-quality` | 按 P3C 规则综合检查 Java 代码质量。 |
| `p3c-coding-style` | 应用 P3C 命名、格式、常量和中文注释规范。 |
| `p3c-advanced-coding` | 审查集合、并发、多线程、控制流和线程安全问题。 |
| `p3c-oop-standards` | 审查 OOP 规则、equals/hashCode、包装类型、序列化和访问控制。 |
| `p3c-exception-logging` | 处理异常、NPE、try-catch-finally 和 SLF4J 日志规范。 |
| `p3c-security-rules` | 审查认证、授权、脱敏、SQL 注入、CSRF 和输入校验。 |
| `p3c-mysql-database` | 应用 P3C MySQL 建表、索引、SQL 优化和 ORM 映射规则。 |
| `p3c-unit-testing` | 编写或审查 Java 单元测试、覆盖率和 AIR 原则。 |

### 安全规则

| 技能 | 用途 |
| --- | --- |
| `windows-utf8-safe-editing` | 处理 Windows 中文路径、中文文件、UTF-8、乱码和编码敏感写入。 |
| `safe-bulk-editing` | 执行多文件修改、批量替换、字段迁移或格式化前的安全编辑流程。 |
| `secure-code-checklist` | 检查用户输入、SQL、命令执行、路径、模板、权限、敏感数据和日志风险。 |
| `git-commit-helper` | 仅在用户明确要求 Git 操作时，辅助检查变更、暂存、提交和提交信息。 |

### 代码质量检测与审查

| 技能 | 用途 |
| --- | --- |
| `jie023-test-quality-engineer` | 统筹代码级质量测试，不直接运行 API 请求。 |
| `test-quality-report-template` | 生成代码质量、安全、数据库、构建或测试质量报告。 |
| `test-quality-common-review` | 审查通用代码质量、可维护性、复杂度、重复、异常和日志。 |
| `test-quality-common-code-simplifier` | 审查过度复杂、重复、深层分支、过度抽象或难读代码。 |
| `test-quality-common-security-review` | 审查密钥、注入、XSS、路径穿越、权限、敏感日志和错误泄露。 |
| `test-quality-common-input-validation` | 审查请求参数、DTO 校验、边界、上传、ID 和多层校验覆盖。 |
| `test-quality-common-database-review` | 审查 SQL、Mapper、Repository、迁移、事务、分页、索引和租户过滤。 |
| `test-quality-common-performance-review` | 审查循环、重复计算、缓存、分页、对象分配和渲染成本。 |
| `test-quality-common-testing-review` | 审查单元、集成、回归、边界、权限和安全测试覆盖。 |

#### 构建质量审查

| 技能 | 用途 |
| --- | --- |
| `test-quality-build-review` | 诊断构建、编译、打包、类型、依赖或工具链失败。 |
| `test-quality-build-java-review` | 诊断 Java、Maven、Gradle、Spring Boot 和注解处理器构建问题。 |
| `test-quality-build-node-review` | 审查 Node.js、npm、pnpm、yarn、TypeScript、Vite、Webpack 和 ESLint 构建问题。 |
| `test-quality-build-go-review` | 诊断 Go build、go vet、模块、依赖、导入和编译器问题。 |
| `test-quality-build-cpp-review` | 诊断 C++ CMake、编译、链接、模板、include 和 ABI 问题。 |
| `test-quality-build-kotlin-review` | 诊断 Kotlin、Gradle、Ktor、Android 和 Kotlin Multiplatform 构建风险。 |
| `test-quality-build-rust-review` | 诊断 Rust cargo、借用检查、Cargo.toml、features、edition 和 MSRV 问题。 |

#### 语言专项审查

| 技能 | 用途 |
| --- | --- |
| `test-quality-language-java-review` | 审查 Java/Spring、P3C、事务、MyBatis、Lombok、NPE、异常和 JavaDoc。 |
| `test-quality-language-typescript-review` | 审查 TypeScript/JavaScript、类型安全、Zod、异步错误、XSS 和依赖。 |
| `test-quality-language-vue-review` | 审查 Vue 3、Composition API、props、表单、Pinia、XSS 和渲染性能。 |
| `test-quality-language-python-review` | 审查 Python、FastAPI、Django、Pydantic、类型提示、pytest 和安全质量。 |
| `test-quality-language-go-review` | 审查 Go 错误处理、context、goroutine/channel、Gin/GORM/SQL、竞态和测试。 |
| `test-quality-language-cpp-review` | 审查 C++ RAII、内存安全、生命周期、move/copy、并发、未定义行为和测试。 |
| `test-quality-language-kotlin-review` | 审查 Kotlin、Android、Ktor、协程、Flow、空安全和测试。 |
| `test-quality-language-rust-review` | 审查 Rust 所有权、借用、tokio、Axum、错误处理、并发和惯用安全。 |
| `test-quality-language-php-review` | 审查 PHP/Laravel 代码质量、安全、测试、框架模式和命名空间。 |
| `test-quality-language-fsharp-review` | 审查 F# 函数式风格、类型安全、模式匹配、计算表达式和性能。 |
| `test-quality-language-arkts-review` | 审查 ArkTS/HarmonyOS、V2 状态管理、导航、权限、存储和构建校验。 |

#### 架构与模式审查

| 技能 | 用途 |
| --- | --- |
| `test-quality-pattern-backend` | 审查后端分层、API、Service/Repository、事务、幂等、缓存、消息和异步任务。 |
| `test-quality-pattern-api-design` | 审查 REST API 合同、资源命名、HTTP 语义、分页、错误、鉴权和兼容性。 |
| `test-quality-pattern-springboot` | 审查 Spring Boot Controller、Service、Repository、校验、安全、事务和配置。 |
| `test-quality-pattern-java` | 审查 Java CRUD 分层、DTO/VO/DO、事务、异常、注释、P3C、分页和批处理。 |
| `test-quality-pattern-vue` | 审查 Vue 组件架构、composable、Pinia、表单、路由和渲染。 |
| `test-quality-pattern-nestjs` | 审查 NestJS 模块、Controller、Provider、DTO、Guard、Interceptor 和配置。 |
| `test-quality-pattern-django` | 审查 Django/DRF 设置、中间件、模型、ORM、迁移、序列化器、权限和事务。 |
| `test-quality-pattern-flutter` | 审查 Dart/Flutter 空安全、Widget 架构、状态管理、异步错误、路由和性能。 |
| `test-quality-pattern-compose-multiplatform` | 审查 Compose Multiplatform 或 Jetpack Compose 状态、导航、主题和性能。 |
| `test-quality-pattern-android-clean-architecture` | 审查 Android 整洁架构分层、用例、仓储、ViewModel 和依赖边界。 |
| `test-quality-pattern-database-migrations` | 审查数据库迁移、回滚安全、批处理、在线 DDL 和 schema 演进风险。 |
| `test-quality-pattern-cpp-standards` | 审查 C++ Core Guidelines、接口、资源管理、类、表达式、并发和模板。 |
| `test-quality-pattern-cpp-testing` | 审查 C++ 测试、Google Test、fixture、断言、覆盖率和测试设计。 |

## 使用方式

在对话中可以直接点名技能，例如：

```text
使用 $jie023-java-backend-architect 生成用户模块 CRUD
使用 $test-quality-language-vue-review 审查这个 Vue 组件
使用 $windows-utf8-safe-editing 修改中文文档
```

也可以通过任务内容触发对应技能，例如“根据 Controller 生成接口文档”会匹配 `jie023-api-doc-writer`，“检查 SQL 注入风险”会匹配代码质量检测或安全规则相关技能。

## 新增技能约定

新增技能建议使用以下结构：

```text
skills/<分类>/<skill-name>/
├── SKILL.md
└── agents/
    └── openai.yaml
```

`SKILL.md` 推荐包含：

- YAML front matter：`name` 与 `description`。
- 使用场景：说明什么时候必须使用该技能。
- 执行流程：列出可复用步骤、检查顺序或输出要求。
- 输出格式：约定报告、代码、文档或清单格式。
- 禁止事项：明确高风险或不允许的操作。

`agents/openai.yaml` 推荐包含：

- `interface.display_name`：展示名称。
- `interface.short_description`：一句话能力说明。
- `interface.default_prompt`：默认调用提示。
- `policy.allow_implicit_invocation`：是否允许隐式触发。

## 维护原则

- 修改中文文档、中文路径或编码敏感内容时，必须显式关注 UTF-8。
- 多文件或批量改动前先明确影响范围，避免误改和编码损坏。
- 技能描述应短而明确，便于 Agent 通过任务内容自动匹配。
- 技能内容应优先沉淀可执行流程、检查清单和输出模板，减少泛泛说明。
- Git 操作只在用户明确要求时执行，并遵循仓库提交规范。
