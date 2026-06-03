---
name: "test-quality-pattern-vue"
description: "Use when reviewing Vue component architecture, composables, Pinia, forms, routing, rendering, or Vue project patterns."
---
# Vue 模式质量审查

## 定位

用于审查 Vue 3、TypeScript、Composition API、Pinia、Vue Router、表单和组件架构相关代码质量。只做静态代码层审查，重点发现会影响可维护性、可靠性、安全性、性能和可测试性的实现问题。

## 内置规则覆盖

- 组件职责：检查巨型组件、跨层依赖、模板逻辑过重、重复 UI 逻辑和业务状态混杂。
- Composable：检查 `useXxx` 命名、返回接口、请求副作用、监听器、定时器、生命周期清理和可测试性。
- Pinia：检查领域拆分、临时状态误入全局 store、action 并发、错误处理、持久化和退出登录重置。
- API 封装：检查请求拦截、错误归一、取消重复请求、分页参数、loading 状态和服务端校验结果处理。
- 表单与路由：检查校验触发、防重复提交、异步校验、权限守卫、动态参数、开放跳转和 404/403 兜底。

## 检查重点

- **Vue 组件架构**：组件职责是否单一，模板、状态、计算属性、方法、生命周期和样式是否清晰分层；是否存在巨型组件、跨层依赖、重复 UI 逻辑或难以复用的业务耦合。
- **Composition API 组合逻辑**：`composables` 是否以 `useXxx` 命名并返回清晰接口；是否把副作用、请求、监听器、定时器和生命周期清理封装完整；是否滥用全局状态或把 UI 细节塞进通用组合函数。
- **Props / Emit**：`defineProps`、`withDefaults`、`defineEmits` 是否类型明确；是否直接修改 props；双向绑定是否使用 `update:modelValue` 或明确事件；事件命名、载荷和父子职责是否一致。
- **Pinia**：store 是否按领域拆分；state、getter、action 边界是否清晰；是否把临时组件状态放入全局 store；认证状态、持久化、重置、错误处理和并发请求是否可靠。
- **表单**：表单 model、rules、校验触发、提交防抖/防重复、重置逻辑、异步校验和错误展示是否完整；是否在提交前执行客户端校验并处理服务端校验结果。
- **路由**：路由懒加载、权限守卫、元信息、动态参数、重定向、404/403、登录跳转和页面级数据加载是否清晰；是否存在未验证的跳转地址或循环导航风险。
- **XSS 与前端安全**：重点检查 `v-html`、`innerHTML`、动态 URL、第三方富文本、用户输入回显、开放重定向、token 存储和权限判断；用户可控内容必须转义、净化或白名单处理。
- **性能渲染**：列表渲染 `key`、大列表虚拟化、昂贵 computed/watch、深度监听、重复请求、组件切分、异步组件、缓存策略、图片资源和不必要重渲染。
- **可测试性**：业务逻辑是否可从组件中抽出测试；store、composable、表单校验、路由守卫和关键交互是否可单元测试或组件测试；是否存在强耦合浏览器全局对象导致难以 mock。

## 检查流程

1. 确认项目技术栈：Vue 版本、TypeScript、Vite/Webpack、Pinia/Vuex、Vue Router、UI 组件库、测试框架。
2. 读取入口配置和目录结构：`package.json`、`src`、`router`、`stores`、`composables`、`views`、`components`、`api`、`tests` 或 `__tests__`。
3. 搜索高风险模式：安全、路由、store、表单、渲染性能、watch 副作用、props 修改、未清理资源。
4. 抽样读取关键文件：优先业务核心页面、复用组件、全局 store、路由守卫、表单提交链路和公共 composable。
5. 对照分级标准输出问题。每个问题必须包含文件路径、行号、等级、原因、影响和可执行建议。
6. 若证据不足，不做断言；说明缺少的上下文和下一步验证方式。

## 分级标准

### 阻断

- 用户可控内容未经可信净化直接进入 `v-html`、`innerHTML`、动态脚本、危险 URL 或富文本渲染，存在实际 XSS 利用路径。
- 路由守卫或前端权限实现导致未授权页面、数据或管理操作可直接访问，且缺少后端兜底迹象。
- 关键表单提交缺少必要校验、重复提交保护或错误处理，可能造成资金、订单、权限、账号等关键业务错误。
- 构建入口、路由入口、全局 store 初始化或核心页面存在明显运行时崩溃路径，主流程不可用。
- 审查目标缺少关键文件或无法读取核心上下文，导致无法给出可信结论。

### 高风险

- 组件职责严重膨胀，单个组件同时承担请求、权限、表单、复杂状态机和多块 UI，导致缺陷难定位、难复用、难测试。
- composable 内部副作用不可控，存在未清理监听器、定时器、订阅、请求竞态或跨组件共享可变状态。
- props 被子组件直接修改，或 emit 载荷无类型约束导致父子状态不同步。
- Pinia store 混入大量页面临时状态、未提供重置逻辑，登录切换、租户切换或退出后可能泄漏旧状态。
- 路由动态参数、重定向参数或返回地址未校验，可能产生开放重定向、越权导航或错误数据加载。
- 大列表、深度 watch、模板内重计算或无 key 渲染会在常见数据量下造成明显卡顿。

