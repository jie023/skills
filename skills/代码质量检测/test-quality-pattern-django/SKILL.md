---
name: "test-quality-pattern-django"
description: "Use when reviewing Django or DRF code involving settings, middleware, models, ORM queries, migrations, serializers, viewsets, permissions, transactions, tests, or production security."
---
# Django 模式质量审查

## 定位

Django / DRF 代码质量审查 skill，用于静态检查项目中的架构分层、安全配置、数据访问、接口权限、事务一致性、性能风险和测试覆盖。审查目标是输出问题、风险依据和修复建议，不默认修改业务代码。

## 内置规则覆盖

- settings 安全：检查 `DEBUG`、`SECRET_KEY`、`ALLOWED_HOSTS`、CSRF、CORS、Cookie 安全属性、环境变量和日志敏感信息。
- 中间件与认证：检查 middleware 顺序、认证后端、权限默认值、匿名访问兜底和多租户/对象级权限。
- ORM 与模型：检查 N+1、`select_related`/`prefetch_related`、索引、唯一约束、软删除、时间字段和模型方法副作用。
- migration 风险：检查不可逆迁移、大表锁表、默认值回填、字段 rename/drop、数据迁移幂等和回滚方案。
- DRF 层：检查 serializer 校验、viewset action 边界、pagination/filter/order、异常响应、权限类和输入输出字段泄露。
- 事务一致性：检查 `transaction.atomic` 范围、外部调用入事务、`on_commit`、并发更新、锁粒度和幂等请求。
- 缓存策略：检查缓存 key 维度、权限/租户隔离、TTL、主动失效、序列化兼容、缓存穿透/击穿和敏感数据缓存。
- Signal 模式：检查 signal 注册位置、重复注册、幂等、副作用、事务内触发、`on_commit`、异常传播和测试可控性。
- 测试覆盖：检查权限、serializer、viewset、model constraint、migration 和关键 ORM 查询是否有可验证测试。

## 执行边界

- 只做代码层质量审查，不跑 API 请求。
- 不默认执行任何 Git 操作。
- 不自动修复问题，只输出问题和修复建议。
- 输出必须包含文件路径、行号、问题等级、原因、影响和建议。
- 报告输出遵循 `test-quality-report-template`。
- 如需要修改代码，必须等待用户确认后逐项处理。
- 多文件修改必须按安全批量修改规则执行。
- 中文路径和中文内容必须保持 UTF-8。

## 检查流程

1. 确认项目类型：识别 `settings.py`、`settings/`、`manage.py`、`urls.py`、`INSTALLED_APPS`、`REST_FRAMEWORK`、Django/DRF 版本线索。
2. 读取安全入口：检查 settings、安全中间件、认证配置、CORS/CSRF、cookie、host、debug、日志和密钥来源。
3. 串联接口链路：从 URL/router 到 view/viewset，再到 serializer、permission、queryset、model、migration。
4. 检查数据访问：审查 ORM 查询、跨租户过滤、对象级权限、N+1、索引、分页、批量操作和缓存一致性。
5. 检查写操作一致性：关注多表写入、库存/余额/状态流转、`transaction.atomic`、`select_for_update`、`F()` 表达式、幂等和并发。
6. 检查 migrations：评估不可逆迁移、数据迁移、字段默认值、唯一约束、索引、锁表、线上兼容和回滚路径。
7. 检查测试：查看权限、serializer 校验、viewset 行为、事务、迁移、N+1 查询数、边界输入和回归测试。
8. 输出结论：只报告有证据的问题；无法确认的风险必须说明缺失信息和下一步验证方式。

## 分级标准

### 阻断

满足任一项即可列为阻断：

- 明确生产环境 `DEBUG = True`、硬编码真实 `SECRET_KEY`、数据库密码、令牌或第三方密钥。
- `ALLOWED_HOSTS = ['*']` 或等价全放开，且代码明显用于生产配置。
- 认证或对象级权限缺失导致任意用户可读写他人数据、租户数据或管理数据。
- serializer 暴露或允许写入敏感字段，例如 `is_staff`、`is_superuser`、`password`、`owner`、`tenant`、余额、状态机关键字段。
- 通过字符串拼接执行 SQL、`extra()`、`RawSQL` 或 `raw()`，且参数来自请求输入，存在注入风险。
- 涉及资金、库存、订单、权限、状态流转的多表写入缺少事务保护，可能产生脏数据或重复扣减。
- migration 明确会删除字段/表、重建大表、不可逆数据变更或长时间锁表，且没有回滚/兼容策略。

### 高风险

