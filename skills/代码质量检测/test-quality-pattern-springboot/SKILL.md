---
name: "test-quality-pattern-springboot"
description: "Use when reviewing Spring Boot Controller, Service, Repository, Bean Validation, security, transactions, exceptions, configuration, Actuator, filters/interceptors, MyBatis/JPA, or tests."
---
# Spring Boot 模式质量审查

## 定位

用于 Spring Boot 项目的代码层质量审查，重点判断分层设计、Web 入口、业务事务、数据访问、安全配置、异常处理、配置隔离、可观测性和测试是否存在可落地的问题。

## 执行边界

- 只做静态代码审查；不默认运行 API 请求、启动服务、修改代码或生成修复补丁。
- 不执行任何 Git 操作；如用户另有明确要求，也必须遵守当前会话的 Git 限制。
- 不启用原始资料中的 hooks、后台观察、斜杠命令、自动修复或强制调用流程。
- 输出必须基于证据：文件路径、行号、具体代码内容或配置片段。
- 不能确认的问题标记为待验证，并说明需要读取的文件、运行的测试或环境条件。
- 中文路径、中文内容和报告必须保持 UTF-8。

## 内置规则覆盖

- Spring Boot 分层：Controller、Service、Repository/Mapper、Config、ExceptionHandler、Security、Scheduler、Listener 职责必须清晰。
- 自动配置：Bean 条件、Profile、ConfigurationProperties、AOP、事务代理和循环依赖要结合启动路径判断。
- Web/API：参数校验、统一响应、异常映射、权限注解、跨域、文件上传和幂等要纳入检查。
- 数据访问：事务传播、只读事务、MyBatis/JPA 查询、N+1、分页、锁和批量操作要结合业务场景判断。
- 可观测性：日志、traceId、健康检查、配置脱敏、Actuator 暴露面和启动失败可诊断性要检查。
- 外部依赖韧性：检查 HTTP/RPC/消息/缓存调用的超时、重试上限、指数退避、中断恢复、熔断降级和事务外调用边界。
- 生产默认值：检查 HikariCP 连接池超时、最大连接数、健康检查、限流共享存储、`Retry-After`、ProblemDetails 或统一错误结构。

## 检查流程

1. 确认项目技术栈：读取 `pom.xml`、`build.gradle`、`application*.yml`、`application*.properties`、主启动类和测试目录，识别 Spring Boot、Spring Security、MyBatis、JPA、Validation、Actuator、测试框架版本。
2. 建立代码地图：定位 Controller、Service、Repository/Mapper、DTO/VO、异常处理、SecurityConfig、配置类、Filter、Interceptor、测试类。
3. 串联调用链：按典型请求路径检查 Controller 入参、Service 事务与业务边界、Repository/Mapper SQL 或 ORM 行为、异常和响应结构。
4. 分专项审查：按本文件的检查点逐项寻找证据，不凭命名猜测问题。
5. 分级输出：按阻断、高风险、中风险、建议优化分组；同一根因合并，避免重复刷屏。
6. 给出验证建议：每个高于建议级别的问题都要说明推荐的单元测试、集成测试、配置检查或启动验证方式。

## 怎么检查

- 先看入口再看实现：从 Controller 映射和 DTO 入参开始，沿 Service、Repository/Mapper、异常处理和配置文件追踪完整证据链。
- 先确认框架版本再判断写法：Spring Boot 2/3、Spring Security 5/6、`javax`/`jakarta`、MyBatis/JPA 差异必须结合依赖文件判断。
- 先找可证明影响再分级：只有能用代码片段、配置片段或调用链证明的问题才进入阻断/高风险/中风险。
- 横向对比同类实现：同一项目内已有 Controller、异常模型、安全规则、测试风格优先作为一致性依据。
- 对跨文件问题列主证据：报告主入口文件和关键支撑文件，不把同一根因拆成大量重复问题。
- 对环境相关问题写清验证条件：例如生产 profile、反向代理、数据库方言、Actuator 访问策略、测试 profile。

## 问题等级

### 阻断

