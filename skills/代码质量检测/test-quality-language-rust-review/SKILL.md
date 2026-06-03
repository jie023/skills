---
name: "test-quality-language-rust-review"
description: "Use when reviewing Rust code quality, ownership, borrowing, tokio, Axum, error handling, concurrency, or idiomatic safety."
---
# Rust 语言专项质量审查

## 定位

Rust 专项代码质量审查。用于审查 Rust 项目的所有权、借用、生命周期、unsafe、panic、错误处理、并发安全、异步运行时、Clippy、测试和依赖安全风险。

## 内置规则覆盖

- Rust 规范：所有权、借用、生命周期、错误处理、模块边界、trait 设计、feature 和依赖安全都纳入审查。
- 安全边界：`unsafe`、FFI、裸指针、`MaybeUninit`、`transmute`、`unsafe impl Send/Sync` 必须有可验证不变量和 `SAFETY` 说明。
- panic 风险：生产路径中的 `unwrap`、`expect`、`panic!`、`todo!`、`unimplemented!` 需要判断是否可被外部输入触发。
- 异步并发：锁跨 `await`、阻塞操作进入 runtime、channel 背压、task 泄漏、取消和超时处理必须检查。
- 工具质量：Clippy、rustfmt、cargo test、RUSTSEC、cargo deny/audit 是可建议验证手段，不默认运行。

## 检查重点

- 所有权、借用和生命周期：不必要 `clone`、过度 `'static`、生命周期标注错误、`Rc`/`RefCell` 滥用、循环引用、返回悬垂引用风险。
- `unsafe`：必要性、边界范围、`SAFETY` 注释、不变量维护、FFI 指针合法性、`unsafe impl Send/Sync` 正确性、`transmute`/裸指针/`MaybeUninit` 使用。
- panic 风险：生产代码中的 `unwrap`、`expect`、`panic!`、`todo!`、`unimplemented!`，以及可由外部输入触发的断言失败。
- `Result` 和错误处理：错误类型边界、`?` 传播、上下文保留、`thiserror`/`anyhow` 使用边界、错误吞噬、日志泄漏敏感信息。
- Send/Sync 和并发：锁顺序、锁跨 `await`、阻塞操作进入 async runtime、`tokio::spawn` 的 `Send + 'static` 约束、channel 背压和关闭处理。
- tokio/Axum/Actix 等服务端场景：请求输入校验、超时、取消、阻塞 I/O、共享状态、handler 错误映射。
- Clippy、rustfmt、cargo test：是否有 lint、格式、单元测试、集成测试、异步测试、失败用例覆盖。
- 依赖安全：`Cargo.toml`/`Cargo.lock` 依赖版本、默认 features、未使用依赖、RUSTSEC 漏洞、重复或过时 crate。
- 性能和内存：不必要分配、过早 `collect`、过度 `String`/`Vec` 拷贝、锁粒度、iterator 可读性和大函数。

## 检查流程

1. 确认范围：优先识别 `Cargo.toml`、`Cargo.lock`、`src/**/*.rs`、`tests/**/*.rs`、`benches/**/*.rs`，以及当前任务明确要求审查的文件。
2. 按内置规则读取 Rust 源码、Cargo 配置和测试，只依据当前项目代码和可验证证据判断。
3. 搜索线索：用关键词定位高风险代码，再读取上下文，不要只凭单行匹配下结论。
4. 分层审查：先查阻断和高风险，再查中风险和建议优化。涉及安全、并发、unsafe、panic 的问题优先确认。
5. 工具验证：项目环境允许时运行 `cargo fmt --check`、`cargo clippy --all-targets --all-features -- -D warnings`、`cargo test --all --all-features`。依赖安全工具存在时运行 `cargo audit` 或 `cargo deny check`。
6. 记录限制：如果无法运行命令、缺少依赖、需要联网或环境不完整，报告中说明原因和替代复核方式。

## 风险分级

### 阻断

- 存在可导致未定义行为、内存破坏、数据竞争或越权访问的 `unsafe` 问题。
- `unsafe impl Send/Sync` 缺少可验证不变量，或共享状态可能跨线程不安全。
- 可由外部输入触发的生产路径 panic、进程退出、服务不可用。
- 硬编码凭证、SQL 拼接注入或对外部输入执行不安全反序列化，可能导致密钥泄露、数据泄露或远程风险。
- 依赖存在已知严重安全漏洞，或关键安全依赖版本不可接受。
- `cargo test`、`cargo clippy -D warnings` 在相关范围失败，且失败与本次审查范围相关。

