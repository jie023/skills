---
name: "test-quality-build-review"
description: "Use when diagnosing build, compile, package, type, dependency, or toolchain failures without automatically fixing code."
---
# 通用构建质量审查

## 定位

通用构建质量审查，只做构建系统识别、错误归类、风险判断和最小修复建议。适用于无法先确定语言专项时的编译、打包、类型检查、依赖、工具链和 CI 构建问题。

## 检查重点

- 构建系统识别：Maven、Gradle、npm/pnpm/yarn、Go modules、Cargo、CMake、Python build、Docker/CI 等。
- 构建入口：`build`、`compile`、`test`、`typecheck`、`lint`、`package`、`docker build` 是否明确且可重复。
- 工具链版本：JDK、Node、pnpm/npm、Go、Rust、CMake、Python、Gradle wrapper、Maven wrapper 是否与项目声明一致。
- 依赖问题：缺包、版本冲突、scope 错误、锁文件不一致、依赖重复、传递依赖冲突。
- 错误分组：先区分语法/类型、依赖、配置、生成代码、资源、测试、环境变量、平台兼容和 CI 专属问题。
- 修复顺序：先修阻断性的依赖/配置/生成代码问题，再看类型和业务代码编译错误。
- 最小建议：给出最小变更方向，不把构建修复扩大成架构重构或业务逻辑重写。
- 停止条件：需要安装依赖、下载网络资源、改生产配置、修改密钥、删除文件或大范围重构时必须停下并说明。
- 剩余风险：无法运行构建或缺少环境时，说明未验证项和下一步验证命令。

## 怎么检查

1. 先查找构建入口文件：`pom.xml`、`build.gradle(.kts)`、`package.json`、`go.mod`、`Cargo.toml`、`CMakeLists.txt`、`pyproject.toml`、`Dockerfile`、CI 配置。
2. 读取脚本和 wrapper：确认项目推荐命令、工具版本、环境变量、profile、workspace、monorepo 结构。
3. 如用户没有要求执行构建，只做静态诊断；如需要运行命令，先说明目的和可能写入的目录。
4. 如果已有错误日志，先按第一处根因分组，不要逐行机械罗列所有级联错误。
5. 对依赖/锁文件问题，必须同时查看声明文件和锁文件；无法确认时说明需要安装或解析依赖树。
6. 对 CI 问题，比较本地构建入口和 CI 脚本差异，重点看环境变量、缓存、工作目录、平台和权限。
7. 对修复建议，优先给出单点最小改动；多文件修改必须等待用户确认后逐项处理。

## 常用搜索线索

- 构建入口：`pom.xml`、`build.gradle`、`gradlew`、`package.json`、`pnpm-lock.yaml`、`go.mod`、`Cargo.toml`、`CMakeLists.txt`。
- 版本声明：`.java-version`、`.node-version`、`.nvmrc`、`engines`、`toolchain`、`sourceCompatibility`、`rust-toolchain.toml`。
- 常见脚本：`build`、`compile`、`typecheck`、`lint`、`test`、`package`、`ci`、`docker`。
- 错误线索：`Cannot find module`、`Could not resolve`、`NoSuchMethodError`、`Unsupported class file major version`、`undefined reference`、`borrow checker`、`cannot find symbol`。
- 生成代码：`target/generated-sources`、`build/generated`、`annotationProcessor`、`kapt`、`ksp`、`protobuf`、`openapi`。

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

## 平台兼容

- Codex：优先使用 `rg` / `rg --files` 搜索，`Get-Content -Encoding UTF8` 读取中文路径文件；复杂任务用 `update_plan` 更新状态。
- Claude：优先使用 `Glob` / `Grep` / `Read`，复杂任务用可用 Todo 工具更新状态。
- Trae：优先使用 `SearchCodebase` / `Read`，复杂任务用可用任务清单工具更新状态。
- 任一平台都必须保留文件路径、行号和最小必要代码/配置片段。

## 输出要求

按阻断、高风险、中风险、建议优化分组。每条问题包含：

- 文件路径和行号
- 问题等级
- 构建系统或命令范围
- 根因判断
- 风险原因
- 最小修复建议
- 无法确认时的下一步验证建议

## 内置规则覆盖

- 构建系统识别：按 `package.json`、`Cargo.toml`、`pom.xml`、`build.gradle`、`go.mod`、`pyproject.toml`、`CMakeLists.txt`、Docker/CI 配置识别主构建链路。
- 错误归类：先分配置阶段、依赖解析、编译/类型检查、链接、测试、打包、运行镜像和 CI 环境问题。
- 级联错误处理：优先定位第一处根因，不把后续由缺依赖、缺生成代码或版本冲突引发的级联错误重复报告。
- 修复策略：只给最小修复建议；缺包、版本冲突、循环依赖、生成代码缺失、环境变量缺失分别给出验证方向。
- 停止条件：需要联网安装依赖、改生产配置、删除文件、调整密钥、降级依赖或做架构性改动时必须停下说明。
- 验证建议：能验证时建议最小构建命令；不能运行时说明缺少的工具链、环境变量、依赖仓库或平台条件。
