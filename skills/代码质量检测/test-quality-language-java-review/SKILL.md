---
name: "test-quality-language-java-review"
description: "Use when reviewing Java or Spring code quality, P3C compliance, transactions, MyBatis, Lombok, NPE, exceptions, JavaDoc, or Chinese comments."
---
# Java 语言专项质量审查

## 定位

面向 Java / Spring / MyBatis 项目的代码层质量审查。重点检查 P3C 规范、分层边界、参数校验、事务一致性、SQL 风险、异常日志、NPE、Lombok、JavaDoc、中文注释和项目既有工具类复用。

本 skill 只做静态代码审查和审查报告输出，不默认修复代码、不跑 API 请求、不执行 Git 操作。若用户要求继续修复，再按用户确认范围逐项处理。

## 必须兼容的个人 Java 偏好

- 所有代码注释必须使用中文，专有名词和关键字可保留英文原文。
- public 类、public 方法、复杂逻辑必须有完整 JavaDoc。
- 方法 JavaDoc 包含功能描述、`@param`；有返回值时写 `@return`；显式抛出异常时写 `@throws`。
- 类文件头部必须包含 `@author 李杰`。
- 遵循阿里巴巴 P3C 编码规范，优先服从项目现有风格。
- 优先使用项目已有工具类、公共方法、本地 helper 和已引入的成熟工具类，避免重复造轮子。
- 中文路径和中文内容必须保持 UTF-8，不得基于乱码内容判断问题。

## 内置规则覆盖

- P3C 基础：命名、常量、集合、并发、异常、日志、日期时间、BigDecimal、空指针和序列化规则都纳入审查。
- Spring 分层：Controller 只做参数接收和响应转换，Service 承载业务和事务，Mapper/DAO 只做数据访问，禁止跨层乱依赖。
- MyBatis/SQL：动态 SQL 必须参数化，排序/表名/列名使用白名单，批量和分页要有边界，XML 与 Mapper 参数保持一致。
- 事务一致性：写操作、库存/金额/状态流转、消息发送和缓存更新要检查事务边界、回滚条件和幂等。
- 异常日志：禁止吞异常、重复打印堆栈、日志泄漏敏感信息；业务异常和系统异常要分层表达。
- 注释规范：public 类、public 方法、复杂逻辑必须中文 JavaDoc，类头包含 `@author 李杰`。
- 细分场景可结合本地 Java/P3C skill 的口径，例如 `java-personal-standards`、`java-common-standards`、`java-validation-annotations`、`java-controller`、`java-service`、`java-dao`、`p3c-coding-style`、`p3c-exception-logging`、`p3c-security-rules`。

## 执行流程

1. 明确审查范围：确认用户指定的文件、目录、模块或变更点；未指定时先搜索 Java 项目结构和关键入口。
2. 读取上下文：读取目标文件及其直接依赖，包括 Controller、Service、Mapper/XML、DTO/VO、异常处理、配置、工具类和测试。
3. 识别项目约定：查看包结构、统一返回类型、异常基类、日志工具、Bean Validation 分组、事务写法、MyBatis 写法、工具类偏好。
4. 按分级清单检查：先查阻断和高风险，再查中风险和建议优化；每个问题必须有文件路径和行号。
5. 交叉验证：同一问题至少结合调用链、注解、SQL/XML、DTO 字段或配置中的一种证据确认；无法确认时标为待验证，不要臆断。
6. 输出报告：按阻断、高风险、中风险、建议优化分组，说明原因、怎么检查到、修复建议和验证建议。
7. 如用户要求修复：先确认修复范围；多文件修改按安全批量修改规则执行；保持 UTF-8 和中文注释要求。

## 分级检查项

### 阻断

- 安全敏感信息泄露。
  怎么检查：搜索硬编码密码、token、密钥、数据库连接串、AK/SK、私钥、`@Value` 默认敏感值、日志输出敏感字段；检查配置文件和 Java 常量是否暴露真实凭据。
- SQL 注入或危险动态 SQL。
  怎么检查：搜索 MyBatis `${}`、字符串拼接 SQL、`order by` 直接拼接、`like '%" +`、`statement.execute`；结合入参来源确认是否使用参数绑定、白名单字段或枚举限定。
- 缺失关键权限或水平越权校验。
  怎么检查：检查 Controller/Service 的用户身份、租户、组织、数据归属校验；重点看按 id 查询、修改、删除接口是否只校验登录而未校验数据归属。
- XSS、路径穿越或不安全依赖造成安全风险。
  怎么检查：检查 Web 输出、富文本、模板变量是否按上下文转义；文件上传/下载路径是否 normalize 后限制在根目录；依赖版本、锁文件和安全公告是否存在明显高危风险。