- CORS/CSRF/cookie 安全配置过宽，例如 `CORS_ALLOW_ALL_ORIGINS = True`、跨站请求未配 CSRF、`SESSION_COOKIE_SECURE` 或 `CSRF_COOKIE_SECURE` 缺失。
- `DEFAULT_PERMISSION_CLASSES` 默认放开，或 view/viewset 使用 `AllowAny` 但接口包含非公开数据。
- `get_queryset()` 未按 `request.user`、组织、租户、状态过滤，存在横向越权风险。
- `get_object()`、custom action、函数视图绕过 object permission 或直接 `Model.objects.get(pk=...)`。
- `fields = '__all__'`、`exclude = (...)`、动态字段拼接导致输入/输出面不可控。
- `perform_create`、`create`、`update` 未绑定当前用户/租户，信任客户端传入所有权字段。
- 中间件顺序破坏安全链路，例如 `SecurityMiddleware` 缺失或过晚，认证依赖的 session/auth middleware 顺序错误。
- 数据迁移 `RunPython`/`RunSQL` 无反向逻辑、无幂等保护，或一次性全表加载大数据。

### 中风险

- 列表接口、嵌套 serializer、模板循环或 custom action 中存在明显 N+1 查询，缺少 `select_related` / `prefetch_related`。
- 列表接口未分页或允许无限 `page_size`、无限排序、无限搜索字段。
- 高频过滤、排序、唯一性判断缺少索引、约束或数据库级校验。
- 复杂业务逻辑堆在 viewset/serializer 中，难以复用、测试和事务化。
- serializer 只依赖前端校验，缺少 `validate_<field>`、`validate` 或 model constraint。
- `get_or_create`、先查后改、计数后写入等并发敏感逻辑缺少唯一约束、锁或重试策略。
- 测试只覆盖 happy path，缺少权限拒绝、越权访问、边界输入、事务回滚或查询数断言。

### 建议优化

- settings 可拆分为 base/development/production/test，并用环境变量注入密钥和数据库配置。
- 公共查询沉淀为 QuerySet/Manager，避免重复过滤条件和权限条件散落。
- 跨模型业务写入沉淀到 service 层或 domain service，viewset 只负责协议适配。
- serializer 按 read/write/action 拆分，减少条件字段和过宽输入面。
- 为复杂迁移添加分阶段发布说明：先兼容字段，再回填，再切换读写，最后清理旧字段。
- 增加 `assertNumQueries`、pytest query count、factory、APIClient/APITestCase 覆盖关键接口。

## 怎么检查

### Settings 与安全

- 检查 `DEBUG`、`SECRET_KEY`、`ALLOWED_HOSTS`、`DATABASES` 是否从环境读取，生产配置是否安全默认关闭。
- 检查 `SECURE_SSL_REDIRECT`、`SECURE_HSTS_SECONDS`、`SECURE_HSTS_INCLUDE_SUBDOMAINS`、`SECURE_HSTS_PRELOAD`。
- 检查 `SESSION_COOKIE_SECURE`、`CSRF_COOKIE_SECURE`、`SESSION_COOKIE_HTTPONLY`、`CSRF_COOKIE_HTTPONLY`、`SECURE_REFERRER_POLICY`。
- 检查 `CORS_ALLOWED_ORIGINS`、`CORS_ALLOW_ALL_ORIGINS`、`CSRF_TRUSTED_ORIGINS` 是否最小化。
- 检查 `REST_FRAMEWORK` 默认认证、权限、分页、限流、异常处理、renderer/parser。

### Middleware

- 确认 `SecurityMiddleware` 靠前，`SessionMiddleware` 在 `AuthenticationMiddleware` 前。
- 使用 session auth 时确认存在 `CsrfViewMiddleware`，并核对接口是否通过装饰器绕过。
- 检查自定义 middleware 是否每个分支都返回 response、是否吞异常、是否在请求链路中做重查询或写库。
- 关注记录日志的 middleware 是否泄露 token、cookie、密码、身份证、手机号等敏感数据。

### ORM、QuerySet 与 N+1

- 从 serializer 字段、模板字段、循环访问的外键/多对多关系倒推 queryset 是否预加载。
- ForeignKey/OneToOne 用 `select_related`，ManyToMany/reverse FK 用 `prefetch_related` 或 `Prefetch`。
- 检查循环内 `.get()`、`.filter()`、`.count()`、`.exists()`、`.all()`、`.save()` 是否造成重复查询或逐行写入。
- 检查 `annotate`、`aggregate`、`Subquery`、`values` 是否比 Python 循环汇总更合适。
- 检查高频查询条件是否有 `db_index`、`UniqueConstraint`、`Index`、`CheckConstraint`。

### Migrations

- 读取新增/变更 migration，确认模型变更与 migration 一致，没有遗漏生成或手改错误。
- 对 `RemoveField`、`DeleteModel`、`AlterField(null=False)`、新增唯一约束、重命名字段做高风险审查。
- `RunPython` 必须优先使用 `apps.get_model`，避免直接导入当前 model。
- 数据迁移应具备幂等、分批、反向逻辑或明确不可逆说明；大表变更要评估锁表和发布顺序。
- 不把请求时业务逻辑、外部 API 调用、不可重复随机逻辑写入 migration。