### 高风险

- 生产代码中对外部输入、I/O、网络、数据库、配置解析使用 `unwrap`/`expect`。
- 数据库查询、命令参数、文件路径、反序列化格式或 Web handler 未对外部输入做白名单、参数绑定、大小限制和格式校验。
- 错误处理丢失上下文、吞掉错误、错误类型穿透层级，导致调用方无法正确处理。
- async 代码中持有 `std::sync::Mutex`/`RwLock` 跨 `await`，或在 runtime 线程执行阻塞 I/O/CPU 密集任务。
- 生命周期或所有权设计绕开编译器约束，例如滥用 `Box::leak`、`static mut`、`RefCell` 运行时借用、`Rc` 循环引用。
- 缺少关键路径测试，尤其是错误分支、并发分支、输入校验和 unsafe 边界。

### 中风险

- 不必要 `clone`、`to_owned`、`collect`、堆分配或锁粒度过大，造成明确性能或内存压力。
- 公共 API 缺少文档、错误枚举不稳定、命名不符合 Rust API Guidelines。
- Clippy 普通告警未处理，或 lint 被宽泛 `allow` 掩盖且没有理由。
- 输入校验不完整，但暂未发现直接安全后果。
- 测试只覆盖成功路径，缺少边界值、失败路径或异步超时场景。

### 建议优化

- 可用借用、`Cow`、iterator、模式匹配、`From`/`TryFrom` 改善可读性和惯用性。
- 可拆分过长函数或模块，减少重复错误映射和重复校验。
- 可收紧依赖 features、移除未使用依赖、补充 `cargo deny`/`cargo audit` 到 CI。
- 可补充 `rustfmt`、Clippy、测试命令的本地或 CI 说明。

## 怎么检查

### 所有权、借用和生命周期

- 检查 `clone`、`to_string`、`to_owned`、`collect` 是否只是为绕过借用而引入，优先确认是否可用引用、iterator、`Cow` 或更清晰的数据边界。
- 检查函数签名是否暴露不必要生命周期，是否用 `'static` 掩盖真实生命周期。
- 检查 `Rc`、`Arc`、`RefCell`、`Mutex` 的组合是否合理，循环引用应考虑 `Weak`。
- 检查 `Pin`、自引用结构、裸指针缓存、全局状态是否有清晰不变量。

### unsafe

- 每个 `unsafe` 块都要说明为什么安全，复杂场景应有中文或项目既有语言风格的 `SAFETY` 注释。
- 确认 unsafe 范围尽量小，安全封装的 public API 不应把不变量负担转嫁给普通调用方。
- FFI 要检查空指针、对齐、生命周期、所有权释放、字符串编码、线程回调和 ABI。
- 对 `transmute`、`from_raw`、`into_raw`、`MaybeUninit`、`assume_init`、`get_unchecked`、`static mut` 必须逐项验证前置条件。

### panic、unwrap 和 expect

- 测试代码、示例代码、构建脚本中的 `unwrap`/`expect` 可按语境判断；生产运行路径默认视为风险。
- 启动期不可恢复配置错误可以允许 `expect`，但错误信息必须明确且不能泄漏密钥。
- 外部输入、网络、数据库、文件、时间、序列化、锁获取等失败点应返回 `Result` 或映射为业务错误。

### Result 和错误处理

- 库层优先返回具体错误类型，应用边界可使用 `anyhow` 聚合上下文。
- `thiserror` 适合定义领域错误，`anyhow::Context` 适合补充调用上下文。
- 不应把所有错误压成字符串或 `Box<dyn Error>` 后丢失可匹配性。
- 日志要保留排查信息，但避免输出 token、密码、密钥、连接串等敏感数据。

### 安全输入边界

- 检查配置、测试夹具、示例代码和默认值中是否硬编码 token、密码、私钥、连接串或内部地址。
- SQLx、Diesel、SeaORM、tokio-postgres 等数据库访问必须使用参数绑定；动态排序、列名、表名必须白名单。
- serde、bincode、ron、yaml、json 等反序列化输入要限制来源、大小、格式和错误处理，不对不可信输入直接构造危险对象。
- 文件路径、URL、命令参数和重定向目标必须做协议、根目录、域名或枚举白名单校验。

### Send、Sync 和并发

