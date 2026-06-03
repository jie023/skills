---
name: "test-quality-language-typescript-review"
description: "Use when reviewing TypeScript or JavaScript code quality, strict type safety, any/unknown usage, type narrowing, Zod/DTO validation, async error handling, XSS, frontend/backend boundaries, package dependencies, or tests."
---

# TypeScript/JavaScript 专项质量审查

## 定位

用于 TypeScript/JavaScript 代码质量审查，重点覆盖类型安全、运行时边界校验、异步正确性、Web/Node 安全、依赖风险和测试充分性。

## 内置规则覆盖

- 类型安全：`strict`、`noImplicitAny`、`strictNullChecks` 不应被弱化；公共 API、DTO、SDK 返回值不能退化为 `any`。
- 运行时边界：`req.body`、URL 参数、FormData、localStorage、env、JSON.parse、第三方 API 和数据库返回值必须 schema 校验或类型收窄。
- Web/Node 安全：XSS、RCE、SQL/NoSQL 注入、路径穿越、命令执行、SSRF、敏感环境变量暴露都按高优先级检查。
- 异步正确性：floating promise、空 catch、非 Error 抛出、未取消请求、事务半提交和资源泄漏必须定位。
- 依赖风险：新增包、install script、peer dependency、锁文件不一致、前端产物 sourcemap/mock/调试开关都要检查。

## 执行流程

1. 明确审查范围：优先使用用户指定文件、目录、差异片段或问题描述；没有明确范围时，先搜索 package、tsconfig、src、routes、controllers、pages、app、components、tests 等关键位置。
2. 识别项目上下文：确认运行时 Node/Deno/Bun/Browser、框架 React/Next/Nest/Express/Vite、包管理器 npm/pnpm/yarn/bun、测试框架 Jest/Vitest/Playwright。
3. 读取配置：检查 `package.json`、`tsconfig*.json`、`eslint` 配置、测试配置、环境变量校验入口和边界 schema 目录。
4. 静态审查代码：围绕外部输入、DTO/schema、类型断言、异步调用、HTML 渲染、服务端/客户端边界、依赖变更和测试文件读取上下文。
5. 可执行检查：仅在用户允许且项目已有脚本时运行 typecheck、lint、test、audit；不默认安装依赖、不默认访问网络、不默认发起 API 请求。
6. 分级输出：按阻断、高风险、中风险、建议优化分组；每个问题必须给出文件路径、行号、证据、影响、修复建议和验证方式。
7. 不自动修复：默认只报告问题。若用户要求修改代码，再按任务范围逐项处理。

## 分级检查项

### 阻断

- strict 被关闭或弱化：检查 `tsconfig*.json` 是否禁用 `strict`、`noImplicitAny`、`strictNullChecks`，或在变更中降低类型检查强度。会让核心类型保护失效时阻断。
- 外部输入未校验即进入关键路径：检查 `req.body`、`req.query`、`params`、`FormData`、`localStorage`、`process.env`、第三方 API、消息队列、文件和数据库返回值。涉及鉴权、金额、权限、持久化、命令、文件路径、HTML 输出时必须有 Zod/DTO 或等价 schema 校验。
- XSS/RCE/注入风险：检查 `innerHTML`、`dangerouslySetInnerHTML`、`document.write`、`eval`、`new Function`、SQL/NoSQL 字符串拼接、`child_process`、`fs` 路径拼接。用户可控数据未清洗、未参数化或未白名单校验时阻断。
- 敏感信息泄露：检查硬编码 token、密钥、密码、私有 URL，以及前端包中误暴露服务端环境变量或内部错误详情。
- 关键异步错误无人处理：检查会导致请求静默成功、事务半提交、资源泄漏或进程崩溃的 floating promise、空 `catch`、非 `Error` 抛出和未处理 rejection。
- 依赖存在明确高危风险：检查新增包、版本范围、install script、锁文件一致性和审计结果；关键路径引入已知高危漏洞或供应链风险时阻断。

### 高风险

