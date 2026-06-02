---
name: rust-guidelines
description: Rust 编码规范——写 Rust 代码时的 clippy lint 基线、错误处理、并发与共享状态模式、所有权设计、unsafe 边界、模块组织、测试规范和 Cargo 约定。适用于所有 Rust 项目（库、二进制、嵌入式、Wasm）。
license: MIT
metadata:
  tags: rust, coding-standards, clippy, concurrency, error-handling, ownership, cargo
---

# Rust 编码规范

Rust 的安全机制可以防止数据竞争和内存错误，但 **不合理的使用模式会削弱这些保障**。以下规则在"零成本抽象"和"可维护性"之间划定边界。

## 1. clippy 与 lint 基线

**所有项目必须运行 `cargo clippy -- -D warnings` 且零告警通过。** 这是最低要求，不可协商。

在 `Cargo.toml` 或 `lib.rs`/`main.rs` 顶部声明项目级 lint：

```rust
// lib.rs / main.rs 顶部
#![deny(
    clippy::all,
    clippy::pedantic,
    clippy::unwrap_used,        // 禁止裸 unwrap()
    clippy::expect_used,        // 禁止裸 expect()
    clippy::panic,              // 禁止 panic!()
    missing_docs,               // 公开 API 必须有文档
    unsafe_code,                // 禁止 unsafe 代码（库项目）
    missing_debug_implementations,
    trivial_casts,
    trivial_numeric_casts,
    unused_import_braces,
    unused_qualifications
)]
```

**例外放宽**（仅适用于二进制/应用项目）：

```rust
// 二进制项目中可放宽：
#![allow(clippy::unwrap_used, clippy::expect_used)]
// 测试中完全允许：
#[cfg(test)]
#![allow(clippy::unwrap_used, clippy::panic)]
```

**正确**：用 `?` 传播错误，避免 `unwrap`。

```rust
fn load_config(path: &str) -> Result<Config, ConfigError> {
    let content = std::fs::read_to_string(path)?;      // 传播 IO 错误
    let config: Config = toml::from_str(&content)?;     // 传播解析错误
    Ok(config)
}
```

**错误**：用 `unwrap` 隐藏错误路径。

```rust
fn load_config(path: &str) -> Config {
    let content = std::fs::read_to_string(path).unwrap();  // 文件不存在 → panic
    toml::from_str(&content).unwrap()                      // 解析失败 → panic
}
```

## 2. 错误处理

### 2.1 thiserror vs anyhow 的选择

**库项目用 `thiserror`，二进制/应用项目用 `anyhow`。**

| 项目类型 | 推荐 crate | 理由 |
|---------|:---------:|------|
| 库 (`lib.rs`) | `thiserror` | 调用方需要匹配具体错误变体 |
| 二进制 (`main.rs`) | `anyhow` | 最终只需向上传播，不需要细粒度匹配 |
| 混合项目 | `thiserror` 在 `lib.rs` + `anyhow` 在 `main.rs` | 各司其职 |

**正确（库项目）**：

```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum StorageError {
    #[error("Key not found: {0}")]
    NotFound(String),

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),

    #[error("Serialization failed: {0}")]
    Serde(#[from] serde_json::Error),
}

// 必须为 Tauri 命令实现 Serialize
impl serde::Serialize for StorageError {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where S: serde::Serializer {
        serializer.serialize_str(&self.to_string())
    }
}
```

**正确（二进制项目）**：

```rust
use anyhow::{Context, Result};

fn main() -> Result<()> {
    let config = load_config("config.toml")
        .context("Failed to load application config")?;
    // ...
    Ok(())
}
```

**错误**：在库项目中使用 `anyhow`。

```rust
// 库调用方无法匹配具体错误类型
pub fn get_user(id: u64) -> anyhow::Result<User> { /* ... */ }
```

### 2.2 禁止 `String` 作为错误类型

**永远不要用 `Result<T, String>` —— 调用方无法区分错误，且丢失回溯信息。**

```rust
// ❌ 坏实践：String 错误
fn read_config() -> Result<Config, String> {
    std::fs::read_to_string("config.toml").map_err(|e| e.to_string())
}

// ✅ 好实践：自定义错误类型
fn read_config() -> Result<Config, ConfigError> {
    let content = std::fs::read_to_string("config.toml")?;
    Ok(toml::from_str(&content)?)
}
```

