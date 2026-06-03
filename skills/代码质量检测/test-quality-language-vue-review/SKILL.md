---
name: "test-quality-language-vue-review"
description: "Use when reviewing Vue 3 code quality, Composition API, props typing, forms, Pinia, XSS, component structure, or rendering performance."
---
# Vue 语言专项质量审查

## 定位

Vue 专项代码质量审查，覆盖 Vue 3、TypeScript 和组件质量。

## 内置规则覆盖

- Vue 3 规范：组件结构、Composition API、props/emit、slot、composable、Pinia、router 和 API 封装都纳入审查。
- 类型质量：`defineProps`、`defineEmits`、ref/reactive、computed、watch、表单模型和 API 响应要有明确类型。
- 安全风险：`v-html`、动态 URL、上传预览、富文本、第三方内容和浏览器存储要检查 XSS 与敏感信息泄露。
- 性能与渲染：列表 key、computed/watch、深度监听、重复请求、无效重渲染和组件拆分要结合场景判断。

## 检查流程

1. 确认检查范围：`.vue`、`.ts`、`stores/`、`composables/`、`api/`、`router/` 中与本次模块相关的文件。
2. 优先读取组件入口和被引用的 composable、store、API 请求文件。
3. 按“阻断 -> 高风险 -> 中风险 -> 建议优化”的顺序检查。
4. 每个问题必须落到具体文件和行号；无法确认时说明缺少哪些上下文。
5. 不运行 API 请求；如需运行构建、类型检查或测试，必须先说明命令和目的。

## 具体检查项

### 阻断问题

| 检查项 | 怎么检查 | 典型问题 |
| ------ | -------- | -------- |
| XSS 风险 | 搜索 `v-html`、`innerHTML`、富文本渲染、用户输入直接输出 | 未白名单净化直接渲染 HTML |
| 敏感信息泄露 | 检查 token、密码、手机号、身份证、密钥是否进入前端常量、日志、URL、localStorage | `console.log(token)`、硬编码密钥 |
| 输入未校验 | 检查表单字段是否有 `rules`、提交前是否调用 `validate()` | 未校验直接提交接口 |
| 权限/路由绕过 | 检查路由守卫、按钮权限、页面权限是否只靠前端隐藏 | 只隐藏按钮但接口仍可调用 |

### 高风险问题

| 检查项 | 怎么检查 | 典型问题 |
| ------ | -------- | -------- |
| Props 类型不完整 | 检查 `defineProps` 是否使用接口类型，必填、默认值是否清楚 | Props 无类型、可选值无默认值 |
| Emits 定义不完整 | 检查 `defineEmits` 是否声明事件名和参数类型 | 随意 `emit('update', data)` |
| 直接修改 props | 搜索 `props.xxx =`、对 props 内对象直接赋值 | 破坏单向数据流 |
| 没有使用 Composition API | 检查是否使用 Vue 2 Options API 或混合写法 | 新代码仍使用 `data/methods` |
| 组件过大 | 统计单文件行数和职责数量 | 单组件超过 800 行或同时处理多个业务 |
| 函数过长/嵌套过深 | 检查方法是否超过 50 行、条件嵌套是否超过 4 层 | 提交方法混合校验、组装、请求、跳转 |
| API 封装不规范 | 检查 axios/fetch baseURL、timeout、token interceptor、刷新 token、错误归一和取消重复请求 | 请求配置散落在组件里、无超时或无统一鉴权注入 |
| API 错误处理缺失 | 检查请求是否有 `try/catch`、loading 复位、错误提示 | 请求失败后页面无反馈 |

### 中风险问题

