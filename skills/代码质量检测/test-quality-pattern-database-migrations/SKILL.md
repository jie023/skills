---
name: "test-quality-pattern-database-migrations"
description: "Use when reviewing database migrations, rollback safety, batching, online DDL, schema evolution, or migration risk."
---
# 数据库迁移模式质量审查

## 定位

数据库迁移模式质量审查，重点判断 schema 变更、数据回填、回滚、零停机发布、大表 DDL 和迁移工具使用是否安全。

## 适用范围

- 用户要求审查迁移脚本、DDL、DML、回滚方案、数据库发布、表结构演进或 migration 工具时使用。
- 涉及 PostgreSQL、MySQL、Prisma、Drizzle、Kysely、Django migration、TypeORM、golang-migrate 等场景时使用。
- 默认只做静态审查，不连接数据库、不执行 SQL、不运行迁移命令；需要执行时必须先取得用户确认。

## 检查流程

1. 明确范围：读取 migration 文件、schema/model 变更、应用代码兼容改动、回滚脚本和发布说明。
2. 区分类型：schema 变更、数据回填、索引变更、约束变更、列重命名/删除、迁移工具配置。
3. 评估数据规模：表大小、锁影响、事务时长、批大小、执行窗口、是否生产等无法确认时标记待验证。
4. 检查兼容发布：确认旧代码和新代码在迁移前后是否都能运行，必要时使用 expand-contract。
5. 检查回滚和验证：确认回滚方案、备份、数据一致性校验、失败重试和脏状态处理。
6. 每个问题必须给出文件路径、行号、风险、建议和下一步验证方式。

## 分级检查项

### 阻断

- **破坏性 DDL 无兼容发布**：直接重命名/删除列、改类型、拆表合表，应用代码仍可能访问旧结构。
  - 怎么检查：对比 migration、model/schema、SQL 和应用查询；确认是否先扩展、双写/双读、回填、再收缩。
- **大表锁表风险**：大表直接 `ALTER TABLE`、`CREATE INDEX`、加 NOT NULL/DEFAULT、全量更新。
  - 怎么检查：搜索 DDL 和全表 DML，确认是否使用 concurrent/online DDL、批处理、执行窗口和锁影响说明。
- **Schema 和数据迁移混在一个长事务**：DDL 与全量 backfill 同文件同事务执行。
  - 怎么检查：读取迁移文件，确认 schema change 和 data backfill 是否拆分。
- **已部署迁移被修改**：修改历史 migration 文件导致环境漂移。
  - 怎么检查：查看迁移命名、时间戳、迁移链和发布记录；无法确认是否已部署时标记高风险待确认。
- **无回滚或回滚破坏数据**：down 脚本删除新数据、不可逆却未声明、强制回滚可能丢失生产数据。
  - 怎么检查：读取 down/rollback 逻辑，确认是否安全、可执行、或明确标记不可逆并有 forward fix 方案。

### 高风险

- **新增 NOT NULL 字段不安全**：已有表新增非空字段无默认值、无分批回填或无后续约束步骤。
  - 怎么检查：搜索 `ADD COLUMN`、`NOT NULL`、默认值和回填脚本。
- **索引创建方式不适合生产**：PostgreSQL 未用 `CONCURRENTLY`，MySQL 未说明 online DDL，索引字段与查询不匹配。
  - 怎么检查：查看索引语句、where/order/join 字段、迁移工具事务限制。
- **数据回填无分批和进度控制**：一次性更新全表，无 batch、limit、游标、断点续跑、重试或日志。
  - 怎么检查：搜索 `UPDATE`、`INSERT INTO SELECT`、循环、batch_size、limit、skip locked。
- **多版本应用不兼容**：迁移和应用发布顺序要求过强，灰度/回滚期间旧新版本不能同时工作。
  - 怎么检查：确认读写路径是否兼容新旧字段，是否有双写、读 fallback 和验证阶段。
- **迁移依赖当前 schema 类型**：ORM migration 依赖当前业务模型类型，未来 schema 变化会导致历史迁移不可重放。
  - 怎么检查：Prisma/Drizzle/Kysely/Django/TypeORM 迁移是否使用冻结 schema 或 apps.get_model 等安全方式。
- **缺少备份和验证计划**：生产迁移无备份、无行数校验、无一致性校验、无回滚演练。
  - 怎么检查：查看发布说明、迁移注释、校验 SQL、监控和回滚步骤。

