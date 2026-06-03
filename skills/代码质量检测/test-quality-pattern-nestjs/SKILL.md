---
name: "test-quality-pattern-nestjs"
description: "Use when reviewing NestJS modules, controllers, providers, DTO validation, guards, interceptors, config, or backend TypeScript patterns."
---
# NestJS 模式质量审查

## 定位

NestJS 模式质量审查，重点覆盖模块边界、Controller/Provider 分层、DTO 校验、Guards/Pipes/Interceptors/Filters、配置校验、认证授权、持久化事务、测试和生产默认项。

## 适用范围

- 用户要求审查 NestJS API、模块结构、Controller、Provider、DTO、Guard、Interceptor、Exception Filter、Config、Prisma/TypeORM、测试时使用。
- 涉及 public API、登录鉴权、角色权限、环境变量、事务、后台任务、事件消费、异常响应和 Swagger/OpenAPI 时使用。
- 默认只做静态审查，不启动服务、不跑 API 请求、不运行测试或构建；需要运行时先说明目的并等待确认。

## 检查流程

1. 明确范围：读取用户指定模块、Controller、Service、DTO、module、config、test 和 persistence 文件。
2. 建立模块图：确认 feature module、shared/common module、provider export/import、循环依赖和依赖注入边界。
3. 跟请求链路：从 Controller 到 DTO/Pipe/Guard/Interceptor/Service/Repository/Filter/Response DTO。
4. 按分级检查项判断：先安全、校验、鉴权、事务和错误响应，再查配置、测试和可维护性。
5. 每个问题必须包含文件路径、行号、证据、风险、建议和验证方式；无法确认时说明缺少的信息。

## 分级检查项

### 阻断

- **公开接口缺失认证授权**：敏感 Controller/route 没有 Guard、角色权限或资源级鉴权。
  - 怎么检查：搜索 `@Controller`、`@UseGuards`、`@Roles`、`JwtAuthGuard`，沿 service 检查 owner/tenant/resource 权限。
- **DTO 校验未生效**：public API 没有全局 `ValidationPipe`，或缺少 `whitelist/forbidNonWhitelisted/transform`，导致外部输入直达业务。
  - 怎么检查：读取 `main.ts`、DTO、Controller 入参和测试，确认 class-validator/class-transformer 是否生效。
- **敏感信息泄露**：直接返回 ORM entity、password hash、token、内部异常、堆栈或环境变量。
  - 怎么检查：查看 response DTO/serializer、exception filter、logger、entity 字段和返回对象。
- **事务边界错误**：多步写入由 Controller 编排，或 Prisma/TypeORM 事务缺失导致部分提交。
  - 怎么检查：读取 service 写操作、transaction block、repository/provider 调用链和异常处理。
- **环境配置无启动期校验**：生产密钥、数据库连接、JWT secret、第三方配置缺失时服务仍启动。
  - 怎么检查：读取 `ConfigModule.forRoot`、validation schema、typed config service 和默认值。

### 高风险

- **Controller 过厚**：Controller 中写业务、事务、数据库、缓存或外部 SDK 调用。
  - 怎么检查：查看 Controller 方法长度、注入依赖类型和是否直接访问 ORM/client。
- **Module export/import 过宽或循环依赖**：模块暴露过多 provider，使用 forwardRef 掩盖设计问题。
  - 怎么检查：读取 `@Module` imports/providers/exports，搜索 `forwardRef` 和跨模块直接依赖。
- **Guard/Pipe/Interceptor/Filter 职责混乱**：鉴权、校验、序列化、日志、错误映射分散且不可预测。
  - 怎么检查：读取 common 下实现，确认职责是否单一、顺序是否稳定、异常是否被正确处理。
- **错误响应不一致**：异常 filter 不统一，业务错误和系统错误状态码/结构混乱。
  - 怎么检查：搜索 `throw new`、`HttpException`、`ExceptionFilter`、错误码和响应体。
- **配置散落或硬编码**：直接读取 `process.env`，在业务代码中按环境分支，缺少 typed config。
  - 怎么检查：搜索 `process.env`、`ConfigService.get`、默认值和配置工厂。
- **测试缺失关键链路**：Guard、ValidationPipe、ExceptionFilter、Controller e2e、Service 事务缺少测试。
  - 怎么检查：查 `*.spec.ts`、e2e、TestingModule，确认是否复用生产全局 pipe/filter。

### 中风险

- **Provider 依赖方向不清**：Service 直接依赖具体 SDK/ORM，缺少可测试的窄接口或 adapter。
  - 怎么检查：查看 constructor 注入和 provider token，判断是否易 mock。