- 对外入口完全缺少参数校验。
  怎么检查：检查 Controller 入参是否使用 `@Valid` / `@Validated`，DTO/VO 字段是否有 `@NotNull`、`@NotBlank`、`@Size`、`@Pattern` 等约束，分页、排序、文件路径、金额、数量是否有边界限制。
- 明显导致编译失败或运行主流程不可用的问题。
  怎么检查：检查类名/文件名不一致、缺失 import、调用不存在的方法、泛型或返回类型不匹配、Spring Bean 无法注入、Mapper 方法与 XML id 不一致。

### 高风险

- 事务边界错误导致数据不一致。
  怎么检查：检查多表写入、先写库后远程调用、循环写库、删除/更新组合是否有 `@Transactional`；检查 `try-catch` 吞异常、手动返回失败但未回滚、非 public 方法或同类内部调用导致事务失效。
- 异常处理破坏问题定位或吞掉失败。
  怎么检查：搜索 `catch (Exception`、空 catch、只 `printStackTrace`、只记录 `e.getMessage()`、重新抛出裸 `RuntimeException`；检查日志是否包含上下文和堆栈，返回给用户的信息是否泄露内部细节。
- NPE 高概率路径。
  怎么检查：检查数据库查询、远程调用、Map/List 取值、自动拆箱、级联 getter、`Optional.get()`、`Collection` 元素为空；结合返回值约定判断是否需要空值分支。
- MyBatis 结果错误或数据污染。
  怎么检查：检查 Mapper 接口参数是否缺少 `@Param`，XML 字段映射是否错别字，更新/删除是否缺少主键或逻辑删除条件，列表查询是否缺少分页或排序白名单。
- 分层职责混乱影响维护和测试。
  怎么检查：检查 Controller 是否承载业务逻辑或直接访问 Mapper，Service 是否处理 HTTP 细节，DAO 是否写业务判断，DTO/VO/DO 是否混用。
- public 类或 public 方法缺少符合个人规则的中文 JavaDoc。
  怎么检查：搜索 `public class`、`public interface`、`public enum`、`public .*\\(`，核对上方是否有 `/** */`，类注释是否含 `@author 李杰`，方法注释是否含必要的 `@param`、`@return`、`@throws`。

### 中风险

- P3C 命名、格式、常量和魔法值问题。
  怎么检查：检查类名 UpperCamelCase、方法和变量 lowerCamelCase、常量大写下划线、布尔字段不以 `is` 开头、缩进不使用 tab、魔法值是否抽为常量或枚举。
- Lombok 使用不符合项目约定或引入副作用。
  怎么检查：查看项目已有 POJO/VO/DTO 写法；检查是否过度使用 `@Data` 导致敏感字段进入 `toString`、实体类 `equals/hashCode` 风险、构造器注解与项目规范冲突。
- 重复造轮子或忽略项目已有工具类。
  怎么检查：搜索项目内同名工具、`*Util`、`*Utils`、公共常量、Hutool/Apache Commons/Guava 使用情况；对比新增逻辑是否重复实现字符串、集合、日期、对象转换、加密等通用能力。
- 日志级别和日志内容不合理。
  怎么检查：检查 debug/info/warn/error 使用场景，异常日志是否带堆栈，是否重复打印，是否把用户可控输入、手机号、身份证、token、密码等敏感信息原样输出。
- 性能和资源风险。
  怎么检查：检查循环内查库/远程调用、N+1 查询、未分页全量查询、大对象反复创建、流未关闭、正则在方法体内重复编译、集合初始化未估算容量。
- 缓存、统一响应或全局异常处理不一致。
  怎么检查：检查热点读取是否缺少缓存或缓存失效策略；Controller 返回是否绕过项目统一 `Result`/响应模型；全局异常处理是否遗漏校验异常、业务异常和系统异常映射。
- 文件过大或类职责膨胀。
  怎么检查：关注超过约 800 行的类、巨型 Service/Controller、多个领域职责混在同一类、内部私有方法堆积导致测试困难。
- 测试缺口。
  怎么检查：查看是否存在对应单元测试或集成测试；业务分支、参数校验、异常分支、事务回滚、Mapper SQL 是否有覆盖；无法运行测试时说明替代复核。

### 建议优化

- 方法过长、嵌套过深或职责过多。
  怎么检查：查看超过约 50 行的方法、超过 4 层嵌套、多个业务阶段混在一个方法；优先建议提取私有方法或领域方法，不做无关重构。
