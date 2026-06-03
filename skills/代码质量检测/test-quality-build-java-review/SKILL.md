---
name: "test-quality-build-java-review"
description: "Use when diagnosing Java, Maven, Gradle, Spring Boot compile, dependency, annotation processor, or toolchain failures."
---
# Java 构建质量审查

## 定位

用于审查 Java 项目的构建链路质量，覆盖 Maven、Gradle、Spring Boot、JDK、依赖解析、注解处理、测试跳过、打包产物、CI、Docker、编码与静态检查配置。目标是找出会导致本地、流水线或容器环境构建不稳定的根因，并输出可验证的修复建议。

## 检查重点

- 构建工具识别：检查 `pom.xml`、`mvnw`、`build.gradle`、`build.gradle.kts`、`gradlew`，确认项目实际使用 Maven、Gradle 或多模块混合构建。
- JDK 版本一致性：检查 `maven-compiler-plugin`、`maven-toolchains-plugin`、Gradle `java.toolchain`、`sourceCompatibility`、`targetCompatibility`、`.java-version`、`Dockerfile`、CI 配置中的 JDK 版本是否一致。
- Spring Boot 插件：检查 `spring-boot-maven-plugin` 或 `org.springframework.boot` Gradle 插件版本是否与 Spring Boot 依赖管理匹配，确认 `repackage`、主类、分层 jar、排除项配置合理。
- 依赖冲突：检查 Maven `dependency:tree -Dverbose` 或 Gradle `dependencies`、`dependencyInsight` 输出，关注重复版本、传递依赖覆盖、BOM 顺序、父 POM 继承、`resolutionStrategy` 强制版本。
- 依赖 scope/configuration：检查 Maven `compile`、`provided`、`runtime`、`test`、`optional` 与 Gradle `implementation`、`api`、`compileOnly`、`runtimeOnly`、`testImplementation` 是否符合运行时和测试需要。
- 注解处理器：检查 Lombok、MapStruct、QueryDSL、Spring Configuration Processor 等是否配置在 `annotationProcessorPaths`、`annotationProcessor`、`compileOnly` 或 IDE/CI 可识别的位置。
- profile 与环境差异：检查 Maven profile、Gradle property、Spring profile、CI 环境变量是否导致本地与流水线使用不同仓库、JDK、测试、资源或打包参数。
- resource filtering：检查 `resources`、`filtering`、`delimiters`、`application*.yml`、编码配置，避免二进制资源被过滤、占位符未替换或中文乱码。
- 测试执行策略：区分 `-DskipTests`、`-Dmaven.test.skip=true`、Gradle `-x test` 的影响，检查 Surefire/Failsafe、JUnit Platform、集成测试命名和跳过条件是否误伤质量门禁。
- 打包产物：检查 jar/war 名称、classifier、manifest、主类、依赖是否进入产物、Docker COPY 路径、Spring Boot fat jar 与普通 jar 是否混用。
- CI 构建链路：检查流水线命令是否包含 wrapper 校验、依赖缓存、私服凭据、测试、静态检查、制品归档，确认本地命令和 CI 命令可复现。
- Docker 构建：检查基础镜像 JDK/JRE 版本、构建阶段与运行阶段路径、时区、编码、非 root 用户、健康检查、`JAVA_TOOL_OPTIONS`、容器内启动命令。
- 编码 UTF-8：检查 Maven `project.build.sourceEncoding`、`project.reporting.outputEncoding`、Gradle `JavaCompile.options.encoding`、资源过滤编码、Docker/CI locale，确保中文源码和配置不乱码。
- 质量工具：检查 P3C、Checkstyle、SpotBugs、PMD、Error Prone、JaCoCo 等是否接入构建、是否被默认跳过、版本是否兼容当前 JDK。

## 内置规则覆盖

- Maven 根因判断：父 POM、BOM、`dependencyManagement`、`pluginManagement`、profile、wrapper 和有效 POM 都可能改变实际构建结果。
- Gradle 根因判断：`settings.gradle*`、版本目录、pluginManagement、wrapper、toolchain、configuration cache、build cache 和子项目插件应用都要一起看。
- 常见阻断：`cannot find symbol`、依赖无法解析、注解处理器未运行、JDK major version 不兼容、Spring Boot repackage 失败、测试框架未发现。
- 依赖建议：优先建议查看 dependency tree / dependencyInsight；不要直接降级依赖，除非已经确认版本冲突来源和兼容范围。
- 测试跳过：区分 `skipTests`、`maven.test.skip`、`-x test` 和 profile 条件跳过；默认跳过测试应标为高风险。
- 编码与资源：中文源码、过滤资源、Docker locale、CI locale 和 `JavaCompile.options.encoding` 必须统一 UTF-8。

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