- **DTO/Entity/Response 混用**：Create/Update DTO、Entity、Response DTO 不分离。
  - 怎么检查：查看 Controller 入参和返回类型，确认是否直接暴露 entity。
- **后台任务和事件消费放在 HTTP 模块里**：队列、scheduler、consumer 与 Controller 混杂。
  - 怎么检查：搜索 `@Processor`、`@Cron`、event handler、consumer 和 module 归属。
- **日志和可观测性不足**：缺少 correlation id、结构化日志、请求上下文或 health check。
  - 怎么检查：查看 logger/interceptor/middleware、health module 和 provider 初始化。
- **依赖和版本兼容风险**：Nest、class-validator、class-transformer、TypeORM/Prisma、Swagger 版本不匹配。
  - 怎么检查：读取 `package.json`、lockfile 和相关配置。

### 建议优化

- **按 feature module 组织代码**：DTO、Controller、Service、Repository 贴近模块，cross-cutting 放 common。
- **使用 typed config**：把环境变量统一校验和转换为类型安全配置。
- **统一响应和错误文档**：Swagger/OpenAPI 补安全方案、错误 schema 和分页 schema。
- **测试结构改进**：Provider 单测 mock 依赖，Controller/e2e 测真实 pipe/guard/filter。

## 常用搜索线索

按当前平台工具执行搜索，结果必须包含文件路径、行号和具体命中内容。

```text
@Module|imports:|providers:|exports:|forwardRef
@Controller|@Get|@Post|@Put|@Patch|@Delete|@Body|@Param|@Query
ValidationPipe|whitelist|forbidNonWhitelisted|transform
class .*Dto|@IsString|@IsNotEmpty|@IsOptional|@ValidateNested|@Type\\(
@UseGuards|CanActivate|JwtAuthGuard|RolesGuard|@Roles|SetMetadata
Interceptor|ExceptionFilter|@Catch|HttpException|throw new
ConfigModule|ConfigService|process\\.env|validateEnv|Joi|zod
PrismaService|TypeOrmModule|Repository|transaction|QueryRunner|\\$transaction
TestingModule|createTestingModule|supertest|e2e|spec\\.ts
```

## 判断标准

- 阻断：可能导致未授权访问、输入绕过、敏感泄露、部分提交或生产配置缺失仍启动。
- 高风险：分层、模块、错误响应、配置或测试缺陷可能造成线上故障和维护困难。
- 中风险：DTO、依赖方向、后台任务、日志、版本兼容存在明显隐患。
- 建议优化：模块组织、typed config、文档和测试结构可提升质量。

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
  - 位置：path/to/file.ts:line
  - 证据：命中的关键代码或调用链
  - 原因：为什么违反 NestJS 质量要求
  - 建议：如何修改或下一步验证
```

没有发现对应等级问题时写“未发现”。无法确认的问题要说明原因和下一步验证建议。

## 内置规则覆盖

- 模块边界：检查 module imports/exports、provider 作用域、循环依赖、跨模块直接访问和共享模块膨胀。
- DTO 与校验：检查 `ValidationPipe`、`class-validator`、`class-transformer`、白名单、嵌套对象和枚举/分页参数校验。
- Guard 与权限：检查认证 Guard、角色/资源权限、对象级权限、公开接口标记和默认拒绝策略。
- Interceptor / Filter：检查异常归一、错误码、日志上下文、响应包装、超时、重试和敏感信息泄露。
- 配置管理：检查 `ConfigModule`、环境变量 schema、默认值、密钥读取、测试配置隔离和启动失败策略。
- 数据访问：检查 Prisma/TypeORM 事务、连接生命周期、N+1、分页、软删除、唯一约束和并发更新。
- 生产默认值：检查异步 provider 初始化失败处理、数据库/缓存 health check、公共接口显式限流、审计日志和启动期依赖校验。
- 测试覆盖：检查 controller、service、guard、pipe、interceptor、repository mock 和 e2e 权限路径。

## 多平台兼容

- Codex：优先使用 `rg`、`rg --files`、`Get-Content -Encoding UTF8`，复杂任务使用 `update_plan`。
- Claude：优先使用 `Read`、`Grep`、`Glob`，复杂任务使用可用的 Todo 工具。
- Trae：优先使用 `SearchCodebase`、`Read`，复杂任务使用可用的任务清单工具。
- 任一平台都必须遵守：不默认执行 Git、不自动修复、不批量改文件、中文内容保持 UTF-8。
