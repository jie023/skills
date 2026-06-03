---
name: "test-quality-pattern-java"
description: "Use when reviewing Java CRUD, layered architecture, Controller, Service, DAO/Mapper, DTO, VO, DO, validation, transactions, exceptions, comments, P3C, MapStruct, pagination, batch operations, or utility reuse."
---
# Java 模式质量审查

## 定位

用于审查 Java / Spring Boot / MyBatis-Plus CRUD 分层代码的结构、边界、事务、校验、异常、注释和 P3C 风险。只做代码层质量审查，不跑 API 请求，不默认修改代码。

## 检查重点

- **Controller**：接口路径、HTTP 方法、参数绑定、`@Validated` / `@Valid`、统一响应、接口注释或 OpenAPI 注解、是否只做编排不写业务逻辑。
- **Service**：业务规则、事务边界、幂等和并发、异常语义、批量处理、是否避免直接透出持久化对象。
- **DAO/Mapper**：SQL 条件、分页、批量、N+1、字段映射、`@Param`、XML 与接口一致性、是否绕过已有 BaseMapper / LambdaQuery 工具。
- **DTO/VO/DO**：入参和出参职责分离、校验注解、敏感字段、命名、时间/金额类型、DO 与表字段映射。
- **对象转换**：MapStruct 或项目既有转换器复用，避免散落手写复制；更新场景关注 `@MappingTarget` 和空值覆盖。
- **事务**：写操作使用 `@Transactional(rollbackFor = Exception.class)`，只读查询可用 `readOnly = true`，避免同类自调用导致事务失效。
- **P3C 与注释**：类、public 方法、复杂逻辑应有中文注释，类文件头部包含 `@author 李杰`；集合、并发、异常、日志、命名遵循项目规范和 P3C。
- **工具类复用**：优先复用项目已有 `Result`、分页对象、异常类、断言、字符串/集合/日期/金额工具，不重复造轮子。

## 执行规则

- 不默认执行任何 Git 操作。
- 不自动修复问题，只输出问题和修复建议；如用户要求修改，再按普通代码修改流程处理。
- 输出必须包含文件路径、行号、问题等级、原因、建议；无法确认的问题要说明验证方式。
- 报告输出遵循 `test-quality-report-template`，但不要引入该模板未要求的冗余章节。
- 多文件修改必须按安全批量修改规则执行。
- 中文路径、中文注释和中文报告必须保持 UTF-8。

## 检查流程

1. 明确审查范围：确认用户指定的模块、包、变更文件或关键业务流程。
2. 搜索入口：先找 Controller、Service、Mapper、DTO/VO/DO，再顺着调用链读取实现和 XML/注解 SQL。
3. 识别分层边界：判断接口层、业务层、数据访问层、数据对象之间是否职责清晰。
4. 检查核心风险：事务、校验、异常、SQL、分页批量、对象转换、P3C、注释、工具类复用。
5. 对照项目约定：优先使用项目已有基类、返回体、异常、分页、转换、工具类和命名风格。
6. 分级输出：只报告可定位、可复现或有明确验证路径的问题，避免泛泛而谈。

## 问题分级

- **阻断**：会导致编译失败、启动失败、接口不可用、事务完全失效、数据写错/删错、权限或敏感数据严重泄露的问题。
- **高风险**：可能造成线上数据不一致、重复提交、并发覆盖、SQL 注入、分页错误、异常吞掉、回滚失败、敏感字段返回的问题。
- **中风险**：分层职责混乱、DTO/VO/DO 混用、校验缺失、Mapper 条件不严谨、批量无边界、对象转换散落、注释不完整、P3C 明确违规。
- **建议优化**：命名、局部重复、工具类复用、查询可读性、轻量性能优化、可测试性和维护性改进。

## 怎么检查