导致服务无法启动、核心接口不可用、编译失败、数据库写入错误、认证授权完全失效、生产配置明显暴露或测试无法覆盖关键链路的问题。

常见判断：

- Bean 注入循环、条件配置缺失、配置属性无法绑定，启动必然失败。
- Controller 映射冲突、请求体/参数绑定无法通过，核心接口不可访问。
- 数据写入缺少必要事务或错误事务传播，已能证明会造成脏数据或部分提交。
- 安全配置放开敏感接口、管理端点或写接口，且无其他防护证据。

### 高风险

可能造成越权、数据泄露、数据不一致、线上故障扩大、错误响应不可控或严重性能退化的问题。

常见判断：

- `permitAll`、CORS、CSRF、Actuator 暴露范围过大。
- 使用 MyBatis `${}` 拼接用户输入，或 JPA 查询存在明显 N+1 且作用于列表接口。
- `@Transactional` 因自调用、非 public 方法、异常吞掉、checked exception 未配置回滚而失效。
- 全局异常直接返回异常消息、堆栈或敏感字段。
- Filter/Interceptor 未清理上下文、未调用链路、重复读取 body 破坏后续解析。

### 中风险

会降低可维护性、可测试性、稳定性或可观测性，但短期不一定直接造成线上事故的问题。

常见判断：

- Controller 承载业务逻辑、直接暴露 Entity、分页参数无限制。
- Service 事务范围过大，把远程调用、文件 IO、消息发送放进数据库事务。
- Repository/Mapper 查询职责过重、动态 SQL 分支不可读、缺少必要索引线索。
- 配置未按 profile 隔离，测试和生产共用默认值。
- 测试只验证 happy path，缺少校验失败、权限失败、事务回滚、异常映射场景。

### 建议优化

不构成明确缺陷，但能提升一致性、可读性、测试性或长期维护质量的改进项。

常见判断：

- 可改用构造器注入、`@ConfigurationProperties`、切片测试、统一响应模型或统一错误码。
- 命名、包结构、DTO 转换、日志字段、指标标签有一致性提升空间。
- 参考项目版本，可采用更贴近当前 Spring Boot/Spring Security 版本的写法。

## 专项检查点

### Controller

- 是否保持薄入口：只做参数接收、权限/校验触发、响应转换，不承载核心业务流程。
- `@RequestBody` 是否配合 `@Valid`，`@RequestParam`、`@PathVariable`、方法级约束是否配合 `@Validated` 生效。
- HTTP 方法、状态码、路径命名、分页排序参数、幂等语义是否与接口行为一致。
- 是否直接返回 Entity、异常对象、内部枚举或敏感字段。
- 是否存在无限分页、未限制上传大小、未校验文件类型、未处理空请求体或绑定失败。
- 权限注解、租户/组织隔离参数、用户上下文是否能从入口贯穿到业务层。

### Service

- 是否使用构造器注入，避免字段注入和隐式可变依赖。
- 事务是否放在业务边界：写操作使用 `@Transactional`，查询按需使用 `readOnly = true`。
- 是否存在事务自调用、private/final 方法事务、异步方法事务、异常被捕获后不回滚。
- 是否把远程调用、消息发送、文件 IO、长耗时循环放在事务内。
- 业务校验、权限校验、幂等控制、并发控制、状态流转是否集中且可测试。
- 日志是否包含关键业务标识，避免记录密码、令牌、身份证、手机号等敏感信息。

### Repository / Mapper

- Repository/Mapper 是否只负责数据访问，不混入业务编排、HTTP 上下文或响应对象。
- MyBatis 参数是否使用 `#{}` 绑定；使用 `${}` 时必须有白名单或常量来源证据。
- 动态 SQL 是否覆盖空集合、空字符串、NULL、排序字段白名单、分页边界。
- JPA 是否存在 N+1、懒加载越界、错误 cascade、错误 orphanRemoval、Open Session in View 依赖。
- 批量写入、乐观锁、软删除、多租户、数据权限字段是否与业务一致。
- 自定义 SQL、JPQL、Specification 是否有参数绑定和索引友好性证据。

### Bean Validation