### 中风险

- **迁移命名和顺序不清**：文件名不表达目的，依赖关系不明确，可能乱序执行。
  - 怎么检查：查看 migration 命名、版本号、依赖声明和工具排序规则。
- **约束和索引命名不稳定**：自动生成名称难维护，跨环境不一致。
  - 怎么检查：查看 constraint/index 名称是否显式、可读、可追踪。
- **审计字段或默认值不一致**：新增表缺 created_at/updated_at、软删除、租户字段或业务唯一约束。
  - 怎么检查：对比同库同类表结构和项目约定。
- **脏状态处理不明确**：迁移失败后如何恢复版本、force、重跑、幂等未说明。
  - 怎么检查：查看迁移工具状态表、失败处理说明和脚本幂等性。
- **测试环境数据规模过小**：只在空库或小数据量验证，无法暴露锁表和耗时风险。
  - 怎么检查：查看测试说明、预估行数、执行计划或压测/演练记录。

### 建议优化

- **使用 expand-contract 模式**：添加新结构、双写/回填、读新结构、再删除旧结构。
- **拆分迁移 PR/发布步骤**：schema、应用兼容、数据回填、清理分批上线。
- **添加校验脚本**：迁移前后行数、空值、唯一性、差异校验。
- **文档化不可逆迁移**：说明为什么不可逆、如何 forward fix、如何恢复备份。

## 常用搜索线索

按当前平台工具执行搜索，结果必须包含文件路径、行号和具体命中内容。

```text
ALTER TABLE|DROP COLUMN|RENAME COLUMN|CHANGE COLUMN|MODIFY COLUMN
ADD COLUMN|NOT NULL|DEFAULT|CREATE INDEX|CONCURRENTLY|ONLINE
UPDATE .* SET|INSERT INTO .* SELECT|DELETE FROM|TRUNCATE
batch|batch_size|LIMIT|SKIP LOCKED|cursor|chunk
rollback|down\\(|reverse|RunPython|SeparateDatabaseAndState
migrate|migration|schema|prisma|drizzle|kysely|typeorm|golang-migrate
created_at|updated_at|deleted_at|tenant_id|unique|foreign key
```

## 判断标准

- 阻断：可能导致生产锁表、数据丢失、不可回滚、环境漂移或多版本应用不可用。
- 高风险：迁移可执行但大数据、灰度、失败重试、索引或回填存在明显事故风险。
- 中风险：命名、约束、审计、脏状态和测试规模存在维护或发布隐患。
- 建议优化：发布步骤、校验脚本、文档和演练可进一步提升。

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
  - 位置：path/to/migration.sql:line
  - 影响：锁表、数据、兼容发布或回滚风险
  - 原因：为什么当前迁移不安全
  - 建议：如何拆分、回滚、分批或验证
```

没有发现对应等级问题时写“未发现”。无法确认的问题要说明原因和下一步验证建议。

## 内置规则覆盖

- Expand-contract：新增可空列、新表、双写、回填、切读、删除旧字段必须分阶段，避免同版本直接重命名或删除线上字段。
- 大表 DDL：检查锁表、在线 DDL 能力、索引创建方式、超时、批大小、回滚窗口和业务低峰执行安排。
- 数据回填：必须具备幂等、分批、断点续跑、失败重试、校验 SQL、审计记录和异常停止条件。
- 回滚方案：schema、数据、应用版本、灰度开关和读写路径都要有回滚或暂停策略。
- 兼容发布：迁移前后旧代码与新代码都应可运行，避免枚举、默认值、非空约束和唯一索引破坏历史数据。
- 工具特定风险：检查迁移状态表、dirty state、Prisma client 生成、Drizzle/Kysely SQL 输出、Django `apps.get_model` 和历史迁移可重放性。

## 多平台兼容

- Codex：优先使用 `rg`、`rg --files`、`Get-Content -Encoding UTF8`，复杂任务使用 `update_plan`。
- Claude：优先使用 `Read`、`Grep`、`Glob`，复杂任务使用可用的 Todo 工具。
- Trae：优先使用 `SearchCodebase`、`Read`，复杂任务使用可用的任务清单工具。
- 任一平台都必须遵守：不默认执行 Git、不自动修复、不批量改文件、中文内容保持 UTF-8。
