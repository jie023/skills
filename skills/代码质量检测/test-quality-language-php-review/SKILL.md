---
name: "test-quality-language-php-review"
description: "Use when reviewing PHP or Laravel code quality, security, testing, framework patterns, imports, namespaces, or coding style."
---
# PHP 语言专项质量审查

## 定位

PHP/Laravel/Symfony 专项代码质量审查，重点覆盖 PSR-12、类型安全、输入输出安全、数据库安全、Auth/Session、服务分层、测试覆盖和调试残留。

## 适用范围

- 用户要求审查 PHP、Laravel、Symfony、Composer、PHPUnit、Pest、PHPStan、Psalm 相关代码时使用。
- 涉及 Controller、FormRequest、DTO、Service、Repository、Eloquent/Doctrine、Blade/Twig、Session/Auth、composer 依赖和测试时使用。
- 只做静态代码质量审查，不启用 hooks，不默认运行 Pint、PHP-CS-Fixer、PHPStan、Psalm、PHPUnit、Pest 或 `composer audit`；需要运行时先说明目的并等待确认。

## 检查流程

1. 明确审查范围：优先审查用户指定的 `.php`、`composer.json`、测试配置和相关模板文件。
2. 识别框架：Laravel、Symfony、原生 PHP、Inertia、Doctrine/Eloquent、PHPUnit/Pest。
3. 读取上下文：从路由/Controller/FormRequest 到 Service/Domain/Repository/Model/View/Test 建立调用链。
4. 按分级检查项判断：先安全和数据风险，再看类型、分层、测试和风格。
5. 每个问题必须包含文件路径、行号、证据、风险、建议和验证方式；无法确认时说明缺少的信息。

## 分级检查项

### 阻断

- **SQL 注入或危险查询**：Controller/View/Service 中拼接 SQL，动态排序/字段/表名未白名单。
  - 怎么检查：搜索 `DB::raw`、`whereRaw`、`orderByRaw`、`selectRaw`、`PDO::query`、字符串拼接 SQL。
- **XSS 或未转义输出**：Blade/Twig/raw HTML 输出用户输入或富文本未过滤。
  - 怎么检查：搜索 `{!!`、`|raw`、`echo`、`htmlspecialchars` 缺失、富文本字段输出点。
- **认证、授权、CSRF 或 Session 风险**：状态变更请求缺 CSRF，登录后未刷新 session，敏感接口无 policy/gate/middleware。
  - 怎么检查：查看路由 middleware、policy/gate、FormRequest authorize、session regenerate 和 CSRF 配置。
- **mass assignment 写入越权字段**：`fillable/guarded` 不当，用户可写角色、状态、金额、租户等敏感字段。
  - 怎么检查：查看 Model `$fillable/$guarded`、`create/update/fill` 的请求数据来源和过滤。
- **硬编码密钥和敏感配置**：密码、token、连接串、私钥写在代码或提交配置里。
  - 怎么检查：搜索 `password`、`secret`、`token`、`api_key`、`.env`、连接串和 config 默认值。
- **危险调试语句进入主流程**：`dd`、`dump`、`var_dump`、`die`、`exit` 出现在生产可达路径。
  - 怎么检查：搜索调试函数并确认是否在业务代码、Controller、Job 或模板中。

### 高风险

- **输入校验缺失**：请求数据未通过 FormRequest、Symfony Validator、DTO 或显式校验就进入业务逻辑。
  - 怎么检查：从 Controller 参数和 `$request->input/all` 追踪到 Service/Model/SQL。
- **类型安全不足**：新代码缺少 `declare(strict_types=1);`、参数/返回类型、typed property，或用数组承载复杂业务结构。
  - 怎么检查：读取文件头、函数签名、DTO/Value Object、array shape 和 docblock。
- **密码和敏感数据处理不安全**：未用 `password_hash/password_verify`，敏感字段未脱敏或日志泄露。
  - 怎么检查：搜索密码处理、日志调用、response resource/array 输出和隐藏字段。
- **事务和数据一致性风险**：多表写入、库存/余额/订单状态变更缺事务、幂等或异常回滚。
  - 怎么检查：查看 `DB::transaction`、Doctrine transaction、Job 重试和异常处理。
- **分层职责混乱**：Controller 写业务规则，Model 过度承担 domain 决策，Service 直接处理 HTTP 细节。
  - 怎么检查：沿调用链查看 auth/validation/serialization/business/persistence 是否各在合适层。
- **依赖安全风险**：新增 Composer 包来源不明、废弃包、版本过宽或缺锁文件。
  - 怎么检查：查看 `composer.json`、`composer.lock` 和包用途；不能联网时标记需 audit 验证。

### 中风险

- **测试覆盖不足**：缺少 PHPUnit/Pest 对校验、权限、异常、数据库、Service 规则和回归场景的测试。
  - 怎么检查：查 `tests/`、`phpunit.xml`、Pest 配置和相关测试命名。
