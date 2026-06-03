---
name: "test-quality-language-python-review"
description: "Use when reviewing Python, FastAPI, Django, Pydantic, type hints, pytest, validation, or Python security quality."
---
# Python 语言专项质量审查

## 定位

用于 Python 代码质量专项审查，覆盖类型安全、输入校验、Web 框架、数据库访问、异步、资源释放、测试和安全风险。默认只做静态审查和结论输出，不自动修复，不默认执行 Git 操作，不跑真实 API 请求。

## 内置规则覆盖

- Python 基础：类型提示、异常边界、上下文管理器、资源释放、可变默认参数、全局状态、动态执行和包结构都纳入审查。
- Web 安全：FastAPI/Django 的认证授权、对象级权限、CSRF/CORS、SQL 注入、命令注入、路径穿越、反序列化和敏感配置要优先检查。
- 数据访问：ORM 查询、事务、N+1、select_related/prefetch、raw SQL、迁移和索引风险要结合调用场景判断。
- Pydantic/Serializer：外部输入、环境变量、文件上传、分页、排序和跨字段约束必须明确校验。
- 测试质量：pytest fixture、mock、异步测试、数据库测试隔离和安全/权限回归用例都要检查缺口。

## 检查流程

1. 确认审查范围：优先审查用户指定文件；未指定时只搜索 Python 相关文件与配置，不触碰 Git。
2. 识别技术栈：查找 `FastAPI`、`Django`、`Pydantic`、`SQLAlchemy`、`pytest`、`mypy`、`ruff`、`asyncio` 等线索。
3. 建立文件清单：包含业务代码、路由/视图、schema/serializer、service、repository/DAO、models、settings、tests。
4. 按分级清单逐项审查：先阻断和高风险，再中风险和建议优化。
5. 对每个问题给出文件路径、行号、等级、触发代码、风险原因、修复建议和验证方式。
6. 不能确认的问题不要臆断，说明缺少的上下文和下一步验证建议。

## 分级检查项

### 阻断

| 检查项 | 怎么检查 | 常用搜索线索 |
| --- | --- | --- |
| 硬编码密钥、密码、Token | 检查配置、settings、测试夹具和客户端初始化；确认是否把真实凭据写入代码或默认值。 | `SECRET_KEY`、`password=`、`token=`、`api_key`、`Authorization` |
| SQL 注入 | 检查 SQL 是否由 f-string、`%`、`format`、字符串拼接构造；ORM/raw SQL 是否使用参数绑定。 | `execute(`、`raw(`、`extra(`、`f"SELECT`、`%s`、`format(` |
| 命令注入 | 检查 `os.system`、`subprocess` 是否接收外部输入，是否启用 `shell=True`。 | `os.system`、`subprocess.`、`shell=True`、`Popen(` |
| 路径穿越与不安全文件访问 | 检查上传、下载、模板、静态文件和导入路径是否校验根目录、扩展名和归一化结果。 | `open(`、`Path(`、`send_file`、`FileResponse`、`../` |
| 不安全反序列化 | 检查是否对用户输入使用 `pickle`、`yaml.load` 或动态导入执行。 | `pickle.loads`、`pickle.load`、`yaml.load`、`eval(`、`exec(` |
| 认证/授权缺失 | FastAPI 检查路由依赖和安全方案；Django/DRF 检查 `permission_classes`、`IsAuthenticated`、对象级权限。 | `@router.`、`Depends(`、`Security(`、`permission_classes`、`AllowAny` |

### 高风险