- 注释与代码不同步或表达不清。
  怎么检查：核对参数名、返回值、异常说明、业务含义是否与实现一致；注释必须中文且准确，不为自解释私有逻辑强行堆注释。
- 领域模型和返回结构可读性不足。
  怎么检查：检查 DO/DTO/VO/BO 是否职责清晰，统一返回类型是否符合项目现有封装，Map 返回是否可替换为明确 VO。
- 可维护性和一致性改进。
  怎么检查：对比同模块已有 Controller/Service/Mapper 风格；只建议与当前代码相邻、收益明确、风险低的调整。

## 常用搜索线索

Codex 优先使用 `rg` 和 `rg --files`，搜索结果需保留文件路径、行号和命中内容。不要用 Git 命令获取变更范围。

```powershell
rg --files -g "*.java" -g "*.xml"
rg -n "@RestController|@Controller|@RequestMapping|@PostMapping|@GetMapping" .
rg -n "@Transactional|catch \\(|rollbackFor|TransactionAspectSupport" .
rg -n "\\$\\{|order by|executeQuery|executeUpdate|Statement|like .*\\+" .
rg -n "@Valid|@Validated|@NotNull|@NotBlank|@NotEmpty|@Size|@Pattern|@Min|@Max" .
rg -n "public class|public interface|public enum|public .*\\(" .
rg -n "@author|@Author|/\\*\\*|TODO|FIXME" .
rg -n "password|passwd|secret|token|apiKey|accessKey|privateKey|AK|SK" .
rg -n "v-html|innerHTML|escapeHtml|HtmlUtils|normalize\\(|resolve\\(|PathTraversal|CVE|dependency-check" .
rg -n "printStackTrace|catch \\(Exception|RuntimeException\\(|log\\.error|LOG\\.error" .
rg -n "selectList|for \\(|stream\\(\\)|parallelStream|Pattern\\.compile|Cache|Redis|Result|ControllerAdvice|ExceptionHandler" .
```

针对 MyBatis XML，重点搜索：

```powershell
rg -n "<select|<insert|<update|<delete|resultMap|parameterType|jdbcType|LIMIT|delete_state" .
rg -n "#\\{|\\$\\{|<if|<foreach|ORDER BY|GROUP BY" .
```

## 多平台兼容

- Codex：搜索优先 `rg` / `rg --files`；读取中文文件显式使用 UTF-8；修改优先 `apply_patch`；不要执行 Git 操作。
- Claude：搜索优先 `Grep` / `Glob` / `Read`；编辑优先 `Edit` / `Write`；中文文件读写后复核编码。
- Trae：搜索优先 `SearchCodebase` / `Read`；编辑使用平台编辑工具；输出同样包含路径、行号和分级。
- Windows：路径优先使用正斜杠，PowerShell 读取中文文件时使用 `-LiteralPath` 和 `-Encoding UTF8`。
- Linux / macOS：可使用同样的 `rg` 搜索线索；路径分隔符保持平台原生格式。

## 输出格式

按以下结构输出，发现的问题必须按严重级别排序。没有命中的分组可省略；如果完全没有问题，明确写“未发现阻断/高风险/中风险问题”，并说明仍存在的验证限制。

```markdown
## 阻断
- 文件：path/to/File.java:42
  等级：阻断
  问题：一句话描述问题。
  怎么检查：说明通过哪条代码、调用链、注解、SQL 或配置确认。
  原因：说明违反的规则或实际风险。
  建议：给出可执行修复方向。
  验证：建议补充或运行的测试/构建/静态检查。

## 高风险
- 文件：path/to/File.java:88
  等级：高风险
  问题：一句话描述问题。
  怎么检查：说明证据。
  原因：说明风险。
  建议：给出修复方向。

## 中风险
- 文件：path/to/File.java:120
  等级：中风险
  问题：一句话描述问题。
  怎么检查：说明证据。
  建议：给出修复方向。

## 建议优化
- 文件：path/to/File.java:160
  等级：建议优化
  问题：说明可维护性或一致性优化点。
  怎么检查：说明与项目现有模式或规则的差异。
  建议：给出低风险改进方向。

## 复核说明
- 已读取范围：列出关键文件或目录。
- 未验证项：说明无法确认的运行时行为、测试或外部依赖。
```

## 审查边界

- 不默认执行 Git、构建、测试、API 请求或数据库操作；如用户要求运行验证，再按项目实际命令执行。
- 不把命令、agent 元数据或自动化流程原样迁移到审查过程。
- 不因个人偏好做大范围重构；只指出当前审查范围内影响正确性、安全性、可维护性和规范一致性的问题。
- 对证据不足的问题使用“待验证”表述，并给出下一步验证方法。