- `any` 滥用：检查 `: any`、`as any`、`Record<string, any>`、泛型默认 `any`、公共 API 返回 `any`。优先建议改为精确类型或 `unknown` 后收窄。
- `unknown` 未收窄：检查 `unknown`、`JSON.parse`、`response.json()`、catch error、第三方 SDK 返回值是否经过 `typeof`、`in`、判别联合、schema safeParse 或自定义 type guard。
- 类型断言绕过检查：检查 `as SomeType`、双重断言 `as unknown as`、非空断言 `!`、`@ts-ignore`、`@ts-expect-error`。没有前置 guard 或解释时按高风险处理。
- DTO/schema 漂移：检查 Zod/Joi/Yup/class-validator schema 与 DTO/type/interface 是否重复维护且字段不一致；优先用 `z.infer` 或单一来源生成类型。
- 异步流程不正确：检查 `forEach(async ...)`、循环内串行 await、未 await 的 promise、事件处理器 fire-and-forget、`Promise.all` 局部失败处理、超时/取消缺失。
- 前后端边界混淆：检查 Next/SSR client component 引入 server-only 模块、浏览器代码读取服务端 secret、后端直接信任前端类型、接口错误对象直接透传。
- 包依赖破坏可维护性：检查新增重量级依赖、重复功能库、未固定主版本、锁文件缺失、CJS/ESM 混用导致运行时不兼容。

### 中风险

- public 函数和导出 API 缺少显式返回类型，导致隐式 any 或破坏调用方契约。
- 对可空值、索引访问、深层 optional chaining 缺少 fallback，存在运行时 `undefined` 风险。
- React/Next 组件存在 effect 依赖缺失、直接修改 state、动态列表使用 index key、服务端/客户端数据获取职责不清。
- React ErrorBoundary、模块级可变共享状态、render 内联对象/数组/函数导致重复渲染或状态污染。
- Node 请求处理器使用同步 fs、CPU 密集逻辑或 N+1 API/数据库调用，影响事件循环或响应时间。
- 错误处理信息不足：日志缺少上下文、吞错后返回默认值、业务错误和系统错误没有区分。
- 测试覆盖不足：缺少 schema 边界值、非法输入、异步失败、权限/XSS、依赖 mock、回归用例。

### 建议优化

- 使用 `const` 优先、`readonly`、`ReadonlyArray<T>`、判别联合、`satisfies`、`as const` 提升类型表达力。
- 使用参数化查询、HTML sanitizer、路径 `resolve` 后前缀校验、环境变量启动期 schema 校验强化边界。
- 使用命名常量替代魔法值，统一 logger，移除生产 `console.log`。
- 使用 tree-shake 友好的命名导入，避免整包导入大型工具库。
- 在 JS 项目中评估 `checkJs`、JSDoc 类型、ESLint type-aware rules 和渐进式 TS 迁移。

## 怎么检查

- strict 配置：读取 `tsconfig*.json`，确认 `strict`、`noImplicitAny`、`strictNullChecks`、`noUncheckedIndexedAccess`、`exactOptionalPropertyTypes` 是否符合项目风险等级；若配置被修改，重点看是否降级。
- `any/unknown`：搜索所有 TS/JS 源码和类型声明，读上下文判断是否位于边界、公共 API、DTO、服务层或测试 mock；测试中可接受的 `any` 也要确认是否隔离。
- 类型收窄：对每个 `as`、`!`、ignore 注释、`JSON.parse`、`response.json()` 和 catch error，确认是否有运行时校验、type guard、判别字段或 schema。
- Zod/DTO 校验：从路由、controller、server action、API client、表单提交、队列消费者和环境变量入口向内追踪，确认外部数据先校验再进入业务逻辑。
- 异步错误：检查每个 promise 创建点是否 await、return、catch 或集中托管；检查事务、批处理、并发请求和资源清理是否处理失败路径。
- XSS/注入：沿用户输入到 HTML、SQL、NoSQL、shell、文件路径、URL redirect 的数据流检查，确认有 sanitizer、参数化、白名单或路径边界校验。
- 前后端边界：检查前端包不能引入服务端模块和 secret，后端不能信任前端类型声明，接口响应不能暴露内部堆栈或敏感字段。
- React/Next 渲染质量：检查 ErrorBoundary 覆盖关键区域，模块级变量是否跨请求/跨用户共享，render 中 inline object/array/callback 是否应使用 memo、useMemo、useCallback 或组件拆分。
- 依赖：读取 `package.json` 和锁文件，确认新增包用途、版本范围、模块格式、维护状态、漏洞风险和是否已有项目工具可替代。
- 测试：读取 `*.test.*`、`*.spec.*`、`__tests__`、e2e 目录，确认关键风险有正反例、失败路径、边界值和安全回归测试。