| 检查项 | 怎么检查 | 常用搜索线索 |
| --- | --- | --- |
| 类型标注缺失或失真 | 检查 public 函数、服务方法、仓储方法、异步函数的参数和返回值；关注 `Any`、裸 `dict/list`、`Optional` 未处理。 | `def `、`async def `、`Any`、`dict`、`list`、`Optional`、`None` |
| Pydantic/FastAPI 输入校验不足 | 检查请求体、查询参数、路径参数是否有 `Field` 范围、长度、格式、必填约束和 validator。 | `BaseModel`、`Field(`、`field_validator`、`Query(`、`Path(`、`Body(` |
| Django/DRF Serializer 校验不足 | 检查 serializer 是否覆盖跨字段校验、密码校验、只读字段、敏感字段输出。 | `serializers.`、`validate`、`validate_`、`fields =`、`read_only_fields` |
| 异常被吞或泄露细节 | 检查裸 `except`、`except Exception` 后只 `pass/print`，或把内部异常、SQL、堆栈直接返回给客户端。 | `except:`、`except Exception`、`pass`、`print(`、`traceback`、`HTTPException` |
| 依赖注入失效 | FastAPI 检查路由中是否直接 new 数据库/客户端/服务；测试是否难替换依赖。Django 检查 service/repository 是否和视图强耦合。 | `Depends(`、`get_db`、`SessionLocal()`、`Service()`、`requests.` |
| ORM 事务和会话错误 | SQLAlchemy 检查 session 生命周期、commit/rollback；Django 检查跨表写入是否使用 `transaction.atomic`。 | `Session`、`commit(`、`rollback(`、`transaction.atomic`、`objects.create` |
| 异步阻塞或漏 await | 检查 async 路由/任务中是否调用同步 IO、CPU 阻塞、`time.sleep`，以及 coroutine 是否未 await。 | `async def`、`await`、`time.sleep`、`requests.`、`asyncio.create_task` |
| 资源未释放 | 检查文件、网络连接、数据库 session、锁、临时目录是否使用 context manager 或 finally 释放。 | `open(`、`connect(`、`close(`、`with `、`async with`、`finally` |
| 关键逻辑缺测试 | 检查安全校验、异常分支、边界值、无权限、无数据、事务失败是否有 pytest 覆盖。 | `pytest`、`raises`、`parametrize`、`TestClient`、`client.` |
| 依赖存在安全风险 | 检查 requirements、pyproject、lock 文件中新增包、过旧版本、未固定版本和已知高危依赖；不能联网时标记需安全扫描确认。 | `requirements`、`pyproject.toml`、`poetry.lock`、`Pipfile.lock` |

### 中风险

| 检查项 | 怎么检查 | 常用搜索线索 |
| --- | --- | --- |
| Django ORM N+1 | 检查循环内访问外键/多对多；确认使用 `select_related`、`prefetch_related` 或聚合查询。 | `for `、`.all()`、`.filter(`、`select_related`、`prefetch_related` |
| 过大函数或深层嵌套 | 检查函数超过约 50 行、文件超过约 800 行、嵌套超过 3 层时是否可拆分。 | `def `、`class `、`if `、`for `、`try:` |
| 可变默认参数和共享状态 | 检查默认 `[]/{}`、全局缓存、模块级可变对象是否造成状态污染。 | `=[]`、`={}`、`global `、`cache =` |
| 全局可变状态滥用 | 检查模块级 list/dict/set、单例客户端、全局缓存是否跨请求污染或破坏测试隔离。 | `global `、`=[]`、`={}`、`set()`、`dict()` |
| 日志与错误观测不足 | 检查是否用 `print` 代替 logging，异常日志是否缺上下文或泄露敏感信息。 | `print(`、`logging.`、`logger.` |
| Pydantic 版本混用 | 检查 v1/v2 API 是否混用，例如 `validator` 与 `field_validator`、`Config` 与 `model_config`。 | `validator`、`field_validator`、`Config`、`model_config` |
| 配置环境隔离不足 | Django 检查 `DEBUG`、`ALLOWED_HOSTS`、Cookie secure、CSRF、HSTS；FastAPI 检查 CORS 和环境变量默认值。 | `DEBUG`、`ALLOWED_HOSTS`、`CORS`、`CSRF_COOKIE_SECURE`、`SECURE_` |

### 建议优化

| 检查项 | 怎么检查 | 常用搜索线索 |
| --- | --- | --- |
| 工具链一致性 | 检查是否配置 `ruff`、`mypy`、`pytest`、`pyproject.toml`，以及 CI 是否执行关键检查。 | `pyproject.toml`、`mypy`、`ruff`、`pytest` |
| PEP8 命名和格式 | 检查模块、包、函数、变量、类、常量命名是否符合项目规范；优先使用 f-string，避免旧式 `%` 或无必要 `.format`。 | `class `、`def `、`% `、`.format(`、`f"` |
| DTO/VO 表达清晰 | 优先使用 Pydantic、dataclass、TypedDict 或明确模型，不用无约束裸 dict 传递复杂结构。 | `dict[str`、`TypedDict`、`dataclass`、`BaseModel` |
| 不可变和值对象 | 值对象可使用 `frozen=True` dataclass 或 Pydantic frozen 配置，减少隐式修改。 | `@dataclass`、`frozen`、`BaseModel` |
| 查询性能与缓存 | 检查重复查询、未分页、未限制字段、缓存 key 过宽或失效策略不清晰。 | `.all()`、`limit`、`offset`、`cache.get`、`cache.set` |
| 推导式和生成器使用 | 检查列表推导、生成器表达式和内置函数是否提升可读性；大数据量场景避免一次性 list 占用过多内存。 | `for `、`append(`、`yield`、`list(` |
| 测试可维护性 | 检查 fixture 是否清晰、参数化是否覆盖边界、mock 是否只用于外部依赖。 | `fixture`、`mock`、`parametrize`、`raises` |