### 2.3 `?` 操作符传播

**能用 `?` 的地方不使用 `match` 或 `if let Err` 手动展开。** 但需确保错误类型可转换（实现了 `From` trait）。

```rust
// ✅ ? 链式传播
fn process() -> Result<Output, AppError> {
    let raw = fetch_data()?;
    let parsed = parse(&raw)?;
    let output = transform(parsed)?;
    Ok(output)
}
```

**错误**：手动 match 展开——冗长且容易遗漏错误路径。

```rust
fn process() -> Result<Output, AppError> {
    let raw = match fetch_data() {
        Ok(v) => v,
        Err(e) => return Err(e.into()),
    };
    let parsed = match parse(&raw) {
        Ok(v) => v,
        Err(e) => return Err(e.into()),
    };
    // ... 每步都要重复 match，层层嵌套
}
```

## 3. 并发与共享状态

### 3.1 LazyLock 替代 once_cell

**Rust 1.80+ 使用 `std::sync::LazyLock`，不要引入 `once_cell` 或 `lazy_static`。**

（来源：[`std::sync::LazyLock` 官方文档](https://doc.rust-lang.org/std/sync/struct.LazyLock.html)，[once_cell 作者声明](https://docs.rs/once_cell)—— "Most of this crate's functionality is available in `std` starting with Rust 1.70"）

```rust
// ✅ Rust 1.80+ 标准库方案
use std::sync::LazyLock;

static SETTINGS: LazyLock<Settings> = LazyLock::new(|| {
    load_settings_from_disk().unwrap_or_default()
});
```

```rust
// ❌ 旧方案 —— 不应在新项目中使用
use once_cell::sync::Lazy;  // 外部依赖，标准库已有
use lazy_static::lazy_static; // 宏方案，已被替代
```

### 3.2 RwLock vs Mutex vs ArcSwap 选择

**根据读写比例和延迟要求选择正确的同步原语。**

| 类型 | 适用场景 | 读锁成本 | 写锁成本 |
|------|---------|:-------:|:-------:|
| `Mutex<T>` | 读写均衡，或临界区极短 | 互斥 | 互斥 |
| `RwLock<T>` | 读多写少（≥ 10:1） | 共享 | 互斥 |
| `ArcSwap<T>` | 读极高频、写极低频；读必须无锁 | **无锁** | 交换 Arc |

```rust
use std::sync::RwLock;

// ✅ 配置/设置 —— 读多写少，用 RwLock
static CONFIG: LazyLock<RwLock<AppConfig>> = LazyLock::new(|| {
    RwLock::new(AppConfig::default())
});

fn get_config() -> AppConfig {
    CONFIG.read().expect("Config lock poisoned").clone()
}
```

```rust
// Cargo.toml 中需添加：
// [dependencies]
// arc-swap = "1"

use arc_swap::ArcSwap;
use std::sync::Arc;

// ✅ 热路径高频读取 —— 用 ArcSwap 实现无锁读取
static CONFIG: LazyLock<ArcSwap<AppConfig>> = LazyLock::new(|| {
    ArcSwap::from_pointee(AppConfig::default())
});

fn get_config() -> Arc<AppConfig> {
    CONFIG.load_full()  // 无锁读取
}

fn update_config(new_config: AppConfig) {
    CONFIG.store(Arc::new(new_config));  // 原子替换 —— 无锁、不可失败
}
```

**选择决策树**：
```
读写比例 > 100:1 且读在热路径上 → ArcSwap
读写比例 > 10:1 → RwLock
其他 → Mutex
```

### 3.3 锁中毒处理

**永远不要对锁使用 `unwrap()`。** 使用 `expect()` 带原因说明。

锁中毒（poisoning）发生在持有锁的线程 panic 时。此时锁内数据的完整性无法保证：

| 处理方式 | 后果 | 评价 |
|---------|------|:--:|
| `.unwrap()` | 崩溃，信息为空 | ❌ 无法排查 |
| `.expect("msg")` | 崩溃，信息明确 | ✅ 推荐 — 中毒意味着程序状态已破坏，崩溃并重启是最诚实的做法 |
| `unwrap_or_else(\|e\| e.into_inner())` | 恢复锁，继续运行 | ⚠️ 数据完整性未知——恢复了一个已被 panic 破坏的状态 |
| 传播 `Err` 到前端 | 前端收到无法修复的错误 | ⚠️ 隐藏了需要重启的致命错误 |

> **为什么 `expect()` 是正确选择？** 锁中毒的唯一原因是一个 panic 在持有锁时 unwind。此时锁保护的数据可能处于中间状态（只写了一半）。恢复数据并继续运行，等于把损坏的状态传播到整个应用。`expect()` 发出清晰的错误消息后崩溃，让操作系统或进程管理器重启应用——这是最安全的行为。

```rust
// ✅ expect() 带原因说明
let config = CONFIG.read().expect("Config RwLock should never be poisoned");
```

**错误**：

```rust
let config = CONFIG.read().unwrap();  // panic 信息为空，难以排查
```

> ⚠️ **`LazyLock` 与 `Mutex` 的中毒行为不同**：`LazyLock` 初始化闭包 panic 后，锁**不可恢复**（所有后续访问都 panic）。因此 `LazyLock` 的初始化闭包必须处理所有可能 panic 的路径。

### 3.4 异步锁选择

**异步代码中跨 `.await` 持锁时，必须使用 `tokio::sync::Mutex`，而非 `std::sync::Mutex`。**

```rust
use tokio::sync::Mutex;

struct AppState {
    sessions: Mutex<HashMap<String, Session>>,
}

#[tauri::command]
async fn cleanup(state: tauri::State<'_, AppState>) -> Result<(), String> {
    let mut sessions = state.sessions.lock().await;  // tokio::Mutex
    // .await 在此之后出现是安全的
    let expired = find_expired_sessions().await;
    expired.iter().for_each(|id| { sessions.remove(id); });
    Ok(())
}
```

**错误**：

```rust
use std::sync::Mutex;  // 同步 Mutex 跨 .await 持有会死锁

async fn bad_cleanup(state: tauri::State<'_, Mutex<Vec<String>>>) {
    let mut data = state.lock().unwrap();
    some_async_fn().await;  // 死锁！标准库的 MutexGuard 不是 Send
    data.push("done".into());
}
```

## 4. 所有权与借用

### 4.1 Clone 成本控制

**只在所有权确实需要转移时才 Clone。** 满足以下任一条件方可 Clone：① 值需要被移动到新线程/异步任务；② 值需要同时被多个所有者持有；③ API 要求所有权（如 `tokio::spawn(async move { ... })`）。仅读取数据时使用借用（`&T`）。

```rust
// ✅ 借用 —— 只读不获取所有权
fn process(data: &[u8]) -> Output {
    // 不获取所有权
}
```

```rust
// ✅ 确实需要所有权时才 Clone
fn spawn_task(data: Vec<u8>) {
    tokio::spawn(async move {
        process_owned(data);  // 所有权转移到异步任务
    });
}

// ✅ 确实需要所有权时才 Clone
fn spawn_task(data: Vec<u8>) {
    tokio::spawn(async move {
        process_owned(data);  // 所有权转移到异步任务
    });
}
```

**错误**：

```rust
// 不必要的 Clone：明明可以借用
fn process(data: Vec<u8>) -> Output {
    let data = data.clone();  // 调用方已经传入了所有权
    // ...
}
```

### 4.2 Arc 使用规范

**`Arc` 是共享所有权的正确工具，但不要滥用。** Tauri 的 `manage()` 已经内部使用了 `Arc`，不要再次包裹。

```rust
// ✅ Tauri 中直接 manage，不需要额外的 Arc
app.manage(AppState::new());

// ❌ 多余的 Arc——Tauri 内部已处理
app.manage(Arc::new(AppState::new()));
```

### 4.3 同步/异步上下文区分（Tauri 场景）

Tauri 应用中存在两种运行时上下文，**锁的选择取决于代码所在的上下文**：

| 上下文 | 运行时 | 应使用的锁 |
|--------|:-----:|-----------|
| `setup` 钩子 | **同步** | `std::sync::Mutex` / `std::sync::RwLock` |
| `#[tauri::command]` | **异步 (tokio)** | 不跨 `.await` 时可用 `std::sync`；跨 `.await` 必须用 `tokio::sync::Mutex` |
| `tokio::spawn` 内部 | **异步** | `tokio::sync::Mutex` |
| `std::thread::spawn` 内部 | **同步** | `std::sync::Mutex` |

```rust
// ✅ setup 钩子中 —— 同步上下文，std::sync 原语正确
tauri::Builder::default()
    .setup(|app| {
        // 同步代码块
        let state = std::sync::Mutex::new(AppState::default());
        app.manage(state);                    // manage() 在 setup 中安全
        Ok(())
    })
```

```rust
// ✅ 命令中 —— 异步上下文，不跨 .await 可用 std::sync
#[tauri::command]
async fn quick_read(state: tauri::State<'_, std::sync::RwLock<Config>>) -> Result<String, String> {
    let config = state.read().expect("Config lock poisoned");
    Ok(config.version.clone())   // 读后立即释放 —— 不跨 .await
}
```

**错误**：在异步命令中用 `std::sync::Mutex` 跨 `.await`。

```rust
#[tauri::command]
async fn bad_blocking(state: tauri::State<'_, std::sync::Mutex<Vec<String>>>) -> Result<(), String> {
    let mut data = state.lock().expect("Lock poisoned");
    // 锁在此处被持有
    tokio::time::sleep(Duration::from_secs(1)).await;  // 死锁！
    data.push("done".into());
    Ok(())
}
```

> ⚠️ **禁止在异步上下文中使用 `block_on`**：`tokio::runtime::Handle::block_on()` 在已进入 tokio runtime 的上下文中调用会导致 panic。Tauri 命令中调用任何 `.block_on()` 方法都将崩溃。

> ⚠️ **`Send + Sync` 约束**：`tokio::spawn` 中的闭包必须是 `Send`。`std::sync::MutexGuard` 不是 `Send`，因此不能在 `.await` 期间持有。

## 5. 模块与可见性

### 5.1 lib.rs 是公共 API 边界

**`lib.rs` 中 `pub` 导出的内容构成库的公共 API。** 内部实现细节永远设为 `pub(crate)` 或私有。

**正确**：

```rust
// lib.rs
pub mod commands;    // 公开模块
mod state;           // 内部模块（不公开）
mod utils;           // 内部工具

// 重新导出关键类型
pub use state::AppState;
```

**错误**：将所有类型设为 `pub`，暴露内部实现细节。

```rust
// lib.rs —— 调用方可以看到并依赖内部实现
pub mod commands;
pub mod state;       // state 模块内部结构暴露给所有调用方
pub mod utils;
pub use state::*;    // 通配符重导出 —— API 变更时破坏所有调用方
```

### 5.2 模块文件组织

```rust
// ✅ 推荐：2018 edition 风格，mod.rs 或同名文件
src/
├── lib.rs
├── main.rs
├── commands/
│   ├── mod.rs       // 或 commands.rs
│   ├── users.rs
│   └── files.rs
└── models/
    ├── mod.rs
    └── user.rs
```

## 6. unsafe 代码

### 6.1 unsafe 最小化原则

**`unsafe` 块必须尽可能小，并用 SAFETY 注释解释不变量。**

```rust
// ✅ 最小 unsafe 块 + 安全注释
/// # Safety
///
/// `ptr` 必须非空、对齐正确、指向已初始化的 `T`。
unsafe fn read_value<T>(ptr: *const T) -> T {
    // SAFETY: 调用方确保 ptr 非空且指向已初始化的 T
    unsafe { ptr.read() }
}
```

**错误**：

```rust
// ❌ unsafe 块范围过大，无安全注释
unsafe {
    let x = ptr.read();
    let y = ptr.add(1).read();
    process(x, y);
}
```

### 6.2 库项目必须 `#[deny(unsafe_code)]`

**发布给他人使用的库项目，必须在 `lib.rs` 中声明 `#![deny(unsafe_code)]`。** 如果确实需要 `unsafe`，将其隔离在独立模块中并加 `#[deny(unsafe_code)]` 豁免标记。

## 7. 测试

### 7.1 单元测试放在同一文件

**单元测试使用 `#[cfg(test)] mod tests` 放在被测代码同一文件中。**

```rust
// src/parser.rs
pub fn parse(input: &str) -> Result<Ast, ParseError> { /* ... */ }

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parse_simple_expression() {
        let result = parse("1 + 2").unwrap();
        assert_eq!(result.nodes.len(), 2);
    }

    #[test]
    fn parse_invalid_input_returns_error() {
        assert!(parse("").is_err());
    }
}
```

### 7.2 集成测试放在 tests/ 目录

**跨模块、端到端测试放在项目根目录 `tests/` 下。**

## 8. Cargo 约定

### 8.1 语义化版本

严格遵循 SemVer。`0.x.y` 阶段的 breaking change 仅递增 `y`。

### 8.2 Feature 命名

Feature 名称用小写 + 连字符。避免使用动词，优先使用名词或模块名。

```toml
[features]
default = ["json", "config-json5"]
json = ["serde_json"]
config-json5 = ["tauri/config-json5"]
tray-icon = ["tauri/tray-icon"]
```

### 8.3 Release profile

```toml
[profile.release]
opt-level = "s"         # 按大小优化
strip = "debuginfo"     # 剥离调试符号
lto = "thin"            # 平衡编译时间和性能
panic = "abort"         # 减小二进制体积
codegen-units = 1       # 最大化跨模块内联
```

## 9. serde 序列化模式

### 9.1 重命名字段

**用 `#[serde(rename_all = "...")]` 统一转换风格，而非逐个字段 rename。**

```rust
#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct UserInfo {
    pub user_name: String,    // Rust: snake_case → JSON: "userName"
    pub first_name: String,   // → "firstName"
    pub created_at: String,   // → "createdAt"
}
```

### 9.2 默认值与向后兼容

**使用 `#[serde(default)]` 和自定义默认函数，确保新增字段不影响旧数据反序列化。**

```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct Config {
    pub version: u32,

    #[serde(default = "default_timeout")]
    pub timeout_ms: u64,       // 新增字段——旧配置文件中不存在也能加载

    #[serde(default)]          // 使用类型的 Default 实现
    pub enable_logging: bool,
}

fn default_timeout() -> u64 { 5000 }
```

## 10. 编程范式

### 10.1 枚举驱动控制流

**用带数据的枚举 + `match` 替代字符串匹配和 `if-else` 链。** Rust 的 `match` 提供编译时穷尽检查，添加新变体时编译器会强制更新所有分支。

**正确**：

```rust
enum AppCommand {
    Start { config: StartConfig },
    Stop { reason: Option<String> },
}

fn handle(cmd: AppCommand) -> Result<(), String> {
    match cmd {
        AppCommand::Start { config } => start(config),
        AppCommand::Stop { reason } => stop(reason),
    }
}
```

**错误**：字符串驱动——运行时错误，编译器无法检查完整性。

```rust
fn handle(cmd: &str, payload: &str) -> Result<(), String> {
    if cmd == "start" { start(payload) }
    else if cmd == "stop" { stop(payload) }
    else { Err(format!("Unknown command: {cmd}")) }  // 编译时无法发现遗漏
}
```

### 10.2 Newtype 模式

**给原始类型包裹语义标签，零运行时开销。** 防止不同类型 ID 互相误传。

**正确**：

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct UserId(u64);

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct SessionId(String);

fn get_user(id: UserId) -> Option<User> { /* ... */ }
fn get_session(id: SessionId) -> Option<Session> { /* ... */ }

// 调用方无法传错：
let user = get_user(UserId(42));        // ✅ 编译通过
// let session = get_session(UserId(42));  // ❌ 编译错误 —— 类型不匹配
```

**错误**：裸原始类型——运行时静默接受错误 ID。

```rust
fn get_user(id: u64) -> Option<User> { /* ... */ }
fn get_session(id: u64) -> Option<Session> { /* ... */ }

let session = get_session(user_id);  // 传了 UserId 也能编译，运行时行为未定义
```

### 10.3 Builder 模式

**复杂配置对象使用 Builder 模式渐进构造。** 与 Tauri 自身 API 风格一致（`TrayIconBuilder`、`WebviewWindowBuilder` 等）。

**正确**：

```rust
pub struct Config {
    host: String,
    port: u16,
    tls: bool,
}

pub struct ConfigBuilder {
    host: Option<String>,
    port: Option<u16>,
    tls: bool,
}

impl ConfigBuilder {
    pub fn new() -> Self { Self { host: None, port: None, tls: false } }

    pub fn host(mut self, host: impl Into<String>) -> Self {
        self.host = Some(host.into());
        self
    }

    pub fn port(mut self, port: u16) -> Self {
        self.port = Some(port);
        self
    }

    pub fn tls(mut self, enabled: bool) -> Self {
        self.tls = enabled;
        self
    }

    pub fn build(self) -> Result<Config, &'static str> {
        Ok(Config {
            host: self.host.ok_or("host is required")?,
            port: self.port.unwrap_or(443),
            tls: self.tls,
        })
    }
}

