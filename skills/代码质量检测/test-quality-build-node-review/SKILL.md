---
name: "test-quality-build-node-review"
description: "Use when reviewing Node.js, npm, pnpm, yarn, TypeScript, Vite, Webpack, Babel, ESLint, CI, workspace, and frontend build quality issues."
---
# Node/TypeScript 构建质量审查

## 定位

面向 Node.js、TypeScript 和现代前端工程的构建质量审查 skill。用于静态审查构建配置、依赖关系、脚本约定、产物风险、CI 构建一致性和前端安全暴露问题。

## 适用范围

- `package.json`、`package-lock.json`、`npm-shrinkwrap.json`
- `pnpm-lock.yaml`、`pnpm-workspace.yaml`
- `yarn.lock`、`.yarnrc.yml`
- `.nvmrc`、`.node-version`、`.npmrc`、`volta`、`engines`
- `tsconfig*.json`、`vite.config.*`、`webpack.config.*`、`babel.config.*`
- `.eslintrc*`、`eslint.config.*`、`postcss.config.*`
- `.env*`、CI 配置、Dockerfile、构建脚本、monorepo workspace 配置

## 检查重点

- `package.json scripts`：确认 `build`、`typecheck`、`lint`、`test`、`preview`、`prepare`、`postinstall` 等脚本是否存在、命令是否可执行、是否依赖本机绝对路径或隐式全局命令。
- 包管理器与 lockfile：确认 npm、pnpm、yarn 是否混用；lockfile 是否与 `packageManager`、CI 安装命令、workspace 配置一致；是否存在多个 lockfile 导致依赖树不可复现。
- Node/npm 版本：检查 `engines`、`packageManager`、`.nvmrc`、`.node-version`、`volta`、CI Node 版本是否一致，避免本地和 CI 构建行为漂移。
- TypeScript 配置：检查 `tsconfig*.json` 的 `extends`、`include`、`exclude`、`paths`、`baseUrl`、`moduleResolution`、`strict`、`skipLibCheck`、`noEmit`、`declaration`、`composite` 是否符合项目构建方式。
- Vite/Webpack/Babel 配置：检查入口、别名、mode、target、sourcemap、publicPath/base、assetsDir、polyfill、tree shaking、chunk 拆分、Babel preset/plugin 与浏览器目标是否一致。
- ESLint 与类型检查：检查 lint 是否纳入 CI，ESLint parser、plugin、resolver 与 TypeScript、React/Vue 版本是否匹配，避免 lint 只在本地生效。
- 依赖冲突：检查 `dependencies`、`devDependencies`、`peerDependencies`、`overrides`、`resolutions` 是否存在重复、错位、过宽版本、peer 不满足、运行时依赖误放到 dev 依赖等问题。
- 构建产物：检查 `dist`、`build`、`lib`、`types`、`exports`、`files`、`main`、`module`、`types`、`sideEffects` 是否匹配发布或部署目标。
- 环境变量：检查 `.env*`、构建脚本、CI secrets、Vite `VITE_*`、Webpack DefinePlugin、Next/Nuxt 公开变量前缀，识别前端产物中可能暴露的密钥或内部地址。
- CI 构建：检查安装命令、缓存 key、Node 版本、lockfile 校验、构建矩阵、产物上传、失败策略是否与本地脚本一致。
- monorepo workspace：检查 `workspaces`、`pnpm-workspace.yaml`、包间依赖、构建顺序、`nohoist`、workspace 协议、根脚本与子包脚本是否一致。
- 前端安全构建暴露：检查 source map、调试开关、console、mock 接口、代理配置、内部 API 地址、测试账号、token、Sentry/监控配置是否被打入生产产物。

## 怎么检查

