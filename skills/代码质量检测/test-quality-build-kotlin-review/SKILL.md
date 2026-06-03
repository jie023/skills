---
name: "test-quality-build-kotlin-review"
description: "Use when diagnosing or reviewing Kotlin, Gradle, Ktor, Android, Kotlin Multiplatform, compiler options, dependency alignment, or build quality risks."
---
# Kotlin 构建质量审查

## 定位

用于审查 Kotlin/JVM、Android、Kotlin Multiplatform、Ktor、Compose 项目的构建配置质量、编译稳定性、依赖一致性、静态检查和 CI 可复现性。该 skill 只输出审查发现和验证建议，不默认修改业务代码，不默认执行 Git 操作。

## 适用场景

- Kotlin、Gradle、Android Gradle Plugin、KMP、Ktor、Compose 相关构建失败或构建不稳定。
- Kotlin 编译器参数、JDK toolchain、Gradle wrapper、source set、依赖版本对齐存在风险。
- kapt、ksp、detekt、ktlint、测试任务、CI 缓存、构建变体配置需要质量审查。
- 用户要求对 Kotlin 构建脚本、版本目录、CI 构建流程进行代码级审查。

## 执行规则

- 只做代码层质量审查，不跑 API 请求。
- 不默认执行任何 Git 操作。
- 不自动修复问题，只输出问题和修复建议。
- 不启用 hooks、后台观察、强制命令或自动修复流程。
- 输出必须包含文件路径、行号、问题等级、原因、建议。
- 报告输出遵循 `test-quality-report-template`：先列问题，按阻断、高风险、中风险、建议优化分组；无问题时明确说明剩余风险和未验证项。
- 如需要修改代码，必须等待用户确认后逐项处理。
- 多文件修改必须按安全批量修改规则执行。
- 中文路径和中文内容必须保持 UTF-8。

## 平台检查方式

- Codex：优先用 `rg --files` 查找 `settings.gradle*`、`build.gradle*`、`gradle.properties`、`gradle-wrapper.properties`、`libs.versions.toml`、`*.gradle.kts`、`.github/workflows/*`；用 `rg -n` 搜索插件、版本、任务和配置；用 `Get-Content -Encoding UTF8` 读取目标文件。
- Claude：优先用 `Glob` 查找 Gradle、CI、版本目录和 Kotlin 源集文件；用 `Grep` 定位插件、依赖、compilerOptions、kapt/ksp、detekt/ktlint；用 `Read` 读取上下文。
- Trae：优先用 `SearchCodebase` 按关键词和文件模式搜索；用 `Read` 读取命中文件和相邻上下文。
- 所有平台都必须在报告中给出命中文件路径和行号；不能只给笼统结论。

## 重点检查项

### 版本与工具链

- Kotlin 插件版本是否集中管理，是否与 Gradle、Android Gradle Plugin、Compose Compiler、KSP、Ktor、协程库版本兼容。
- Gradle wrapper 是否提交 `gradle/wrapper/gradle-wrapper.properties`，`distributionUrl` 是否使用明确版本，CI 是否调用 wrapper 而不是系统 Gradle。
- JDK toolchain 是否明确，例如 `kotlin { jvmToolchain(...) }` 或 Java toolchain；Android 项目是否同时检查 `compileOptions`、`kotlinOptions` 或 `compilerOptions` 的 JVM target 一致性。
- 多模块是否混用不同 Kotlin、AGP、KSP、Compose Compiler 或 JVM target。

### Gradle 结构与插件

- `settings.gradle.kts` 是否正确声明 plugin management、dependency repositories、include 模块和版本目录。
- 根项目与子项目是否重复硬编码插件版本，是否存在 `subprojects/allprojects` 过度注入导致配置缓存失效或模块边界不清。
- Kotlin/JVM、Android、KMP、Compose、Ktor 插件是否只应用在需要的模块，`apply false` 与模块级 `plugins` 是否清晰。
- Android 构建变体、flavor、build type 是否影响依赖、测试任务、资源或发布产物。

### 源集与目标平台

- Kotlin source set 是否与项目类型匹配：JVM 使用 `src/main/kotlin`；Android 使用 `src/main`、`src/test`、`src/androidTest`；KMP 使用 `commonMain`、`jvmMain`、`androidMain`、`iosMain` 等。
- KMP `dependsOn` 关系是否合理，平台专属依赖是否误放到 `commonMain`。
- 测试源集是否有对应依赖和任务，避免只配置主源码而漏掉 `testImplementation`、`androidTestImplementation` 或 KMP 测试源集。

### 编译器与注解处理

- Kotlin `compilerOptions` 是否集中、可追踪，`jvmTarget`、`languageVersion`、`apiVersion`、`freeCompilerArgs` 是否与项目兼容。
- `allWarningsAsErrors` 是否在本地和 CI 中有一致策略，是否会因第三方生成代码导致不可复现失败。
- kapt 与 ksp 是否混用；Room、Moshi、Dagger/Hilt、MapStruct 等处理器是否使用匹配的 kapt/ksp 依赖。
- kapt 是否配置增量、correctErrorTypes 等必要参数；KSP 版本是否与 Kotlin 版本严格匹配。