- DTO 入参是否覆盖必填、长度、格式、范围、集合元素、嵌套对象和时间边界。
- 嵌套对象、集合元素是否加 `@Valid`，方法参数校验是否加 `@Validated`。
- Spring Boot 2 与 3 的 `javax.validation` / `jakarta.validation` 包是否与项目版本匹配。
- 校验分组、创建/更新场景、默认值和空字符串处理是否符合业务语义。
- 校验失败是否由统一异常处理返回稳定错误结构，不泄露内部字段名以外的敏感信息。

### 安全配置

- `SecurityFilterChain` 或旧版安全配置是否按项目版本正确生效。
- `authorizeHttpRequests` / `antMatchers` / `requestMatchers` 的公开路径是否最小化，顺序是否正确。
- 登录、刷新令牌、登出、匿名访问、静态资源、Swagger、Actuator 的访问边界是否清楚。
- CSRF、CORS、Session 策略、密码编码器、JWT/Token 过滤器顺序是否匹配前后端形态。
- 方法级权限 `@PreAuthorize`、资源级鉴权、租户隔离是否有测试或调用链证据。
- 反向代理场景是否正确处理 forwarded headers，不能直接信任 `X-Forwarded-For`。

### 事务

- 事务注解是否位于 Spring 代理可拦截的 public 方法或外部调用入口。
- checked exception、业务异常、手动 catch 后返回错误码等场景是否会正确回滚。
- 传播行为、隔离级别、只读事务、超时设置是否有业务理由。
- 批量处理是否避免单个大事务拖垮连接池或锁表。
- 事件发布、消息发送、缓存更新是否考虑事务提交后执行。

### 异常处理

- 是否使用 `@RestControllerAdvice` / `@ControllerAdvice` 统一处理参数校验、业务异常、权限异常、数据不存在、系统异常。
- 是否保持稳定错误模型：错误码、消息、请求标识、字段错误列表、时间戳按项目约定输出。
- `Exception.class` 兜底是否记录堆栈并返回通用消息，避免把内部异常直接透出。
- 是否重复记录同一异常导致日志噪声，或吞掉异常导致调用方误判成功。
- 404、405、415、400、401、403、409、429、500 等常见状态是否映射合理。

### 配置隔离

- `application.yml`、`application-{profile}.yml`、环境变量、启动参数是否职责清楚。
- 生产地址、密钥、密码、Token、AK/SK 是否硬编码在仓库配置中。
- `@ConfigurationProperties` 是否有类型安全绑定、前缀一致性和必要校验。
- dev/test/prod 配置是否隔离，测试是否使用独立 profile、内存库或容器化依赖。
- 条件 Bean、开关配置、默认值是否能在缺失配置时给出明确失败或安全默认。

### Actuator

- `management.endpoints.web.exposure.include` 是否最小化，避免直接暴露 `env`、`beans`、`heapdump`、`threaddump`、`logfile` 等敏感端点。
- 管理端点是否需要认证授权、内网限制、独立端口或独立 base path。
- `show-details`、health groups、readiness/liveness 是否符合部署平台要求。
- 指标和追踪是否避免高基数字段，例如用户 ID、手机号、完整 URL 参数。

### Filter / Interceptor

- Filter 是否优先继承 `OncePerRequestFilter`，并处理好 order、排除路径、异步和 error dispatch。
- 是否始终调用 `filterChain.doFilter`，异常分支是否正确返回状态并停止后续链路。
- MDC、ThreadLocal、用户上下文是否在 finally 中清理。
- 包装 request/response body 是否不会破坏后续读取，是否限制日志体积和敏感字段。
- Interceptor 是否避免承载复杂业务，路径匹配是否覆盖 Swagger、静态资源、Actuator 等例外。

### MyBatis / JPA

- MyBatis XML、注解 SQL、Provider SQL 是否参数绑定安全、字段名白名单明确、结果映射完整。
- Mapper 方法返回值是否能区分不存在、空集合、唯一性冲突和多行异常。
- JPA Entity 是否避免直接作为 API DTO，equals/hashCode、级联、懒加载、枚举映射是否安全。
- 列表查询是否有分页、排序白名单、必要 fetch join / EntityGraph / batch size。
- 数据库时间、逻辑删除、租户字段、审计字段是否由统一机制维护。