- `tokio::spawn` 捕获值需要满足 `Send + 'static`，不要通过泄漏或全局变量粗暴规避。
- async 代码优先使用 `tokio::sync` 锁；使用 `std::sync` 锁时必须确认不会跨 `await` 且临界区很短。
- 多锁场景检查锁顺序和提前释放，避免死锁。
- channel 要检查容量、关闭、发送失败、接收退出和背压处理。
- CPU 密集或阻塞 I/O 应使用 `spawn_blocking`、专用线程池或同步边界隔离。

### Clippy、测试和依赖安全

- Clippy 告警不能只看数量，要确认是否暴露 correctness、panic、perf、suspicious、nursery 中的真实风险。
- `cargo test` 要关注测试失败、未编译 target、feature 组合、异步测试 runtime、数据库或网络依赖。
- 依赖安全检查优先查看 `Cargo.lock` 的实际解析版本，再结合 `cargo audit`、`cargo deny check` 或项目现有安全扫描。
- 新增依赖要检查维护状态、许可证、默认 features、传递依赖、是否已有项目内替代。

## 搜索线索

Codex 优先使用 `rg`，搜索结果需要包含文件路径、行号和匹配内容。常用线索：

```powershell
rg -n "unsafe|unsafe impl|unwrap\(|expect\(|panic!|todo!|unimplemented!|assert!\(|debug_assert!\(" src tests benches
rg -n "transmute|from_raw|into_raw|as_ptr|as_mut_ptr|MaybeUninit|assume_init|get_unchecked|static mut|Box::leak" src tests benches
rg -n "Arc<|Rc<|RefCell<|Cell<|Mutex<|RwLock<|OnceLock|lazy_static|thread_local|spawn\(|spawn_blocking|block_on" src tests benches
rg -n "clone\(|to_owned\(|to_string\(|collect::<Vec|Box<dyn Error>|map_err|context\(" src tests benches
rg -n "password|secret|token|api_key|private_key|sqlx::query|format!\(|serde_json::from_str|bincode|yaml|Command::new|PathBuf|canonicalize" src tests benches
rg -n "deny|allow|warn|forbid|clippy|features|default-features|version|git =" Cargo.toml Cargo.lock
```

如果项目不是标准目录结构，先用 `rg --files` 找 `.rs`、`Cargo.toml`、`Cargo.lock`，再按实际路径替换搜索范围。

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
- 如果用户明确要求修复代码，先确认影响范围；涉及多文件修改时按 `safe-bulk-editing` 执行。
- 不把工具告警机械等同于问题，必须结合代码上下文说明可触发条件和影响。
- 不能确认的问题降级为待验证项，说明缺失信息和下一步验证命令。

## 输出要求

按阻断、高风险、中风险、建议优化分组。无法确认的问题要说明原因和下一步验证建议。推荐格式：

```markdown
# Rust 代码质量审查报告

## 阻断

- [文件:行号] 问题标题
  - 证据：引用关键代码片段或搜索命中。
  - 影响：说明可触发条件、运行后果和风险等级理由。
  - 建议：给出可执行修复方向。
  - 验证：说明修复后应运行的命令或测试。

## 高风险

- 暂无。

## 中风险

- 暂无。

## 建议优化

- 暂无。

## 工具验证

- 已运行：列出 `cargo fmt --check`、`cargo clippy`、`cargo test`、`cargo audit` 或 `cargo deny check` 的结果。
- 未运行：说明原因，例如依赖缺失、需要联网、当前环境不完整。

## 待确认

- 说明需要用户或项目上下文确认的事项。
```

## 多平台兼容

- Codex：优先用 `rg`/`rg --files` 搜索，用 `Get-Content -Encoding UTF8` 读取中文文件，小范围编辑使用 `apply_patch`，不要默认执行 Git。
- Claude：优先用 `Grep`、`Glob`、`Read` 获取路径、行号和内容；需要修改时按用户确认和平台编辑工具执行。
- Trae：优先用 `SearchCodebase`、`Read` 获取上下文；需要修改时用平台编辑工具，避免未确认的批量替换。
- Windows：路径含中文或空格时使用正斜杠和 `-LiteralPath`，读写保持 UTF-8，复核时检查乱码、制表符和隐藏控制字符。
- Linux/macOS：命令示例可直接使用 `rg`、`cargo`、`cargo audit`、`cargo deny`；如果 shell 或工具版本不同，报告中说明替代命令。
