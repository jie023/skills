---
name: "test-quality-language-cpp-review"
description: "Use when reviewing C++ code quality, CMake, RAII, memory safety, object lifetime, move/copy semantics, const correctness, exception safety, concurrency, undefined behavior, GoogleTest, sanitizers, or C++ Core Guidelines compliance."
---
# C++ 语言专项质量审查

## 定位

C++ 专项代码质量审查。用于审查 C/C++ 项目中的语言级缺陷、构建测试风险和可维护性问题，重点关注 C++ Core Guidelines、RAII、资源生命周期和跨平台构建测试行为。

## 内置规则覆盖

- C++ Core Guidelines：接口表达、资源安全、类型安全、Rule of Zero/Five、const correctness、异常安全和并发安全都纳入审查。
- 资源管理：RAII、智能指针、裸 `new/delete`、文件句柄、锁、线程、socket 和系统资源生命周期必须明确。
- 未定义行为：悬垂引用、越界、未初始化、严格别名、空指针、有符号溢出、移位越界和非平凡类型 `memcpy` 都是高风险。
- 构建测试：CMake target、include/link 作用域、标准版本、sanitizer、clang-tidy、GoogleTest/GoogleMock 是审查依据。
- 现代 C++：优先标准库、`std::span`/`string_view` 生命周期、`constexpr`、`noexcept`、模板 concepts 和标准容器选择要结合项目标准判断。

## 检查重点

- RAII 与资源管理：裸 `new`/`delete`、`malloc`/`free`、文件句柄、锁、线程、socket、GPU/系统资源是否由对象生命周期托管。
- 内存安全与生命周期：悬垂指针/引用、返回局部对象地址、容器扩容后迭代器失效、`string_view`/`span` 指向临时对象、double free、泄漏。
- move/copy 语义：Rule of Zero/Five、移动后对象使用、自定义析构却缺少拷贝/移动控制、返回 `const T` 抑制移动、错误 `std::move`。
- const correctness：只读参数、成员函数、局部值、全局状态是否正确使用 `const`/`constexpr`，是否存在 `const_cast` 或可变共享状态。
- 异常安全：析构/释放/swap 不抛异常，构造失败不泄漏，强/基本异常保证，catch by reference，避免吞异常。
- 并发安全：数据竞争、锁生命周期、死锁顺序、条件变量谓词、线程 detach、atomic 内存序、持锁调用未知代码。
- 未定义行为：越界、未初始化、严格别名、空指针解引用、有符号溢出、移位越界、对象对齐、`memcpy`/`memset` 作用于非平凡类型。
- CMake 与构建：target-based CMake、标准版本、include/link 作用域、平台分支、警告级别、依赖查找、生成器差异。
- GoogleTest/GoogleMock：断言选择、fixture 生命周期、mock 所有权、参数化测试、死亡测试、异步测试稳定性。
- sanitizer 与静态检查：ASan、UBSan、TSan、LSan、MSan、clang-tidy、编译器警告是否能覆盖高风险路径。
- Core Guidelines 快速项：`enum class`、`nullptr`、窄化转换、C-style cast、单参构造 `explicit`、头文件自包含、头文件禁止全局 `using namespace`、避免滥用 `std::endl`。
- 高级测试：纯函数、解析器、序列化、边界算法可考虑 fuzz/property testing；测试目标要区分 GoogleTest、Catch2、doctest 或项目既有框架。

## 执行规则

- 只做代码层质量审查，不跑 API 请求。
- 不默认执行任何 Git 操作。
- 不自动修复问题，只输出问题、证据和修复建议；如用户要求修复，再按其确认范围处理。
- 输出必须包含文件路径、行号、问题等级、原因、影响和建议。
- 报告输出遵循 `test-quality-report-template`。
- 中文路径和中文内容必须保持 UTF-8。

## 检查流程

1. 确认范围：识别 C++ 源码、头文件、`CMakeLists.txt`、测试目录、工具链文件和 CI 配置。
2. 按内置规则和本次任务风险读取源码、头文件、构建配置和测试，避免加载无关内容。
3. 静态扫描：先用 `rg` 搜索高风险语法和构建测试线索，再读取命中文件上下文。
4. 语义判断：结合所有权、生命周期、异常路径、并发路径和平台差异判断是否真实成立。
5. 验证建议：能运行编译、测试或 sanitizer 时列出建议命令；不能运行时说明原因和替代验证方式。
6. 输出报告：按等级分组，先列真实风险，无法确认的问题放入“待验证”或说明证据不足。

## 怎么检查

