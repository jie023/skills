---
name: "test-quality-pattern-backend"
description: "Use when reviewing general backend layering, API design, service/repository boundaries, transactions, idempotency, caching, messaging, async jobs, error handling, authorization, tests, and maintainability across backend stacks."
---
# 后端通用模式质量审查

## 定位

用于静态审查后端代码的架构模式、分层边界和工程质量风险。适用于 Java/Spring Boot、Node.js/NestJS/Express/Next API Routes、Go、Python/FastAPI/Django、PHP/Laravel、.NET 等常见后端技术栈。

只做代码、配置、测试和设计层面的质量审查；不默认调用 API、连接数据库、执行破坏性命令或触碰 Git 操作。不自动修复问题，除非用户明确要求进入修改流程。

## 内置规则覆盖

- 后端分层：API、service、repository、domain、infrastructure、job、listener 和 adapter 边界清晰。
- 业务一致性：事务、幂等、状态流转、并发控制、补偿、消息一致性和缓存失效必须检查。
- 外部边界：API、MQ、RPC、文件、第三方回调和定时任务都要做输入校验、超时、重试和错误映射。
- 数据访问：Repository 不泄漏底层实现；查询、分页、索引、批量和锁要结合数据规模判断。
- 运维质量：日志、指标、trace、告警、配置脱敏、降级和限流要覆盖核心路径。
- 生产限流：多实例、容器化或 serverless 场景必须使用 Redis、网关或平台级 limiter，避免进程内计数器导致限流失效。

## 检查流程

1. 明确审查范围：确认用户要求、目标模块、语言栈、框架、入口层、数据访问层和测试目录。
2. 读取上下文：先看路由/Controller/API handler，再看 service、repository/DAO、DTO、异常处理、中间件、配置和测试。
3. 搜索线索：用当前平台的搜索工具定位事务、权限、幂等、缓存、消息、异步、错误处理和跨层调用。
4. 还原调用链：从请求入口追到业务规则、持久化、外部依赖、异步任务和响应输出，判断边界是否清晰。
5. 按风险分级：只报告有代码证据的问题；无法确认时说明缺少的上下文和下一步验证建议。
6. 输出报告：按“阻断 / 高风险 / 中风险 / 建议优化”分组，包含路径、行号、证据、原因和建议。

## 分级标准

### 阻断

- 权限认证、租户隔离、数据归属校验缺失，可能导致越权访问或跨租户数据泄露。
- 关键写操作缺少事务、回滚边界或一致性保护，可能造成资金、库存、订单、审批等核心数据错乱。
- 创建、支付、扣减、发券、回调、消息消费等关键链路缺少幂等控制，重复请求会造成重复业务结果。
- 未校验输入直接进入 SQL、命令执行、文件路径、模板渲染或敏感外部调用，存在明确安全利用风险。
- 异步消息在提交事务前发布、消费端确认时机错误、无去重且会产生不可恢复的业务副作用。
- 错误处理向客户端返回密钥、堆栈、SQL、内部路径、个人敏感信息或生产配置。

### 高风险

- Controller/API handler 直接承载复杂业务、直接访问数据库或跨多个模块写数据，分层边界失效。
- service 与 repository/DAO 职责混乱，业务规则散落在数据访问层、脚本、定时任务或前端可控参数中。
- 事务覆盖范围错误：外部 HTTP/RPC 调用包在事务内、异步逻辑依赖未提交数据、异常被吞导致不回滚。
- 缓存 key 缺少租户、用户、权限、筛选条件或版本维度，或写后没有失效策略，可能返回错误数据。
- 消息、任务、定时器没有重试、死信、超时、告警、追踪 ID 或幂等设计，失败后不可观测或不可补偿。
- API 缺少统一参数校验、分页限制、状态码和错误结构，调用方难以稳定处理异常。
- 外部依赖缺少超时、重试边界、限流、熔断或降级，可能拖垮请求线程和连接池。

### 中风险

- DTO、VO、Entity/Model 混用，内部字段、数据库字段或敏感字段穿透到 API 响应。
- repository 查询存在 N+1、无界列表、全字段查询、缺少批量接口或缺少索引线索。
- 异常被宽泛捕获后只返回空值、布尔值或默认成功，调用方无法区分业务失败和系统失败。
- 日志缺少 requestId/traceId、用户/租户上下文或关键业务标识，线上排障困难。
- 测试只覆盖 happy path，缺少权限失败、事务回滚、幂等重复、并发冲突、缓存失效和消息重试场景。
- 模块间循环依赖、跨层 import、工具类堆积或重复业务规则增加维护成本。

### 建议优化

- 命名、包结构、接口职责、返回结构、错误码、日志字段可以更一致。
- 重复校验、重复查询、重复转换逻辑可抽到已有公共组件或局部 helper。
- 非关键链路可补充契约测试、边界测试、可观测性字段或轻量级文档。
- 可用配置、枚举、策略对象或领域方法替代散落的魔法值和条件分支。

## 怎么检查