### 依赖与版本对齐

- 是否使用版本目录、BOM 或平台依赖对齐 Kotlin、Compose、Ktor、协程、serialization、AndroidX、测试框架版本。
- 是否存在动态版本、SNAPSHOT、重复依赖、冲突排除过宽、强制版本掩盖真实冲突。
- 是否为不同 source set 或 variant 放置正确依赖，避免 Android-only 依赖泄漏到 JVM 或 common source set。
- 是否检查 `dependencyInsight`、锁文件、dependency verification 或依赖替换规则对构建可复现性的影响。

### 静态检查、测试与 CI

- detekt、ktlint 是否有明确任务、配置文件和基线策略，CI 是否执行对应检查。
- 单元测试、Android instrumentation test、KMP 平台测试是否在 CI 覆盖，任务名是否与模块和变体匹配。
- CI 是否使用 wrapper、固定 JDK、缓存 Gradle user home 与 build cache，缓存 key 是否包含 wrapper、版本目录、Gradle 脚本和 lockfile。
- 构建缓存、配置缓存、并行构建开关是否与自定义任务、kapt、Android 变体兼容。

## 怎么检查

1. 定位构建入口：查找 `settings.gradle*`、根 `build.gradle*`、`gradle.properties`、`gradle/wrapper/gradle-wrapper.properties`、`libs.versions.toml`。
2. 定位模块类型：搜索 `org.jetbrains.kotlin.jvm`、`org.jetbrains.kotlin.android`、`org.jetbrains.kotlin.multiplatform`、`com.android.application`、`com.android.library`、`org.jetbrains.compose`、`io.ktor.plugin`。
3. 对照版本矩阵：记录 Kotlin、Gradle、AGP、KSP、Compose Compiler、Ktor、JDK target 的声明位置和实际使用模块，发现混用或缺失要列为风险。
4. 检查 source set：搜索 `sourceSets`、`commonMain`、`jvmMain`、`androidMain`、`androidTest`、`testImplementation`，确认依赖和任务覆盖目标平台。
5. 检查注解处理：搜索 `kapt`、`ksp`、`annotationProcessor`、`symbolProcessor`，确认插件、依赖和处理器版本一致。
6. 检查编译选项：搜索 `compilerOptions`、`kotlinOptions`、`freeCompilerArgs`、`jvmTarget`、`allWarningsAsErrors`、`jvmToolchain`，确认没有模块间冲突。
7. 检查依赖对齐：搜索 `platform(`、`enforcedPlatform`、`bom`、`strictly`、`force`、`resolutionStrategy`、`exclude`、`+` 动态版本，确认是否可复现。
8. 检查质量任务：搜索 `detekt`、`ktlint`、`test`、`check`、`connectedAndroidTest`、`allTests`，确认 CI 是否运行。
9. 检查 CI 与缓存：读取 `.github/workflows`、`.gitlab-ci.yml`、`Jenkinsfile` 等，确认 JDK、wrapper、缓存 key、构建变体和测试任务。
10. 必要时建议用户执行只读或验证命令，例如 `./gradlew --version`、`./gradlew tasks`、`./gradlew check`、`./gradlew dependencyInsight --dependency <name> --configuration <configuration>`；执行前按当前会话规则确认。

## 输出格式

每条发现使用以下格式：

```text
等级：高风险
文件：app/build.gradle.kts:42
问题：KSP 版本与 Kotlin 插件版本不匹配
原因：KSP 版本必须跟随 Kotlin 编译器版本，否则可能出现符号处理失败或增量编译异常。
建议：将 KSP 版本调整为与 Kotlin 版本匹配的发布版本，并在版本目录集中声明。
验证：检查 libs.versions.toml 中 kotlin 与 ksp 的版本声明，必要时运行 ./gradlew :app:kspDebugKotlin。
```

## 内置规则覆盖

- 版本矩阵：Kotlin、Gradle、AGP、KSP、Compose Compiler、Ktor、协程、serialization 和 JDK target 需要成组判断。
- Wrapper 与 toolchain：CI 应使用 Gradle wrapper 和固定 JDK；本地系统 Gradle/JDK 与 CI 不一致要标风险。
- Source set：KMP 的 `commonMain` 不能放平台专属依赖；Android/JVM/iOS source set 依赖和测试任务必须匹配。
- 注解处理：KSP 版本必须跟随 Kotlin 编译器；kapt/ksp 混用、Room/Hilt/Moshi 处理器错配会导致增量编译或符号处理失败。
- 依赖对齐：版本目录、BOM、platform、dependencyInsight、dependency verification、锁文件和 resolutionStrategy 都是判断依据。
- CI 与缓存：缓存 key 应覆盖 wrapper、版本目录、Gradle 脚本、lockfile；配置缓存和构建缓存要确认自定义任务兼容。