let config = ConfigBuilder::new().host("localhost").tls(true).build().unwrap();
```

**错误**：必需参数和可选参数混在构造函数中——调用方必须记住参数位置。

```rust
let config = Config::new("localhost", 443, true, false, 30, None);
// port 和 tls 记住了，但后面三个参数是什么？
```

### 10.4 From/Into 转换

**使用 `From`/`Into` trait 统一跨层数据转换入口。** 避免散落各处的 `as`、手动构造和临时的 `map` 闭包。

**正确**：

```rust
struct CommandParams { cmd: String, payload: serde_json::Value }

impl From<CommandParams> for AppCommand {
    fn from(params: CommandParams) -> Self {
        match params.cmd.as_str() {
            "start" => AppCommand::Start {
                config: serde_json::from_value(params.payload).unwrap_or_default(),
            },
            _ => AppCommand::Unknown,
        }
    }
}

fn handle(params: CommandParams) -> Result<(), String> {
    let cmd: AppCommand = params.into();  // 统一转换入口
    process(cmd)
}
```

**错误**：转换逻辑散布各处，重复且容易不一致。

```rust
fn handle_a(raw: CommandParams) -> Result<(), String> {
    let cmd = match raw.cmd.as_str() { "start" => { /* 手写转换 */ } ... };
    // ...
}
fn handle_b(raw: CommandParams) -> Result<(), String> {
    let cmd = match raw.cmd.as_str() { "start" => { /* 手写转换 —— 与 handle_a 重复 */ } ... };
    // ...
}
```

### 10.5 组合与 Trait

Rust 没有继承，代码复用通过以下方式实现：

| 方式 | 用途 | 示例 |
|------|------|------|
| **组合**（字段嵌套） | 共享数据 | `AppState` 包含 `ConfigState` + `DeviceState` |
| **Trait + 默认实现** | 共享行为 | `trait Loggable { fn log(&self) { ... } }` |
| **扩展 Trait** | 为外部类型添加方法 | `trait WindowExt { fn hide_to_tray(&self); }` |

**正确**——组合嵌入：

```rust
struct DeviceState {
    connected: bool,
    device_name: String,
}

