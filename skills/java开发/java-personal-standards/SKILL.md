---
name: "java-personal-standards"
description: "Use when writing, modifying, reviewing, or generating Java code under the user's personal Java rules, especially when Chinese comments, JavaDoc, author tags, P3C style, tests, or project-local utility reuse matter."
---
# Java 个人代码规范

## 使用场景

- 编写、修改或评审 Java 代码
- 生成 Controller、Service、DAO、DO、VO、DTO 等 Java 文件
- 需要中文注释、JavaDoc、`@author` 或 P3C 风格
- 修改复杂逻辑，需要同步考虑单元测试和验证

## 与已有 skill 的关系

- 本 skill 是个人补充规则，不替代已有 Java/P3C 专项 skill。
- 涉及具体分层代码时，同时参考对应专项 skill：`java-controller`、`java-service`、`java-dao`、`java-do`、`java-vo`。
- 涉及 P3C 细则时，同时参考：`p3c-coding-style`、`p3c-exception-logging`、`p3c-oop-standards`、`p3c-unit-testing`。

## 注释规则

- 所有代码注释必须使用中文。
- public 类、public 方法、复杂逻辑必须编写清晰注释。
- Java public 类和 public 方法优先使用中文 JavaDoc。
- 方法注释包含功能描述和 `@param`；有返回值时写 `@return`；显式抛出异常时写 `@throws`。
- 类文件头部必须包含 `@author`，作者名以全局规则中的“作者”为准。
- 简单私有方法和自解释代码可简写或不写注释。

## 代码风格

- 优先遵循项目现有规范。
- 遵循阿里巴巴 P3C 编码规范。
- 优先使用项目已有工具类、公共方法和本地 helper，避免重复造轮子。
- 不做与当前任务无关的重构、格式化或命名调整。

## 测试与验证

- 新增或修改复杂逻辑时，必须同步编写或更新单元测试。
- 修改完成后优先运行相关测试、构建或静态检查。
- 无法运行验证时，必须说明原因、风险和已完成的替代复核。

## 多平台执行方式

- Claude：代码搜索优先 `Grep` / `Glob` / `Read`，修改优先 `Edit` / `Write`。
- Codex：代码搜索优先 `rg` / `rg --files`，修改优先 `apply_patch`。
- Trae：代码搜索优先 `SearchCodebase` / `Read`，修改优先平台编辑工具。

## 禁止事项

- 禁止生成英文注释替代中文注释。
- 禁止给 `void` 方法强行写无意义 `@return`。
- 禁止给未显式抛出异常的方法强行写空泛 `@throws`。
- 禁止忽略项目已有工具类而重复实现相同逻辑。
