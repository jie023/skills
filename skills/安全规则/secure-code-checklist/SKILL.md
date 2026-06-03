---
name: "secure-code-checklist"
description: "Use when writing, modifying, or reviewing code that handles user input, SQL, command execution, paths, templates, authentication, authorization, sensitive data, logging, frontend output, or data export."
---
# 安全代码检查清单

## 使用场景

- 代码处理用户输入、SQL、命令、路径或模板
- 代码涉及登录、鉴权、权限变更、数据导出或敏感数据
- 需要检查日志脱敏、XSS、富文本、Token、密码、密钥等风险
- 做接口、后端、前端输出或数据访问层安全评审

## 与已有 skill 的关系

- 本 skill 是通用安全检查清单。
- Java/P3C 安全细则优先配合 `p3c-security-rules`。
- API 安全测试优先配合 `test-api-security`。
- MySQL/SQL 安全优先配合 `p3c-mysql-database`。

## 核心检查

- 用户输入必须按业务场景校验，禁止直接拼接进 SQL、命令、路径或模板。
- 数据库操作必须使用参数化查询或 ORM 安全 API，禁止字符串拼接 SQL。
- 命令执行必须避免拼接用户输入；确需传参时使用参数数组或安全转义。
- 敏感数据禁止硬编码，包括密码、密钥、Token、身份证号、手机号等。
- 日志输出必须脱敏，不记录完整密码、密钥、Token、身份证号、手机号等敏感值。
- 输出到前端的数据必须按上下文转义；富文本内容必须使用白名单净化。
- 涉及认证、授权、权限变更、数据导出等操作时，必须进行身份校验和权限校验。

## 审查流程

1. 找出输入来源：请求参数、文件、数据库、第三方回调、环境变量。
2. 找出危险流向：SQL、命令、路径、模板、日志、前端输出、导出文件。
3. 检查是否使用安全 API：参数化查询、参数数组、框架转义、白名单净化。
4. 检查敏感数据是否硬编码、明文传输、明文日志或未脱敏返回。
5. 检查认证授权是否覆盖对象级权限和操作级权限。
6. 能运行安全相关测试时优先运行；无法运行时说明剩余风险。

## 多平台执行方式

- Claude：搜索优先 `Grep` / `Glob` / `Read`，修改优先 `Edit`。
- Codex：搜索优先 `rg`，修改优先 `apply_patch`。
- Trae：搜索优先 `SearchCodebase` / `Read`，修改优先平台编辑工具。

## 禁止事项

- 禁止用字符串拼接 SQL。
- 禁止把用户输入直接拼进 shell 命令。
- 禁止把敏感信息写死在代码、配置示例或测试数据中。
- 禁止将完整敏感值输出到日志或接口响应。
- 禁止把“前端会处理”作为后端权限校验缺失的理由。