1. 确认项目范围：先定位根目录和子包，找出 `package.json`、lockfile、workspace、构建配置和 CI 文件。
2. 读取脚本链路：从 `package.json scripts` 追踪到实际执行的 CLI、配置文件、环境变量和产物目录。
3. 对齐包管理器：比较 `packageManager`、lockfile 类型、CI 安装命令和 workspace 配置，判断是否存在混用或不可复现风险。
4. 对齐版本：比较 `engines`、`.nvmrc`、`.node-version`、`volta`、CI Node/npm/pnpm/yarn 版本。
5. 检查 TypeScript 与构建工具：逐项读取 `tsconfig*.json`、Vite/Webpack/Babel/ESLint 配置，确认别名、模块解析、输出、类型检查和 lint 行为一致。
6. 检查依赖风险：围绕缺失依赖、peer 依赖、重复依赖、运行时依赖错放、过期构建插件和 `overrides/resolutions` 做静态判断。
7. 检查产物与发布字段：比对构建输出目录、包入口、类型声明、导出条件和部署脚本。
8. 检查环境变量与安全暴露：只做静态审查，标记可能进入浏览器产物或日志的敏感信息，不输出密钥原文。
9. 检查 CI 一致性：读取工作流或流水线配置，判断安装、缓存、构建、测试、产物步骤是否覆盖关键脚本。
10. 形成报告：只写已确认问题和无法确认的风险，所有问题必须给出文件路径、行号、问题等级、原因和建议。

## 多平台工具建议

- Codex：优先用 `rg`、`rg --files` 搜索文件和关键字；读取文件可用 `Get-Content -Encoding UTF8`；复杂任务用 `update_plan` 跟踪进度。
- Claude：优先用 `Glob` 查找文件，用 `Grep` 搜索关键字，用 `Read` 读取具体文件。
- Trae：优先用 `SearchCodebase` 搜索代码和配置，用 `Read` 读取具体文件。
- 任一平台都不要绑定单一工具；如果首选工具不可用，使用平台提供的等价搜索和读取能力。

## 执行规则

- 只做代码层质量审查，不跑 API 请求。
- 不默认执行任何 Git 操作。
- 不默认执行安装、构建、测试、审计、网络下载等命令；如用户明确要求或确认执行，报告中说明命令和结果。
- 不自动修复问题，只输出问题、原因、风险和修复建议。
- 不启用 hooks、后台观察、强制自动修复或平台专属命令流程。
- 输出必须包含文件路径、行号、问题等级、原因、建议。
- 报告输出遵循 `test-quality-report-template`。
- 如需要修改代码，必须等待用户确认后逐项处理。
- 多文件修改必须按安全批量修改规则执行。
- 中文路径和中文内容必须保持 UTF-8。

## 输出要求

按 `test-quality-report-template` 输出 Markdown 报告，至少包含：

- 检查范围、检查时间、检查人员、技术栈、使用 skill。
- 问题统计。
- 阻断问题、高风险问题、中风险问题、建议优化。
- 无法确认的风险和后续处理建议。
- 每个问题必须包含：文件路径、行号、问题等级、原因、风险或影响、建议。
- 没有发现问题时，也必须说明检查范围、无法确认项和剩余风险。

## 内置规则覆盖

- 构建系统：优先判断 npm、pnpm、yarn、bun 是否混用；`packageManager`、lockfile、workspace 和 CI 安装命令必须一致。
- 脚本链路：从 `package.json scripts` 追踪 `build`、`typecheck`、`lint`、`test`、`prepare`、`postinstall` 到实际 CLI 和配置文件。
- TypeScript：检查 `strict`、`skipLibCheck`、`paths/baseUrl`、`moduleResolution`、`composite`、`declaration`、`noEmit` 与构建工具是否一致。
- 打包工具：检查 Vite/Webpack/Babel/ESBuild 的入口、alias、target、sourcemap、publicPath/base、tree shaking、chunk 和 polyfill 策略。
- 依赖风险：运行时依赖不能误放到 devDependencies；peerDependencies、overrides/resolutions、重复依赖和过宽版本要说明影响。
- 安全暴露：生产构建不得暴露密钥、内部 API、测试账号、mock 开关、调试 sourcemap 或不该公开的环境变量。
- CI 可复现：Node 版本、包管理器版本、缓存 key、lockfile 校验、workspace 构建顺序和产物上传必须可追溯。