## 常用搜索线索

优先用平台原生搜索工具，搜索结果要带文件路径、行号和具体代码内容。

- 配置与脚本：`rg -n "\"strict\"|\"noImplicitAny\"|\"strictNullChecks\"|\"typecheck\"|\"eslint\"|\"vitest\"|\"jest\"" package.json tsconfig*.json`
- `any/unknown`：`rg -n "\\bany\\b|as any|Record<string, any>|\\bunknown\\b|as unknown as" -g "*.ts" -g "*.tsx" -g "*.js" -g "*.jsx"`
- 类型绕过：`rg -n "@ts-ignore|@ts-expect-error|as [A-Za-z0-9_<>]+|[A-Za-z0-9_\\]\\)]!" -g "*.ts" -g "*.tsx"`
- 边界输入：`rg -n "req\\.body|req\\.query|req\\.params|FormData|localStorage|process\\.env|JSON\\.parse|response\\.json\\(" -g "*.ts" -g "*.tsx" -g "*.js" -g "*.jsx"`
- Zod/DTO：`rg -n "z\\.object|safeParse|parse\\(|class-validator|DTO|Dto|schema" -g "*.ts" -g "*.tsx"`
- 异步错误：`rg -n "forEach\\(async|new Promise|Promise\\.all|Promise\\.allSettled|\\.catch\\(|catch \\(|void .*\\(|await .*\\.map" -g "*.ts" -g "*.tsx" -g "*.js" -g "*.jsx"`
- XSS/注入：`rg -n "dangerouslySetInnerHTML|innerHTML|document\\.write|eval\\(|new Function|child_process|exec\\(|spawn\\(|query\\(|raw\\(|path\\.join|fs\\." -g "*.ts" -g "*.tsx" -g "*.js" -g "*.jsx"`
- 前后端边界：`rg -n "use client|server-only|next/headers|NEXT_PUBLIC_|process\\.env|window\\.|document\\." -g "*.ts" -g "*.tsx" -g "*.js" -g "*.jsx"`
- React 渲染：`rg -n "ErrorBoundary|useMemo|useCallback|React\\.memo|const .*\\[\\]|const .*\\{\\}|let .* =" -g "*.tsx" -g "*.jsx"`
- 测试：`rg --files | rg "(test|spec|__tests__|e2e).*(ts|tsx|js|jsx)$"`

## 多平台兼容

- Codex：优先 `rg`、`rg --files`，再读取文件；不要执行 Git，除非用户明确要求。
- Claude：优先 `Grep`、`Glob`、`Read`；不要依赖 Codex 专用工具名。
- Trae：优先 `SearchCodebase`、`Read`；保持路径和行号可追溯。
- Windows 路径建议使用 `/`，读写中文路径和中文内容保持 UTF-8。
- 不依赖外部目录、远程资料或平台私有斜杠命令；审查规则以内置检查项为准。

## 输出格式

```markdown
## TypeScript/JavaScript 专项审查报告

### 结论
- 结果：通过 / 有建议 / 有中风险 / 有高风险 / 阻断
- 范围：审查的文件、目录或变更说明
- 已读取资料：列出读取的本地项目配置、关键代码和测试文件
- 已执行检查：typecheck/lint/test/audit 或说明未执行原因

### 阻断
1. [阻断] 问题标题
   - 位置：path/to/file.ts:行号
   - 证据：引用最小必要代码事实
   - 影响：说明为什么会造成阻断
   - 建议：给出可执行修复方向
   - 验证：修复后如何验证

### 高风险
- 无 / 同上格式列出

### 中风险
- 无 / 同上格式列出

### 建议优化
- 无 / 同上格式列出

### 残余风险
- 无法确认的点、需要用户补充的上下文或建议后续执行的检查
```

没有发现问题时也要明确写“未发现阻断/高风险/中风险问题”，并说明未执行的检查和残余风险。
