---
name: "test-quality-build-rust-review"
description: "Use when diagnosing Rust cargo build, borrow checker, Cargo.toml, features, edition, MSRV, or dependency failures."
---
# Rust 构建质量审查

## 定位

Rust Cargo 构建、依赖、feature、edition/MSRV、借用检查、clippy 和跨平台 target 的质量审查。只做诊断、归类和修复建议，不默认修改代码或运行会改写状态的命令。

## 检查重点

- Cargo 结构：`Cargo.toml`、`Cargo.lock`、workspace members、profiles、patch、replace、path dependency 是否清晰。
- edition/MSRV：`edition`、`rust-version`、`rust-toolchain.toml`、CI rustc 版本是否一致。
- 构建入口：`cargo check`、`cargo build`、`cargo test`、`cargo clippy`、Makefile/justfile/CI 命令是否一致。
- 依赖解析：feature 冲突、可选依赖、default-features、workspace dependency 继承、锁文件是否应提交。
- 借用检查：生命周期、可变/不可变借用重叠、move 后使用、临时值生命周期、trait bound 错误是否指向真实设计问题。
- build script：`build.rs`、native library、pkg-config、bindgen、环境变量、rerun-if-changed 是否稳定。
- proc-macro/生成代码：宏依赖、派生宏 feature、生成文件和编译顺序是否正确。
- target/平台：`cfg`、target triple、musl/msvc/gnu、WASM、交叉编译、平台专属依赖是否正确隔离。
- clippy/rustfmt：是否作为质量门禁，是否存在过度 `allow` 或用 `unsafe` 绕过编译器的问题。
- 测试构建：unit/integration/doc tests、feature 组合测试、workspace 全量测试是否覆盖关键 crate。

## 怎么检查

1. 先读取 `Cargo.toml`、`Cargo.lock`、`rust-toolchain.toml`、`.cargo/config.toml`、Makefile/justfile、CI 配置，确认构建入口和工具链。
2. 如用户没有明确要求运行命令，只做静态诊断；`cargo update`、`cargo add`、`cargo fix` 会改状态，必须先确认。
3. 对错误日志优先识别 rustc error code、crate 名、feature 组合和 target，不要把级联错误当作多个根因。
4. 对 borrow checker 问题，先判断所有权边界和数据结构设计，不建议用 `clone` 或 `RefCell` 作为默认答案。
5. 对 feature 问题，检查 `default-features`、`features = []`、可选依赖和 workspace 继承，说明具体组合。
6. 对 native/build.rs 问题，检查平台、环境变量和系统库；需要安装系统依赖时必须标为无法直接修复。
7. 对 clippy 问题，区分真实 bug、风格建议和项目策略；不要建议无审批地添加 `#[allow]`。

## 常用搜索线索

- Cargo 配置：`[workspace]`、`members`、`[features]`、`default-features`、`optional = true`、`rust-version`、`edition`。
- 工具链：`rust-toolchain.toml`、`.cargo/config.toml`、`RUSTFLAGS`、`CARGO_TARGET_DIR`、`target.`。
- 构建脚本：`build.rs`、`pkg-config`、`bindgen`、`cc::Build`、`rerun-if-changed`。
- 平台条件：`#[cfg(`、`cfg_attr`、`target_os`、`target_arch`、`target_env`、`wasm32`。
- 常见错误：`E0502`、`E0597`、`E0382`、`unresolved import`、`cannot find macro`、`failed to select a version`、`linking with`。
- 质量门禁：`cargo fmt`、`cargo clippy`、`deny(warnings)`、`#![allow]`、`unsafe`。

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
- 涉及命令、crate、feature 或 target
- 根因判断
- 风险原因
- 最小修复建议
- 无法确认时的下一步验证建议

## 内置规则覆盖

- 诊断顺序：先看 `Cargo.toml`、`Cargo.lock`、workspace、`rust-toolchain.toml`、`.cargo/config.toml`，再按 error code 定位根因。
- 常见错误：`E0502`、`E0597`、`E0382`、unresolved import、cannot find macro、failed to select a version、linking failed。
- 借用检查：优先调整所有权边界和作用域；不要默认建议 `clone`、`RefCell` 或 `unsafe` 绕过设计问题。
- Feature：检查 `default-features`、optional dependency、workspace dependency、target-specific dependency 和 feature 组合测试。
- Native/build.rs：`pkg-config`、`bindgen`、`cc::Build`、系统库、环境变量和 target triple 要一起判断。
- 停止条件：`cargo update`、`cargo add`、`cargo fix`、安装系统库或改 MSRV 会改变项目状态，执行前必须确认。