struct ConfigState {
    theme: String,
    auto_start: bool,
}

struct AppState {
    device: DeviceState,  // 组合嵌入
    config: ConfigState,  // 组合嵌入
}
```

**错误**——试图模拟继承（Rust 不支持，不应强行模拟）：

```rust
// 不要这样做：为了"复用字段"而创造深层嵌套的 struct 层级
struct BaseState { created_at: Instant }
struct DeviceState { base: BaseState, connected: bool }  // 一层
struct ExtendedDeviceState { device: DeviceState, extra: String }  // 两层 —— 难以追踪
```

### 10.6 命名约定

以下规则遵循 [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/naming.html) 和 `rustfmt` 默认行为：

| 元素 | 风格 | 示例 |
|------|------|------|
| 类型 / Trait / Enum | `UpperCamelCase` | `AppState`, `DeviceInfo`, `Read` |
| 函数 / 方法 / 变量 | `snake_case` | `get_user()`, `config_path` |
| 常量 / 静态变量 | `SCREAMING_SNAKE_CASE` | `MAX_RETRIES`, `DEFAULT_PORT` |
| Enum 变体 | `UpperCamelCase` | `AppCommand::Start`, `Color::BrightRed` |
| 模块 / Crate | `snake_case` | `user_service`, `tauri_plugin_log` |
| 构造方法 | `new()` / `with_*()` | `AppState::new()`, `Config::with_timeout(30)` |
| 转换方法 | `as_*()` 廉价；`to_*()` 昂贵；`into_*()` 消耗 | `as_bytes()`, `to_string()`, `into_bytes()` |
| Getter | `field()` | `config.port()` |
| Setter | `set_field()` | `config.set_port(8080)` |
| 布尔方法 | `is_*()` / `has_*()` | `device.is_connected()`, `map.has_key("x")` |
| 错误类型 | `*Error` 后缀 | `StorageError`, `ParseError` |
| Trait 名 | 动词（动作）、形容词（属性） | `Read`, `Write`（动作）；`Debug`, `Clone`（属性） |

```rust
// ✅ 符合 Rust 习惯
const MAX_CONNECTIONS: u32 = 100;
struct UserSession { user_id: String, created_at: Instant }
impl UserSession {
    fn new(user_id: String) -> Self { /* ... */ }
    fn is_expired(&self) -> bool { /* ... */ }
    fn as_summary(&self) -> SessionSummary { /* ... */ }  // 廉价转换
}
```

```rust
// ❌ 不符合 Rust 习惯
const maxConnections: u32 = 100;      // 驼峰风格，应为 SCREAMING_SNAKE
struct user_session { /* ... */ }      // struct 应为 UpperCamelCase
fn getIsExpired(&self) -> bool { }    // getIs* 应为 is_*
```

### 10.7 函数式与命令式权衡

**纯数据转换优先用 `Iterator` 组合子，有副作用或需提前退出时回退到 `for` 循环。**

| 场景 | 推荐风格 | 理由 |
|------|:------:|------|
| 纯数据转换（map/filter/collect） | 函数式 | 意图声明式、无可变状态、无副作用 |
| Option/Result 链 | 函数式 | `and_then`/`map` 消除塔形 `match` 缩进 |
| 步骤含 I/O 或 `?` 传播 | 命令式 | 闭包中 `?` 无法跨函数传播 |
| 需要 `break`/`return` 提前退出 | 命令式 | `try_for_each` 可读性差 |
| 复杂多字段累加器 | 命令式 | `fold` 闭包超过 3 行 → 可读性崩溃 |
| 性能敏感热路径 | 命令式 | `Iterator` 组合子展开优化不稳定 |

**正确（函数式 — 纯转换管道）**：

```rust
// ✅ 声明式：每个步骤是一个纯函数，意图一目了然
let active_users: Vec<String> = users
    .iter()
    .filter(|u| u.is_active)
    .map(|u| u.name.clone())
    .collect();