- **数据库性能风险**：Eloquent N+1、未分页、未限制字段、循环查询、缺 eager loading。
  - 怎么检查：搜索 `foreach` 中查询、`all()`、`get()`、`with`、`paginate`、`chunk`。
- **DTO/Value Object 不足**：复杂请求、命令、第三方 payload 使用大数组传递，业务含义不清。
  - 怎么检查：查看数组 key 是否跨层传播，金额、ID、日期范围是否有明确类型。
- **异常处理和日志不清晰**：返回 `false/null` 隐藏错误，catch 后吞异常，日志缺上下文或泄露敏感信息。
  - 怎么检查：搜索 `catch`、`return false`、`return null`、`Log::`、`logger`。
- **风格和静态分析缺口**：PSR-12、Pint/PHP-CS-Fixer、PHPStan/Psalm 配置缺失或未纳入项目脚本。
  - 怎么检查：查看 `composer.json` scripts、`phpstan.neon`、`psalm.xml`、Pint 配置。
- **命名空间和 import 依赖混乱**：业务代码依赖 global namespace、全限定类名散落或 `use` 分组不符合项目规范。
  - 怎么检查：查看 `namespace`、`use`、全限定类名、helper 函数和同模块 import 组织。

### 建议优化

- **不可变 DTO/只读属性**：PHP 8.1+ 可用 `readonly`、构造器注入和值对象减少隐式修改。
- **依赖注入改进**：服务通过构造函数注入接口或窄契约，避免 service locator 和静态全局依赖。
- **测试组织优化**：分离快速单元测试和数据库/HTTP 集成测试，使用 factory/builder 减少大数组 fixture。
- **Inertia 测试补充**：使用 Inertia 的页面应检查 props、权限、表单错误、重定向和关键交互测试是否覆盖。
- **格式和 import 清理**：统一 `use`、命名空间、类名、方法名和项目格式化工具。

## 常用搜索线索

按当前平台工具执行搜索，结果必须包含文件路径、行号和具体命中内容。

```text
declare\\(strict_types=1\\)|function .*\\(|public function|private function
\\$request->all\\(|\\$request->input\\(|FormRequest|Validator::|rules\\(
DB::raw|whereRaw|orderByRaw|selectRaw|PDO::query|prepare\\(
\\{!!|\\|raw|htmlspecialchars|e\\(
fillable|guarded|create\\(|update\\(|fill\\(
auth\\(|Gate::|Policy|middleware|csrf|session\\(\\)->regenerate
password_hash|password_verify|Hash::make|Hash::check
var_dump|dd\\(|dump\\(|die\\(|exit\\(
PHPUnit|Pest|it\\(|test\\(|assert|factory\\(
Inertia|assertInertia|component\\(|props\\(|namespace |use [A-Za-z_\\\\]+;|\\\\App\\\\
```

## 判断标准

- 阻断：可能导致注入、XSS、越权、CSRF、敏感字段误写、敏感信息泄露或生产中断。
- 高风险：输入校验、类型、事务、分层、依赖或密码处理存在明显线上风险。
- 中风险：测试、性能、DTO、异常日志或静态分析存在维护和质量隐患。
- 建议优化：不可变性、依赖注入、测试组织、格式和 import 可提升一致性。

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
  - 位置：path/to/File.php:line
  - 证据：命中的关键代码或调用链
  - 原因：为什么违反 PHP/Laravel 质量要求
  - 建议：如何修改或下一步验证
```

没有发现对应等级问题时写“未发现”。无法确认的问题要说明原因和下一步验证建议。

## 内置规则覆盖

- PHP 规范：PSR 风格、严格类型、命名空间、自动加载、值对象、DTO、异常和日志都纳入审查。
- 依赖组织：避免业务逻辑依赖 global namespace 和散落全限定类名；import、namespace、helper 使用要与项目规范一致。
- Laravel/Symfony：Controller 保持薄，Request/Form 校验输入，Service 承载业务，Repository/Model 边界清晰。
- 安全：SQL 注入、XSS、CSRF、文件上传、路径穿越、反序列化、调试配置和 `.env` 泄露优先检查。
- 数据访问：N+1、事务、批量更新、软删除、权限作用域和分页限制要结合业务场景判断。
- 测试：Feature/Unit、数据库隔离、权限回归、表单校验、异常路径、队列/事件和 Inertia 页面测试是审查依据。

## 多平台兼容

- Codex：优先使用 `rg`、`rg --files`、`Get-Content -Encoding UTF8`，复杂任务使用 `update_plan`。
- Claude：优先使用 `Read`、`Grep`、`Glob`，复杂任务使用可用的 Todo 工具。
- Trae：优先使用 `SearchCodebase`、`Read`，复杂任务使用可用的任务清单工具。
- 任一平台都必须遵守：不默认执行 Git、不自动修复、不批量改文件、中文内容保持 UTF-8。