## 怎么检查

### 1. 搜索与读取

- Codex：优先使用 `rg --files` 查找构建文件，使用 `rg --line-number` 搜索关键配置，使用 `Get-Content -LiteralPath <path> -Encoding UTF8` 读取 Windows 中文路径文件。
- Claude：优先使用 `Glob` 查找 `pom.xml`、`build.gradle*`、`.github/workflows/*`、`Dockerfile*`，使用 `Grep` 搜索关键配置，使用 `Read` 读取文件。
- Trae：优先使用 `SearchCodebase` 搜索构建配置和错误关键字，使用 `Read` 读取具体文件。
- 搜索结果和报告引用必须包含文件路径、行号和相关代码内容。无法获得行号时，要先换用支持行号的搜索或读取方式。

### 2. 构建文件定位

- 查找 Maven：`pom.xml`、`.mvn/maven.config`、`.mvn/jvm.config`、`mvnw`、父子模块 `<modules>`。
- 查找 Gradle：`settings.gradle`、`settings.gradle.kts`、`build.gradle`、`build.gradle.kts`、`gradle.properties`、`gradlew`、`buildSrc`、版本目录 `libs.versions.toml`。
- 查找运行环境：`Dockerfile`、`docker-compose*.yml`、`.github/workflows`、`.gitlab-ci.yml`、`Jenkinsfile`、`azure-pipelines.yml`、`buildspec.yml`。

### 3. Maven 检查方法

- 检查有效配置：查看父 POM、BOM、`dependencyManagement`、`pluginManagement`、profile 激活条件和 wrapper 版本。
- 检查 JDK：查看 `maven.compiler.release`、`maven.compiler.source`、`maven.compiler.target`、`maven-compiler-plugin`、`maven-toolchains-plugin`。
- 检查依赖：建议用户运行 `mvn dependency:tree -Dverbose`、`mvn dependency:analyze`、`mvn help:effective-pom`；报告中根据配置指出冲突、未声明依赖、未使用依赖或 scope 错误。
- 检查测试：查看 `maven-surefire-plugin`、`maven-failsafe-plugin`、`skipTests`、`maven.test.skip`、JUnit 版本和集成测试绑定阶段。
- 检查资源与编码：查看 `resources`、`filtering`、`project.build.sourceEncoding`、`project.reporting.outputEncoding`。

### 4. Gradle 检查方法

- 检查插件和 wrapper：查看 `plugins`、`buildscript`、`pluginManagement`、`gradle-wrapper.properties`，确认 Gradle 版本支持当前 JDK 与 Spring Boot 插件。
- 检查 JDK：查看 `java { toolchain }`、`sourceCompatibility`、`targetCompatibility`、`tasks.withType(JavaCompile)`。
- 检查依赖：建议用户运行 `./gradlew dependencies --configuration runtimeClasspath` 和 `./gradlew dependencyInsight --dependency <name> --configuration runtimeClasspath`；报告中指出冲突来源和约束配置。
- 检查测试：查看 `tasks.test`、`useJUnitPlatform()`、`-x test`、自定义 test task、JaCoCo 是否接入 `check`。
- 检查资源与编码：查看 `processResources`、`expand`、`filteringCharset`、`JavaCompile.options.encoding`。

### 5. 问题判断

- 阻断：编译失败、依赖无法解析、JDK 版本不兼容、注解处理器缺失、打包产物无法启动、CI 必现失败。
- 高风险：本地和 CI JDK 不一致、测试被默认跳过、Spring Boot 插件和依赖管理不匹配、运行时依赖被配置为 `test` 或 `provided`。
- 中风险：依赖版本由传递依赖隐式决定、profile 行为不透明、资源过滤可能误处理配置、Docker 运行镜像与构建版本不一致。
- 建议优化：补充 wrapper、统一编码、接入 P3C/Checkstyle/SpotBugs/PMD、补充依赖分析和制品归档。

## 输出要求

按阻断、高风险、中风险、建议优化分组。无法确认的问题要说明原因和下一步验证建议。

每个问题使用以下结构，保持与 `test-quality-report-template` 的缺陷报告风格一致：

```text
### [问题等级] 问题标题

- 文件路径：path/to/file
- 行号：12
- 相关代码：`具体配置或代码片段`
- 原因：说明为什么该配置会导致构建失败、不稳定或质量门禁失效
- 建议：给出可执行的修改方向、验证命令或需要用户确认的外部依赖
```

没有发现问题时，也要说明已检查的构建文件、CI/Docker/编码/静态检查范围，以及未执行命令导致的剩余风险。