```

**正确（命令式 — 有副作用）**：

```rust
// ✅ 命令式：每个步骤有 I/O 或需要 ? 传播
let mut results = Vec::new();
for user in &users {
    let data = fetch_user_data(&user.id).await?;  // I/O + ? 传播
    if data.is_valid() {
        results.push(data);
    }
}
```

**错误**：在有副作用时强行用函数式。

```rust
// ❌ 闭包中无法用 ? 传播，只能 unwrap 或手动 match
let results: Vec<_> = users
    .iter()
    .map(|u| fetch_user_data(&u.id).await.unwrap())  // 闭包中 ? 不可用
    .filter(|d| d.is_valid())
    .collect();
```

**Option/Result 链 — 函数式优先**：

```rust
// ✅ and_then 链 —— 无嵌套
fn get_city(user: Option<&User>) -> Option<&str> {
    user.and_then(|u| u.address.as_ref())
        .map(|addr| addr.city.as_str())
}

// ❌ match 塔 —— 每层增加一级缩进
fn get_city(user: Option<&User>) -> Option<&str> {
    match user {
        Some(u) => match u.address.as_ref() {
            Some(addr) => Some(addr.city.as_str()),
            None => None,
        },
        None => None,
    }
}
```

## 11. 快速参考

| 规则 | 核心要求 | 主要例外 |
|------|---------|---------|
| clippy | `cargo clippy -- -D warnings` 零告警 | 测试模块可放宽 unwrap |
| 错误类型 | 库用 `thiserror`，二进制用 `anyhow` | 无 |
| `String` 错误 | 永远禁止 | 无 |
| 全局懒加载 | `std::sync::LazyLock` | Rust < 1.80 用 `once_cell` |
| 读多写少 | `RwLock`；极端热路径用 `ArcSwap` | 读写均衡用 `Mutex` |
| 锁中毒 | 必须 `expect()` + 说明原因 | 无 |
| 异步锁 | 跨 `.await` 用 `tokio::sync::Mutex` | setup 钩子中用 `std::sync` |
| 异步上下文 | 命令中禁止 `block_on`；`MutexGuard` 不可跨 `.await` | 纯同步代码 |
| Arc 包裹 | Tauri `manage()` 不需要额外 `Arc` | 手动线程管理需要 |
| 枚举控制流 | `match` + 数据枚举替代 `if-else` 链 | 简单布尔分支 |
| 函数式 vs 命令式 | 纯转换用 `Iterator` 组合子，有副作用/I/O 用 `for` | `fold` 闭包超 3 行回退命令式 |
| Newtype | 给原始类型加语义标签 | 仅函数内部使用的临时值 |
| unsafe | 最小块 + SAFETY 注释 | 无 |
| 命名 | 类型 CamelCase、函数 snake_case、常量 SCREAMING | 无 |

## 12. 禁止事项

- 禁止使用 `unwrap()` 和裸 `expect()` 在生产代码中（除非用 `#[deny(clippy::unwrap_used)]` 追踪豁免）
- 禁止使用 `String` 作为 `Result` 的错误类型
- 禁止在库项目中使用 `anyhow`（应使用 `thiserror`）
- 禁止在 `LazyLock` 初始化闭包中 panic（不可恢复的中毒）
- 禁止用 `std::sync::Mutex` 跨 `.await` 持锁
- 禁止引入 `once_cell` 或 `lazy_static` 依赖（Rust 1.80+ 有 `LazyLock`）
- 禁止 `unsafe` 块超出最小必要范围
- 禁止 `unsafe` 块无 SAFETY 注释
- 禁止在 Tauri 中手动 `Arc::new()` 包裹 `manage()` 的状态