- **Controller**：检查 `@RestController`、`@RequestMapping`、`@GetMapping` / `@PostMapping` 等路径是否清晰；入参是否用 `@RequestBody @Valid`、`@PathVariable`、`@RequestParam`；是否直接写 SQL、事务或复杂业务逻辑。
- **Service**：检查 public 方法是否有中文注释和清晰返回值；写操作是否有事务；查询是否只读；异常是否使用业务异常；是否复用 DAO/Mapper 和转换器。
- **DAO/Mapper**：检查注解 SQL / XML SQL 是否字段齐全、条件正确、参数绑定安全；分页是否由数据库完成；批量操作是否有空集合保护和数量边界。
- **DTO/VO/DO**：检查 DO 是否只表达持久化结构；DTO 是否覆盖必要入参校验；VO 是否避免返回密码、密钥、内部状态等敏感字段。
- **事务与异常**：检查 `rollbackFor`、事务方法可见性、同类自调用、异步/多线程事务边界；异常是否被吞掉或只打印日志后继续执行。
- **P3C 与注释**：检查命名、常量、集合判空、日志占位符、魔法值、浮点金额、`equals`、空指针风险；类头和 public API 注释是否满足项目要求。
- **Mapper 与转换器**：检查 MapStruct 或项目转换工具是否统一；更新对象时是否误把 null 覆盖到已有字段；列表转换是否避免重复代码。
- **工具类复用**：搜索项目已有 `Result`、`PageResult`、`BusinessException`、`Assert`、`CollectionUtils`、`StringUtils`、日期金额工具，再判断新增代码是否重复实现。
- **Service 单测模式**：检查 `MockitoExtension`、`@Mock`、`@InjectMocks`、mapper/converter mock 是否隔离外部依赖；是否覆盖存在/不存在、异常、事务边界、转换结果和关键断言，避免只验证调用次数不验证业务结果。

## 搜索线索

- 文件名：`*Controller.java`、`*Service.java`、`*ServiceImpl.java`、`*Mapper.java`、`*Dao.java`、`*DTO.java`、`*VO.java`、`*DO.java`、`*.xml`。
- 注解：`@RestController`、`@Controller`、`@Service`、`@Transactional`、`@Mapper`、`@TableName`、`@TableId`、`@Valid`、`@Validated`、`@NotNull`、`@NotBlank`、`@Operation`。
- SQL 与分页：`selectPage`、`page(`、`Page<`、`LIMIT`、`foreach`、`IN (`、`last(`、`${`、`@Select`、`@Update`、`@Delete`。
- 风险词：`TODO`、`printStackTrace`、`catch (Exception`、`return null`、`new RuntimeException`、`BeanUtils.copyProperties`、`password`、`secret`、`token`。
- 工具类：`Result`、`PageResult`、`BusinessException`、`Assert`、`CollectionUtils`、`StringUtils`、`BeanUtil`、`MapStruct`、`Converter`。
- 测试线索：`MockitoExtension`、`@Mock`、`@InjectMocks`、`when(`、`verify(`、`assertThat`、`assertEquals`、`assertThrows`、`*ServiceTest.java`。

## 输出要求

按以下结构输出，问题按阻断、高风险、中风险、建议优化分组；同组内优先列影响范围大的问题。

```markdown
## 阻断
- [文件路径:行号] 问题标题
  - 原因：说明代码为什么有问题。
  - 影响：说明可能造成的结果。
  - 建议：给出可执行修复方向。

## 高风险
- [文件路径:行号] ...

## 中风险
- [文件路径:行号] ...

## 建议优化
- [文件路径:行号] ...

## 验证建议
- 需要补充的单元测试、集成测试、SQL 验证或人工确认项。
```

如果没有发现某级别问题，明确写“未发现”。不要因为没有完整上下文而虚构问题；用“需要确认”说明缺少的配置、表结构或调用链。

## 多平台兼容

- **Codex**：搜索优先 `rg` / `rg --files`；读取中文文件使用 `Get-Content -Encoding UTF8`；不要执行 Git 命令，除非用户明确要求。
- **Claude**：搜索优先 `Grep` / `Glob` / `Read`；中文文件按 UTF-8 读取；不要使用平台外的自动修复或后台观察流程。
- **Trae**：搜索优先 `SearchCodebase` / `Read`；输出仍使用同一分级和字段；中文内容保持 UTF-8。

## 内置规则覆盖

- CRUD 分层：Entity/DO、DTO、VO、Mapper、Service、Controller 职责清晰，禁止 Controller 直接访问 Mapper。
- 对象转换：请求 DTO、领域对象、持久化对象、响应 VO 不混用，字段命名和空值策略要一致。
- Service 模式：事务、幂等、状态机、权限校验、数据聚合和异常转换应集中在业务层。
- Mapper 模式：SQL 参数化、分页、动态条件、批量操作和 XML/接口参数一致性必须检查。
- 注释规范：类、public 方法、复杂逻辑遵守中文 JavaDoc 和 `@author 李杰` 要求。