| 检查项 | 怎么检查 | 典型问题 |
| ------ | -------- | -------- |
| composable 未抽取 | 检查多个组件是否重复同一段状态、请求、表单逻辑 | 复制粘贴加载列表逻辑 |
| composable 暴露可变状态 | 检查 composable 返回值是否应使用 `readonly` 或只暴露更新方法 | 外部组件任意改写内部状态 |
| Pinia 使用不规范 | 检查 store 是否只放共享状态，action 是否清晰 | 临时页面状态塞进全局 store |
| TypeScript 类型弱 | 搜索 `any`、接口缺失、API 响应无类型 | `const data: any = ...` |
| 不可变性不足 | 检查数组、对象更新是否直接原地修改 | `users.value.push(item)` |
| computed 使用不足 | 检查模板或方法中重复计算的表达式 | 每次渲染重复 filter/map |
| 大列表未优化 | 检查长列表、表格、下拉是否分页或虚拟滚动 | 一次渲染上千行 |
| 图片和资源未优化 | 检查大图、懒加载、静态资源引用 | 列表中直接加载原图 |

### 建议优化

| 检查项 | 怎么检查 | 典型问题 |
| ------ | -------- | -------- |
| 文件结构不清晰 | 检查是否符合 `components/views/composables/stores/api/types` 分层 | 业务组件混在公共组件目录 |
| 组件命名不规范 | 检查组件是否 PascalCase，基础组件是否 `Base` / `App` 前缀 | `userList.vue` |
| import 顺序混乱 | 检查是否按类型、Vue、第三方、本地模块分组 | import 随机排列 |
| 生命周期位置混乱 | 检查 `<script setup>` 中 props、emits、state、computed、methods、hooks 顺序 | 生命周期穿插在业务方法中 |
| 测试缺口 | 检查 Vitest / Vue Test Utils 是否覆盖关键渲染、交互、无效输入、校验失败和请求失败 | 只有正常加载测试 |

## 常用搜索方式

```text
v-html
innerHTML
defineProps
defineEmits
props.
any
rules
validate()
useXxxStore
defineStore
axios
baseURL
timeout
interceptor
readonly
console.log
localStorage
sessionStorage
```

## 判断示例

### 直接修改 props

```vue
// 问题：直接修改 props，破坏单向数据流
props.user.name = 'new name'
```

建议改为本地副本或通过 `emit` 通知父组件：

```vue
const localUser = ref({ ...props.user })
localUser.value.name = 'new name'
emit('update', localUser.value)
```

### 表单未校验

```typescript
async function handleSubmit() {
  await saveUser(form.value)
}
```

应检查是否存在表单规则并在提交前调用校验：

```typescript
const valid = await formRef.value?.validate()
if (!valid) {
  return
}
await saveUser(form.value)
```

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

## 输出要求

按阻断、高风险、中风险、建议优化分组。无法确认的问题要说明原因和下一步验证建议。

每个问题使用以下格式：

```text
[高风险] Direct prop mutation
文件：src/components/UserForm.vue:45
原因：直接修改 props 会破坏 Vue 单向数据流，导致父子状态不一致。
建议：使用本地副本或 emit 通知父组件更新。
```

## 内置规则覆盖

- Vue 3 规范：组件命名、单文件组件结构、Composition API、props/emit、slot、provide/inject 和 composable 边界都纳入审查。
- 状态与数据流：Pinia/store、父子通信、表单状态、异步请求、缓存和路由状态要保持单向、可追踪。
- 安全：`v-html`、动态组件、URL、富文本、上传预览、第三方内容和本地存储都要检查 XSS 与敏感信息泄露。
- 性能：大列表 key、computed/watch、深度 watcher、重复请求、无效重渲染、懒加载和组件拆分要结合场景判断。
- 测试：组件测试、表单校验、路由守卫、store 行为、错误态和权限态覆盖是审查依据。

## 多平台兼容

- Claude：优先使用 `Read`、`Grep`、`Glob` 读取项目代码。
- Codex：优先使用 `rg`、`rg --files` 搜索，再读取项目代码。
- Trae：优先使用 `SearchCodebase`、`Read` 读取项目代码。
- 不要求固定存在 `.claude`、`.codex` 或某个平台专属目录；审查规则以内置检查项为准。