## 13. 自检要求

输出 Rust 代码或配置前，逐项检查：

- [ ] `cargo clippy -- -D warnings` 是否零告警通过？
- [ ] 错误类型是否选择了正确的 crate（库→thiserror，二进制→anyhow）？
- [ ] 是否有 `String` 作为错误类型？
- [ ] 全局懒加载是否使用了 `std::sync::LazyLock`（非 `once_cell`）？
- [ ] 读多写少场景是否用了 `RwLock` 或 `ArcSwap`（非 `Mutex`）？
- [ ] 锁访问是否使用 `expect()` 带原因说明？
- [ ] 异步代码中跨 `.await` 是否用了 `tokio::sync::Mutex`（非 `std::sync`）？
- [ ] setup 钩子（同步）和命令（异步）中的锁选择是否正确？
- [ ] 是否禁止在异步上下文中使用 `block_on`？
- [ ] `manage()` 的状态是否不需要额外的 `Arc` 包裹？
- [ ] 控制流是否用枚举 + `match` 替代了字符串匹配？
- [ ] 纯数据转换是否用 `Iterator` 组合子（非 `for` + `push`）？有副作用时是否正确地回退到命令式？
- [ ] 是否用 Newtype 包裹了易混淆的基础类型（ID 等）？
- [ ] 跨层转换是否通过 `From`/`Into` 而非散落的 `map` 闭包？
- [ ] `unsafe` 块是否最小化且有 SAFETY 注释？
- [ ] 命名：类型 CamelCase、函数 snake_case、常量 SCREAMING、布尔方法 is_*/has_*？
- [ ] 新增 serde 字段是否有 `#[serde(default)]`？
- [ ] 单元测试是否放在被测代码的同文件中（`#[cfg(test)]`）？