### 测试

- Controller 使用 MockMvc/WebTestClient 或切片测试覆盖校验失败、权限失败、异常映射、状态码和响应结构。
- Service 测试覆盖事务回滚、状态流转、幂等、并发边界、外部依赖失败。
- Repository/Mapper 测试覆盖真实 SQL、动态 SQL 分支、分页、排序、空集合和约束冲突。
- 安全配置测试覆盖公开路径、受保护路径、角色/权限差异、CSRF/CORS 策略。
- 配置和 Actuator 测试覆盖 profile、属性绑定失败、管理端点暴露范围。
- 集成测试使用独立 profile 和可重复数据；数据库行为优先使用 Testcontainers 或项目既有测试设施。

## 搜索线索

Codex 优先使用 `rg --line-number --no-heading`，搜索结果必须保留文件路径、行号和命中的代码内容。其他平台使用等价工具。

```bash
rg --line-number --no-heading '@RestController|@Controller|@RequestMapping|@GetMapping|@PostMapping|@PutMapping|@DeleteMapping' .
rg --line-number --no-heading '@Service|@Transactional|@Async|@Scheduled' .
rg --line-number --no-heading '@Repository|interface .*Repository|extends JpaRepository|@Mapper|<select|<insert|<update|<delete' .
rg --line-number --no-heading '@Valid|@Validated|@NotNull|@NotBlank|@Size|@Pattern|@Min|@Max|@Email' .
rg --line-number --no-heading 'SecurityFilterChain|authorizeHttpRequests|permitAll|requestMatchers|csrf|cors|PasswordEncoder|OncePerRequestFilter' .
rg --line-number --no-heading '@RestControllerAdvice|@ControllerAdvice|@ExceptionHandler|MethodArgumentNotValidException|BindException|AccessDeniedException' .
rg --line-number --no-heading 'management\.endpoints|management\.endpoint|show-details|server\.forward-headers-strategy|spring\.profiles' .
rg --line-number --no-heading '\$\{|X-Forwarded-For|ThreadLocal|MDC|OpenEntityManagerInView|spring\.jpa\.open-in-view' .
rg --line-number --no-heading '@SpringBootTest|@WebMvcTest|@DataJpaTest|MockMvc|WebTestClient|Testcontainers|@Sql|@ActiveProfiles' .
```

没有 `rg` 时使用平台内置搜索：Claude 用 `Grep`/`Glob`/`Read`，Trae 用 `SearchCodebase`/`Read`，但输出仍必须包含路径、行号和代码内容。

## 输出格式

```markdown
## Spring Boot 质量审查报告

### 阻断
- [阻断] `src/main/java/.../Xxx.java:42`
  - 证据：粘贴或概括命中的关键代码。
  - 问题：说明违反的规则和触发条件。
  - 影响：说明会导致的启动失败、越权、数据错误或核心功能不可用。
  - 建议：给出可执行修复方向，不直接改代码。
  - 验证：建议补充或运行的测试/配置检查。

### 高风险
- 未发现。

### 中风险
- 未发现。

### 建议优化
- 未发现。

### 待验证
- 说明因缺少配置、版本、运行环境或调用链证据而无法确认的点。
```

规则：

- 没有发现的问题必须写“未发现”，不要省略分组。
- 每条问题只报一个主因；同一根因影响多个文件时列出主要入口和代表性证据。
- 不确定时不得升级等级；写入“待验证”并说明下一步怎么确认。
- 修复建议必须结合项目现有框架和版本，不推荐无关重构。

## 多平台兼容

- Codex：优先 `rg` 搜索、读取具体文件、用当前会话可用工具输出报告；不触碰 Git。
- Claude：优先 `Grep`、`Glob`、`Read`；如需要任务清单，使用可用 Todo 工具；不触碰 Git。
- Trae：优先 `SearchCodebase`、`Read`；如需要任务清单，使用平台任务工具；不触碰 Git。
- 任一平台都必须尊重用户指定的写入边界、编码要求、确认流程和报告模板。
