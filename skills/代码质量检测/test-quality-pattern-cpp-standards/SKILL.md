---
name: "test-quality-pattern-cpp-standards"
description: "Use when reviewing C++ Core Guidelines, interfaces, resource management, classes, expressions, concurrency, templates, or standard library use."
---
# C++ 标准模式质量审查

## 定位

C++ Core Guidelines 与现代 C++ 代码质量审查，重点判断接口表达力、资源安全、类型安全、并发安全和标准库使用是否符合可维护的工程实践。

## 检查重点

- 接口表达：函数名、参数数量、强类型封装、所有权语义是否清晰。
- 函数设计：单一职责、`const` / `constexpr` / `noexcept` 使用、返回值优先于输出参数。
- 类与继承：Rule of Zero / Rule of Five、构造完整性、`explicit`、虚析构、禁止构造/析构中调用虚函数。
- 资源管理：RAII、智能指针、裸 `new/delete`、文件句柄/锁/连接等资源生命周期。
- 表达式与类型安全：初始化、C 风格转换、魔法值、`auto` 使用边界、窄化转换。
- 错误处理：异常边界、异常安全、错误类型表达、析构函数抛异常风险。
- 并发：锁的 RAII、共享状态保护、线程生命周期、引用捕获跨线程风险。
- 模板与泛型：concepts 约束、模板复杂度、重载优先于特化、编译期错误可读性。
- 标准库与性能：容器选择、算法复用、`string_view` / `span` / ranges、无意义拷贝、指针追踪和缓存局部性。
- 头文件与命名：头文件依赖、include guard、接口/实现分离、命名是否表达意图。

## 怎么检查

1. 识别项目 C++ 标准与构建方式：查看 `CMakeLists.txt`、`compile_commands.json`、`*.vcxproj`、`conanfile.*`、`vcpkg.json`。
2. 按模块读取头文件和实现文件，先看公共接口，再看资源生命周期和错误边界。
3. 对每个问题定位到具体文件和行号，只报告能从代码证据支撑的问题。
4. 对所有权问题，沿对象创建、传参、保存、释放路径追踪；不能只凭裸指针出现就判错。
5. 对并发问题，追踪共享变量、锁、线程/任务生命周期和异常路径。
6. 对模板问题，优先判断约束是否清晰、错误信息是否可读、是否把简单逻辑过度模板化。
7. 对性能问题，区分真实热点和普通代码；无法确认热点时标为建议优化，并说明需要 profile 或基准测试。

## 常用搜索线索

- 资源风险：`new `、`delete `、`malloc`、`free`、`fopen`、`CloseHandle`、`lock()`、`unlock()`。
- 所有权风险：`T*` 成员、`shared_ptr` 滥用、返回裸指针、保存外部引用、`std::move` 后继续使用。
- 类型风险：`reinterpret_cast`、`const_cast`、C 风格强转、`#define` 常量、未命名魔法值。
- 异常风险：`throw`、`catch (...)`、空 `catch`、析构函数、资源申请后的多出口。
- 并发风险：`std::thread`、`detach`、`async`、`mutex`、`atomic`、引用捕获 lambda。
- 模板风险：`template<`、`enable_if`、`void_t`、偏特化、无约束公共模板。

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
- Claude：优先使用 `Glob` / `Grep` / `Read`，修改前后按平台能力复核 UTF-8。
- Trae：优先使用 `SearchCodebase` / `Read`，避免未确认范围的批量写入。

## 输出要求

按阻断、高风险、中风险、建议优化分组。每条问题包含：

- 文件路径和行号
- 问题等级
- 触发规则或判断依据
- 风险原因
- 修复建议
- 无法确认时的下一步验证建议

## 内置规则覆盖

- C++ Core Guidelines：检查接口语义、所有权表达、生命周期、异常安全、资源管理和类型安全。
- RAII：检查裸 `new/delete`、手工 close/free、异常路径资源泄漏、锁释放和析构副作用。
- 类型安全：检查 `enum class`、`nullptr`、显式转换、signed/unsigned 混算、窄化转换和魔法数字。
- 类与函数：检查 Rule of Zero/Five、拷贝/移动语义、`const` 正确性、参数传递、函数过长和隐藏副作用。
- 智能指针：检查 `unique_ptr`/`shared_ptr`/`weak_ptr` 所有权、循环引用、悬垂引用和不必要共享。
- 错误处理：检查异常边界、错误码、`noexcept`、返回值忽略和构造/析构异常。
- 并发：检查 data race、锁顺序、条件变量谓词、原子内存序和线程生命周期。
- 头文件：检查 include guard、头文件自包含、最小 include、全局 `using namespace`、ODR 风险和 ABI 泄漏。
- 标准库与性能：检查 `std::endl` 滥用、容器选择、迭代器失效、string/view 生命周期、`constexpr` 和不必要拷贝。