### Serializer、ViewSet 与 Permissions

- serializer 检查 `fields`、`read_only_fields`、`write_only`、`extra_kwargs`、`validate`、`create`、`update`。
- 密码必须使用 `set_password` 或项目密码服务，不允许直接保存明文。
- viewset 检查 `queryset`、`get_queryset`、`get_serializer_class`、`perform_create`、`perform_update`、custom action。
- permissions 检查 `permission_classes`、`get_permissions`、`has_permission`、`has_object_permission` 是否覆盖列表和详情场景。
- 自定义 action 要单独检查 method、权限、serializer、事务、分页、对象级权限和返回状态码。
- 不信任客户端传入 `owner`、`user`、`tenant`、`organization`、`status`、`role` 等权限相关字段。

### Transactions 与并发

- 多表写入、状态机变更、余额/库存/额度扣减必须明确事务边界。
- 并发写同一行数据时优先检查 `select_for_update`、`F()` 表达式、唯一约束和幂等键。
- 事务内避免慢外部调用、发邮件、HTTP 请求、消息推送；必要时使用 `transaction.on_commit`。
- 捕获异常后不能静默吞掉事务失败；需要确认回滚、日志和错误响应一致。

### Tests

- 权限测试必须覆盖匿名、普通用户、资源拥有者、同组织/不同组织、管理员。
- serializer 测试覆盖必填、非法值、只读字段、敏感字段写入、嵌套数据和密码处理。
- viewset/API 测试覆盖列表、详情、创建、更新、删除、custom action、分页、过滤、排序。
- 事务和并发敏感逻辑至少覆盖失败回滚、重复提交、库存/余额不足等场景。
- 性能关键列表建议覆盖查询数断言，例如 Django `assertNumQueries` 或项目等价工具。
- migration 涉及数据变更时建议有迁移前后数据样例或单独迁移测试。

## 搜索线索

Codex 优先使用 `rg`，搜索结果必须保留文件路径、行号和命中的代码内容。常用线索：

```bash
rg -n "DEBUG|SECRET_KEY|ALLOWED_HOSTS|SECURE_|SESSION_COOKIE|CSRF_COOKIE|CORS_|CSRF_TRUSTED_ORIGINS|REST_FRAMEWORK|MIDDLEWARE" .
rg -n "permission_classes|authentication_classes|AllowAny|IsAuthenticated|get_permissions|has_object_permission|@permission_classes" .
rg -n "class .*Serializer|fields = '__all__'|exclude = |read_only_fields|write_only|validate_|def validate\\(|def create\\(|def update\\(" .
rg -n "class .*ViewSet|ModelViewSet|GenericViewSet|APIView|@action|def get_queryset|def get_object|perform_create|perform_update" .
rg -n "select_related|prefetch_related|Prefetch|iterator\\(|bulk_create|bulk_update|raw\\(|RawSQL|extra\\(" .
rg -n "transaction\\.atomic|select_for_update|F\\(|get_or_create|update_or_create|on_commit" .
rg -n "RunPython|RunSQL|RemoveField|DeleteModel|AlterField|AddConstraint|AddIndex|unique=True|null=False" .
rg -n "TestCase|APITestCase|APIClient|pytest|assertNumQueries|permission|serializer|migration" .
```

按文件名补充定位：

- settings：`rg --files | rg "settings|base\\.py|production\\.py|development\\.py|test\\.py"`
- URLs/router：`rg --files | rg "urls\\.py|routers?\\.py"`
- DRF：`rg --files | rg "serializers\\.py|views\\.py|viewsets\\.py|permissions\\.py|filters\\.py"`
- 数据层：`rg --files | rg "models\\.py|managers\\.py|querysets\\.py|services\\.py|migrations"`
- 测试：`rg --files | rg "tests|test_.*\\.py|.*_test\\.py"`

## 输出格式

报告按以下顺序输出，发现为空也要说明：

1. 阻断
2. 高风险
3. 中风险
4. 建议优化
5. 无法确认与验证建议

每条问题使用统一格式：

```markdown
- [等级] 文件路径:行号 - 问题标题
  - 证据：引用命中的关键代码或配置。
  - 影响：说明可能导致的安全、数据一致性、性能或维护风险。
  - 建议：给出可执行的修复方向，必要时指出应补充的测试。
```

如果没有发现问题，明确说明“未发现阻断/高风险问题”，并列出仍未覆盖的测试或运行时验证缺口。

## 多平台兼容

- Codex：使用 `rg` / `rg --files` 搜索，读取中文文件时显式 UTF-8；不执行 Git。
- Claude：使用 `Grep` / `Glob` / `Read` 搜索和读取；输出仍保留路径、行号和代码证据。
- Trae：使用 `SearchCodebase` / `Read` 搜索和读取；按同一分级和输出格式报告。
- 任意平台都不得默认调用外部 API、启动服务、执行数据库迁移、修改生产配置或自动修复代码。
