---
name: "test-quality-pattern-cpp-testing"
description: "Use when reviewing C++ tests, Google Test, fixtures, assertions, coverage, or C++ testing design."
---
# C++ 测试模式质量审查

## 定位

C++ 测试设计与测试工程质量审查，重点判断 Google Test / Google Mock、CTest、覆盖率、sanitizer、边界用例和可测试性是否能真实保护代码行为。

## 检查重点

- 测试分层：unit / integration / testdata 是否清晰，是否把慢测试、外部依赖测试和单元测试混在一起。
- Google Test：`TEST` / `TEST_F` 命名、断言选择、失败信息、测试是否只验证一个行为。
- Fixture：共享初始化是否必要，是否隐藏测试前置条件，是否正确清理全局状态和临时资源。
- Google Mock：mock 是否只验证交互，是否过度 mock 值对象或简单数据结构。
- CTest/CMake：测试目标、`gtest_discover_tests()`、测试标签、失败输出和 CI 执行入口是否稳定。
- 覆盖率：关键分支、异常路径、边界值、回归场景是否有测试；不要只追求覆盖率数字。
- Sanitizer：ASan/UBSan/TSan 是否适合当前模块，内存、未定义行为和竞态问题是否能被测试暴露。
- Flaky 风险：真实时间、网络、固定临时路径、线程时序、随机数和共享状态是否导致不稳定。
- 可测试性：业务逻辑是否能脱离 IO、线程、系统时间和外部服务独立测试。
- 性能/压力测试边界：是否把 benchmark 当功能断言，是否缺少输入规模和超时边界。

## 怎么检查

1. 先识别测试框架和入口：查看 `CMakeLists.txt`、`CTestTestfile.cmake`、`tests/`、`*_test.cpp`、CI 配置。
2. 从被测模块反推测试覆盖：公共接口、错误分支、资源释放、并发路径和历史缺陷是否有对应测试。
3. 逐个测试文件检查断言质量：`ASSERT_*` 用于前置条件，`EXPECT_*` 用于同一行为下的多个结果检查。
4. 检查 fixture 是否让测试难以独立理解；发现共享状态时追踪 `SetUp` / `TearDown` / 静态变量。
5. 检查 mock 是否验证实现细节；优先建议 fake/stub 保护可观察行为。
6. 检查 flaky 风险时，必须指出触发条件，例如固定 sleep、固定路径、无界等待或跨测试共享资源。
7. 对覆盖率和 sanitizer 只基于项目现有配置提出审查建议；无法运行时说明需要的命令和环境。

## 常用搜索线索

- 测试入口：`add_executable(`、`target_link_libraries`、`enable_testing()`、`gtest_discover_tests`、`add_test(`。
- gtest/gmock：`TEST(`、`TEST_F(`、`TEST_P(`、`EXPECT_`、`ASSERT_`、`EXPECT_CALL`、`ON_CALL`。
- flaky：`sleep_for`、`Sleep`、`wait_for`、`detach`、固定临时路径、`std::rand`、真实网络地址。
- 共享状态：`static`、全局变量、单例、缓存、环境变量、`SetUpTestSuite`。
- 覆盖率/sanitizer：`--coverage`、`fprofile-instr-generate`、`fsanitize`、`ASAN_OPTIONS`、`TSAN_OPTIONS`。

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
- Claude：优先使用 `Glob` / `Grep` / `Read`，按平台能力逐个文件复核。
- Trae：优先使用 `SearchCodebase` / `Read`，避免未确认范围的批量写入。

## 输出要求

按阻断、高风险、中风险、建议优化分组。每条问题包含：

- 文件路径和行号
- 问题等级
- 测试缺口或测试设计问题
- 风险原因
- 修复建议
- 无法确认时的下一步验证建议

## 内置规则覆盖

- 测试分层：检查 unit、integration、contract、testdata、fixture 和端到端路径是否分清，避免慢测混入单测。
- gtest/gmock：检查 `TEST`/`TEST_F`/`TEST_P`、fixture 生命周期、`ASSERT_*`/`EXPECT_*` 用法、matcher 可读性和 mock 行为边界。
- 替代框架：识别 Catch2、doctest、Boost.Test 项目，按等价断言、fixture、参数化和 discovery 规则审查。
- CMake/CTest：检查 `enable_testing`、`add_test`、test discovery、目标链接、测试资源路径和跨平台运行参数。
- 覆盖率：检查 coverage 编译 flags、链接 flags、Debug/Release 配置一致性、排除规则和报告生成路径。
- Sanitizer：检查 ASan/UBSan/TSan/LSan 启用条件、CI 配置、误报抑制和与 coverage 的组合风险。
- Flaky 控制：检查时间依赖、随机数种子、线程等待、网络/文件依赖、固定临时目录清理和环境隔离。
- Mock 与 fake：检查是否过度 mock、是否验证业务结果而非实现细节，外部系统是否有 fake 或 test double。
- 高级测试：纯函数、解析器、序列化、边界算法可考虑 property test 或 fuzz test，并说明触发条件和输入约束。