- 先看接口边界：public API、构造/析构、所有权转移、线程入口、回调、跨模块头文件。
- 再看资源路径：申请、初始化、早返回、异常、释放、重复释放、失败回滚。
- 对类逐个确认：析构函数、拷贝构造、拷贝赋值、移动构造、移动赋值是否符合 Rule of Zero/Five。
- 对指针和引用逐个确认：谁拥有、谁借用、生命周期是否覆盖使用点，能否改为值、引用、`unique_ptr`、`shared_ptr`、`weak_ptr`、`span`、`string_view`。
- 对并发代码确认共享数据、锁保护范围、锁顺序、条件变量谓词、线程 join 策略、atomic 内存序是否与需求匹配。
- 对 CMake 确认是否以 target 为中心，避免全局 `include_directories`/`link_directories`/`add_definitions` 污染。
- 对测试确认失败信息是否清晰、fixture 是否隔离、mock 所有权是否稳定、异步等待是否有超时和确定性。
- 对头文件确认 include guard、最小 include、自包含、命名空间污染、内联变量和 ABI 暴露；对类型转换确认是否能用构造、`static_cast` 或类型安全封装替代。

## 搜索线索

优先使用 `rg -n`，搜索结果要保留路径、行号和命中代码。常用线索：

- 资源与内存：`new |delete|malloc|free|realloc|shared_ptr|unique_ptr|weak_ptr|make_shared|make_unique|get\(|release\(|reset\(`
- 生命周期：`string_view|span|reference_wrapper|return .*&|return .*\*|std::move|std::forward|emplace|push_back|c_str\(`
- UB 与低层操作：`reinterpret_cast|const_cast|static_cast|memcpy|memset|memcmp|union|offsetof|volatile|assert\(`
- 类型与头文件：`enum |nullptr|NULL|explicit |using namespace|std::endl|#pragma once|#ifndef|#include|static_cast|reinterpret_cast|\(.*\)`
- 异常：`throw |catch \(|catch\(\.\.\.\)|noexcept|swap\(|~[A-Za-z_].*\(`
- 并发：`thread|async|future|promise|mutex|shared_mutex|lock\(|unlock\(|lock_guard|unique_lock|scoped_lock|condition_variable|atomic|detach\(`
- CMake：`cmake_minimum_required|project\(|add_executable|add_library|target_|include_directories|link_directories|add_definitions|find_package|FetchContent`
- GoogleTest：`TEST\(|TEST_F\(|TEST_P\(|TYPED_TEST|EXPECT_|ASSERT_|EXPECT_CALL|ON_CALL|MOCK_METHOD|INSTANTIATE_TEST_SUITE_P`
- 高级测试：`FUZZ_TEST|LLVMFuzzerTestOneInput|rapidcheck|Catch2|doctest|GENERATE|PROPERTY`
- sanitizer/工具：`sanitize|ASAN|UBSAN|TSAN|LSAN|MSAN|clang-tidy|cppcheck|/fsanitize|-fsanitize|-Wall|-Wextra|/W4`

## 等级判定

- 阻断：会导致编译失败、链接失败、核心测试不可运行、确定性崩溃、数据损坏、严重 UB、明显数据竞争或资源泄漏。
- 高风险：生产路径可能触发内存破坏、悬垂生命周期、死锁、异常泄漏、错误所有权、跨平台构建失败或测试误报。
- 中风险：局部可维护性或稳定性问题，如 const 缺失、拷贝成本异常、CMake 作用域污染、测试隔离不足、警告被忽略。
- 建议优化：风格、表达力、轻量性能、现代 C++ 用法、测试覆盖和工具链增强建议。

## 输出格式

按“阻断 / 高风险 / 中风险 / 建议优化”分组。每条问题使用：

```text
- [等级] 文件路径:行号
  问题：具体代码现象。
  影响：为什么会造成风险。
  建议：可执行的修复方向或验证命令。
```

没有发现某等级问题时写“未发现”。无法确认的问题必须说明缺少的证据和下一步验证建议，不要把猜测写成确定结论。

## 多平台兼容

- Windows、Linux、macOS 路径和命令写法不同；报告中给出命令时优先使用 CMake 跨平台形式。
- 区分 MSVC、GCC、Clang 的警告、ABI、标准库和 sanitizer 支持差异；MSVC 可优先建议 ASan，TSan/UBSan 需按工具链确认。
- 区分单配置生成器和多配置生成器：`CMAKE_BUILD_TYPE` 对 Visual Studio/Xcode 不生效，应使用 `--config Debug/Release`。
- 不强制项目升级 C++ 标准；如建议 C++20/23 特性，必须给出 C++17 或项目当前标准下的替代方案。
- 嵌入式、实时系统、无异常/无 RTTI 项目需按项目约束调整建议，不能机械套用桌面服务端规则。