### 中风险

- 组件拆分、命名、目录归属或导入顺序混乱，增加维护成本但暂未看到直接业务故障。
- `watch`、`computed`、`ref/reactive` 使用不当，存在重复请求、依赖不清或边界条件错误风险。
- 表单校验规则分散、异步校验缺少取消或提交状态管理不一致。
- store action 错误处理不统一，loading、error、empty 状态缺失。
- 路由 meta、权限常量、菜单权限和页面权限存在重复定义或弱一致性。
- 测试覆盖缺失关键 composable、store、表单提交或路由守卫。

### 建议优化

- 可抽取复用组件、composable、表单规则、权限判断或 API 调用封装。
- 可补充类型收窄、显式返回类型、事件命名规范、默认值和空状态。
- 可使用异步组件、`defineAsyncComponent`、路由懒加载、缓存或虚拟列表优化体验。
- 可补充组件测试、store 单元测试、composable 测试和关键路由守卫测试。

## 怎么检查

- 使用项目已有 lint、类型和测试配置作为辅助依据；没有用户要求时不要自动修复。
- 阅读代码时优先追完整链路：路由入口 -> 页面组件 -> composable/store -> API -> 表单/渲染输出。
- 对 `script setup` 组件检查导入、props、emits、状态、computed、watch、方法、生命周期、暴露接口和模板绑定是否一致。
- 对 composable 检查入参、返回值、生命周期、副作用清理、错误处理、并发控制和是否可独立测试。
- 对 Pinia 检查 store 拆分、持久化、敏感信息、reset、action 事务性、异步状态和 SSR/多标签页影响。
- 对表单检查 model 与 rules 对齐、字段命名、必填/格式/范围/异步校验、提交 loading、重复提交、取消和重置。
- 对路由检查路由表、守卫、meta、权限、动态参数、懒加载、异常页面和跳转来源校验。
- 对 XSS 检查用户输入进入 DOM、URL、富文本、属性绑定和第三方组件的路径；只在确认数据可信或已净化时降低等级。
- 对性能检查渲染列表、key、watch 深度、模板表达式复杂度、computed 缓存、请求触发点和组件重渲染来源。
- 对可测试性检查逻辑是否可脱离 DOM 测试，依赖是否可注入或 mock，关键路径是否有测试文件。

## 搜索线索

Codex 优先使用 `rg` / `rg --files`，搜索结果需保留文件路径、行号和代码片段。其他平台按下方兼容方式替换工具。

- 组件和页面：`rg -n "<script setup|defineComponent|defineProps|defineEmits|withDefaults" src`
- 组合逻辑：`rg -n "export function use[A-Z]|function use[A-Z]|const use[A-Z]" src`
- Pinia：`rg -n "defineStore|storeToRefs|pinia|persist" src`
- 表单：`rg -n "rules|validate\\(|resetFields|clearValidate|v-model|modelValue|update:modelValue" src`
- 路由：`rg -n "createRouter|beforeEach|beforeResolve|afterEach|meta:|redirect|router\\.push|router\\.replace|useRoute|useRouter" src`
- XSS：`rg -n "v-html|innerHTML|outerHTML|insertAdjacentHTML|DOMPurify|sanitize|javascript:|window\\.open|location\\.href" src`
- 性能：`rg -n "v-for|:key|watch\\(|watchEffect|deep:\\s*true|computed\\(|keep-alive|defineAsyncComponent" src`
- 可测试性：`rg --files | rg "(test|spec)\\.(ts|tsx|js|jsx|vue)$|__tests__"`
- 类型和边界：`rg -n "any\\b|as any|// @ts-ignore|// @ts-expect-error|unknown" src`
- 资源清理：`rg -n "addEventListener|removeEventListener|setInterval|clearInterval|setTimeout|AbortController|onUnmounted|onScopeDispose" src`

## 多平台兼容

- **Codex**：搜索优先 `rg`、`rg --files`；读取具体文件后输出路径和行号；无明确要求不要执行任何 Git 命令；不要自动修改代码。
- **Claude**：搜索优先 `Grep`、`Glob`、`Read`；输出时保留文件路径、行号和证据片段；不要启用后台观察、hooks 或自动修复流程。
- **Trae**：搜索优先 `SearchCodebase`、`Read`；复杂范围先列影响文件；输出审查结论，不默认改代码。
- 任一平台遇到中文路径或中文内容时，读取和写入都按 UTF-8 处理；若出现乱码，停止基于该输出判断。

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

按以下结构输出，优先列问题，不要把摘要放在问题前：

```markdown
## 阻断
- [等级] 文件路径:行号 - 问题标题
  原因：说明证据和触发条件。
  影响：说明可能造成的用户、业务、安全、性能或维护后果。
  建议：给出可执行修改方向。

## 高风险
- ...

## 中风险
- ...

## 建议优化
- ...

## 无法确认 / 需要补充验证
- 文件路径:行号 - 缺少的上下文和建议验证方式。

## 检查范围
- 已查看的目录、文件类型、关键搜索词、未覆盖范围。
```

如果没有发现某一等级问题，明确写“未发现”。报告中必须避免虚构文件、行号或未验证事实。
