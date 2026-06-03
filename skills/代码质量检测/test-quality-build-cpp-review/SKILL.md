---
name: "test-quality-build-cpp-review"
description: "Use when diagnosing C++ CMake, compile, linker, template, include, ABI, or toolchain failures."
---
# C++ 构建质量审查

## 定位

C++ 构建、CMake、编译器、链接器、ABI、依赖管理和跨平台工具链质量审查。只做诊断、归类和修复建议，不默认执行构建或自动修改代码。

## 检查重点

- CMake 结构：`CMakeLists.txt`、toolchain file、preset、target、source、include、link、install/export 是否按 target 管理。
- 构建入口：`cmake -S/-B`、`cmake --build`、Makefile、Ninja、Visual Studio、CI/Docker 命令是否一致。
- C++ 标准：`CMAKE_CXX_STANDARD`、`target_compile_features`、编译器版本和库特性是否匹配。
- include 路径：public/private/interface include 作用域、头文件生成目录、相对路径和系统头是否正确。
- 链接错误：缺实现文件、库链接顺序、静态/动态库混用、导出符号、Windows import library、Linux rpath。
- ABI/运行库：MSVC runtime、libstdc++/libc++、Debug/Release 混链、编译选项和二进制依赖是否一致。
- 依赖管理：Conan、vcpkg、FetchContent、submodule、find_package、pkg-config 版本和可复现性。
- 模板/宏/生成代码：模板实例化、宏定义、protobuf/Qt/moc/thrift/flatbuffers 生成步骤是否纳入构建。
- 跨平台：MSVC/GCC/Clang 差异、Windows/Linux/macOS 路径、编码、平台宏、编译选项兼容性。
- 质量工具：clang-tidy、clang-format、sanitizer、warnings-as-errors 是否合理接入并可在 CI 复现。

## 怎么检查

1. 先读取 `CMakeLists.txt`、`CMakePresets.json`、toolchain 文件、Conan/vcpkg 配置、CI/Docker 脚本，确认实际构建入口。
2. 如用户没有明确要求运行构建，只做静态诊断；生成 build 目录、下载依赖、清理缓存或安装系统库前必须确认。
3. 对错误日志先判断阶段：配置阶段、编译阶段、链接阶段、测试阶段、安装/打包阶段。
4. 对 include 问题，追踪目标的 `target_include_directories` 作用域和头文件实际位置，不建议全局 `include_directories` 作为默认修复。
5. 对链接问题，检查源文件是否加入 target、库是否链接到正确 target、符号是否因模板/inline/visibility/ABI 产生。
6. 对依赖问题，比较包管理配置、lockfile/profile/triplet、CI 缓存和本机工具链差异。
7. 对跨平台问题，必须说明平台、编译器和配置类型，不把某平台专属写法泛化成通用错误。

## 常用搜索线索

- CMake：`add_executable`、`add_library`、`target_sources`、`target_include_directories`、`target_link_libraries`、`find_package`、`FetchContent`。
- 工具链：`CMAKE_TOOLCHAIN_FILE`、`CMAKE_CXX_COMPILER`、`CMAKE_BUILD_TYPE`、`CMAKE_CXX_STANDARD`、`target_compile_features`。
- 依赖：`conanfile.txt`、`conanfile.py`、`vcpkg.json`、`vcpkg-configuration.json`、`pkg_check_modules`。
- 链接：`undefined reference`、`unresolved external symbol`、`multiple definition`、`duplicate symbol`、`LNK`、`rpath`。
- 平台：`_WIN32`、`__linux__`、`__APPLE__`、`MSVC`、`__GNUC__`、`__clang__`。
- 生成代码：`protobuf_generate`、`qt_add_executable`、`AUTOMOC`、`moc`、`flatc`、`thrift`。

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

- Codex：优先使用 `rg` / `rg --files` 搜索，`Get-Content -Encoding UTF8` 读取中文路径文件。
- Claude：优先使用 `Glob` / `Grep` / `Read`，复杂任务用可用 Todo 工具跟踪。
- Trae：优先使用 `SearchCodebase` / `Read`，避免未确认范围的批量写入。
- 任一平台输出都必须包含文件路径、行号和最小必要配置片段。

## 输出要求

按阻断、高风险、中风险、建议优化分组。每条问题包含：

- 文件路径和行号
- 问题等级
- 涉及构建阶段、target、编译器或平台
- 根因判断
- 风险原因
- 最小修复建议
- 无法确认时的下一步验证建议

## 内置规则覆盖

- 诊断阶段：先区分 CMake 配置错误、编译错误、模板实例化错误、链接错误、测试/安装/打包错误。
- CMake 原则：优先 target 级别的 `target_sources`、`target_include_directories`、`target_link_libraries`、`target_compile_features`，避免建议全局污染。
- 链接根因：缺源文件、缺库、链接顺序、重复定义、符号可见性、模板定义位置、静态/动态库混用和 ABI 不一致要分开判断。
- 工具链：MSVC/GCC/Clang、libstdc++/libc++、Debug/Release、运行库、C++ 标准和 sanitizer 选项必须一致。
- 依赖来源：Conan、vcpkg、FetchContent、submodule、pkg-config、find_package 和 CI 缓存都可能影响可复现性。
- 停止条件：生成 build 目录、清缓存、下载依赖、安装系统库或切换工具链前必须确认。