## 框架专项关注

- FastAPI：路由层只做协议适配；业务逻辑放 service；数据库 session、当前用户、配置、客户端通过 `Depends` 或显式构造注入；请求参数用 `Query/Path/Body` 约束；响应模型避免返回敏感字段。
- Pydantic：所有外部输入都要有 schema；字段要有长度、范围、枚举、格式、默认值语义；复杂校验使用 validator；关注 v1/v2 API 一致性。
- Django/DRF：settings 按环境拆分；生产关闭 `DEBUG` 并配置安全 Cookie、CSRF、HSTS；Serializer 承担输入输出约束；ViewSet 有权限、过滤、分页；批量写入或跨表一致性使用事务。
- ORM：优先参数化查询和 ORM 表达式；raw SQL 必须参数绑定；查询循环关注 N+1；session/transaction 生命周期必须清楚。
- 异步：async 代码只调用可 await 的 IO；同步库放线程池或替换为异步客户端；后台任务要处理异常、取消和生命周期。

## 常用搜索流程

- 文件清单：`rg --files -g "*.py" -g "pyproject.toml" -g "requirements*.txt"`
- 类型与异步：`rg -n "def |async def |Any|Optional|typing|await|time\\.sleep|requests\\."`
- 校验与 Web：`rg -n "BaseModel|Field\\(|validator|field_validator|APIRouter|Depends\\(|Query\\(|Path\\(|Body\\(|serializers\\.|permission_classes"`
- 数据库与 SQL：`rg -n "execute\\(|raw\\(|extra\\(|select_related|prefetch_related|transaction\\.atomic|SessionLocal|commit\\(|rollback\\("`
- 安全危险函数：`rg -n "SECRET|password|token|api_key|os\\.system|subprocess|shell=True|pickle\\.load|yaml\\.load|eval\\(|exec\\("`
- 依赖与风格：`rg --files -g "requirements*.txt" -g "pyproject.toml" -g "poetry.lock" -g "Pipfile.lock"`；`rg -n "% |\\.format\\(|global |=\\[\\]|=\\{\\}|append\\("`
- 测试：`rg -n "pytest|parametrize|raises|TestClient|fixture|mock"`

搜索结果必须保留文件路径、行号和命中的具体代码。若平台搜索工具无法自动给行号，读取文件后手动补齐行号。

## 输出格式

按以下顺序输出：阻断、高风险、中风险、建议优化。没有对应问题时写“未发现”。

```markdown
## 阻断
- [等级] 问题标题
  - 位置：path/to/file.py:10
  - 代码：`命中的关键代码`
  - 原因：说明风险和触发条件
  - 建议：给出可执行修复方向
  - 验证：说明应补充的测试、静态检查或人工验证

## 高风险
...

## 中风险
...

## 建议优化
...

## 复核说明
- 已检查范围：
- 未能确认：
- 建议下一步：
```

结论规则：存在阻断或高风险时，结论为“不建议通过”；仅有中风险时，结论为“需修复后通过”；只有建议优化或无问题时，结论为“可通过，并列出残余风险”。

## 多平台兼容

- Codex：优先使用 `rg`、`rg --files` 搜索；读取中文路径时显式 UTF-8；不执行任何 Git 命令，除非用户明确要求。
- Claude：优先使用 `Grep`、`Glob`、`Read`；输出中保留路径、行号、代码片段和分级。
- Trae：优先使用 `SearchCodebase`、`Read`；多文件审查先列范围，再逐类归纳问题。
- 所有平台：默认只审查不修改；如用户要求修复，先确认影响范围，普通代码可直接改，删除数据、生产配置、安全敏感项必须再次确认。