- 分层边界：检查请求入口是否只做协议转换、鉴权、参数校验和响应封装；业务规则应在 service/use case，持久化细节应在 repository/DAO。
- API 设计：检查资源命名、HTTP 方法、状态码、错误体、分页排序、字段过滤、版本兼容、幂等键和向后兼容性。
- Service/Repository：检查 service 是否聚合业务规则和事务，repository 是否只封装数据访问；避免 repository 调 service 或 controller 跨层调 DAO。
- 事务一致性：检查事务注解/transaction block 位置、传播行为、异常回滚、嵌套事务、批量写入、事件发布和外部调用顺序。
- 幂等与并发：检查唯一约束、幂等表、请求号、业务流水号、乐观锁、分布式锁、去重窗口和重复消息处理。
- 缓存：检查 key 维度、TTL、主动失效、写后更新、缓存穿透/击穿/雪崩保护、权限数据缓存和序列化兼容。
- 消息与异步：检查生产者发布时机、消费者 ack、重试、死信、顺序性、重复消费、任务超时、任务隔离和可观测性。
- 错误处理：检查统一异常映射、业务错误码、输入校验错误、日志上下文、敏感信息脱敏和对外错误结构。
- 权限安全：检查认证、授权、角色/权限点、租户隔离、资源归属、服务间调用身份和后台任务权限上下文。
- 限流降级：检查限流维度、共享状态、配额恢复、`Retry-After`、调用方 tier、降级返回和监控告警。
- 测试覆盖：检查 service 单元测试、repository 集成测试、API 契约测试、事务回滚、并发幂等、缓存失效和消息重试测试。
- 可维护性：检查模块边界、依赖方向、重复逻辑、配置外置、命名一致性、复杂方法拆分和公共工具复用。

## 搜索线索

搜索结果必须记录文件路径、行号和关键代码片段。优先使用当前平台原生搜索能力：

- Codex：`rg -n "<关键词>" <路径>`、`rg --files <路径>`，再读取具体文件。
- Claude：优先 `Grep`、`Glob`、`Read`。
- Trae：优先 `SearchCodebase`、`Read`。

常用关键词：

- API 入口：`Controller`、`Router`、`Route`、`handler`、`NextResponse`、`APIRouter`、`urlpatterns`、`@RestController`、`app.get`、`app.post`
- 参数校验：`validate`、`schema`、`DTO`、`Request`、`BindingResult`、`Zod`、`Joi`、`class-validator`、`pydantic`
- Service/Repository：`Service`、`UseCase`、`Repository`、`DAO`、`Mapper`、`Model`、`Entity`、`findBy`、`save`、`update`
- 事务：`@Transactional`、`transaction`、`beginTransaction`、`commit`、`rollback`、`atomic`、`UnitOfWork`
- 幂等并发：`Idempotency-Key`、`idempotent`、`requestId`、`dedupe`、`unique`、`upsert`、`lock`、`version`
- 缓存：`Redis`、`Cache`、`cache`、`@Cacheable`、`setex`、`TTL`、`expire`、`invalidate`
- 消息队列：`Kafka`、`Rabbit`、`MQ`、`Queue`、`Topic`、`Consumer`、`Producer`、`SQS`、`Celery`、`Bull`
- 异步任务：`async`、`await`、`Promise`、`CompletableFuture`、`@Async`、`Executor`、`go func`、`Task.Run`
- 权限：`auth`、`permission`、`role`、`tenant`、`owner`、`principal`、`claims`、`JWT`、`RBAC`、`ACL`
- 错误日志：`throw`、`catch`、`Exception`、`ApiError`、`errorHandler`、`logger`、`traceId`、`requestId`
- 测试：`Test`、`spec`、`test`、`integration`、`contract`、`rollback`、`concurrent`

## 输出格式

按以下结构输出，未发现的问题分组可省略：

```markdown
## 阻断
- [文件路径:行号] 问题标题
  - 证据：引用关键代码或简述命中片段。
  - 原因：说明为什么会导致实际风险。
  - 建议：给出可执行修复方向。
  - 验证：建议补充的测试或人工验证方式。

## 高风险
- [文件路径:行号] 问题标题
  - 证据：
  - 原因：
  - 建议：
  - 验证：

## 中风险
- [文件路径:行号] 问题标题
  - 证据：
  - 原因：
  - 建议：
  - 验证：

## 建议优化
- [文件路径:行号] 优化标题
  - 建议：

## 未确认项
- 缺少的上下文：
- 下一步验证：
```

结论必须基于代码证据。不要只因为命名看起来可疑就下结论；无法从当前代码确认的风险放入“未确认项”。

## 多平台兼容

- 不依赖平台专属斜杠命令、后台 hooks、自动观察或自动修复流程。
- 不默认执行 Git、构建、测试、API 请求、数据库命令或外部网络请求；如用户明确要求，再按当前平台权限执行。
- 路径展示优先使用正斜杠，中文路径和中文内容保持 UTF-8。
- 在 Codex、Claude、Trae 等平台上都应遵循同一审查口径：先搜索和读取证据，再按风险分级输出。
