---
name: tauri-guidelines
description: Tauri v2 桌面应用开发规范——使用 Rust + 前端框架构建 Tauri 应用时的项目结构、命令系统、状态管理、权限模型、IPC 通信和编码准则。覆盖 Tauri v2 (2.x) 全部核心概念。
license: MIT
metadata:
  tags: tauri, desktop, rust, typescript, ipc, permissions, state-management
---

# Tauri v2 桌面应用开发规范

Tauri v2 相比 v1 有架构级变化：权限系统从 `allowlist` 重构为 Capabilities + Permissions 细粒度 ACL 模型，IPC 新增 Channel 支持流式数据传输，多 WebView 支持，移动端支持（iOS/Android）。以下规则基于 Tauri v2 官方文档和实战经验编写，确保应用安全、可维护且符合生态范式。

## 1. 项目结构

**Tauri v2 项目必须遵循以下标准结构：**

```
my-app/
├── src/                    # 前端源码
├── src-tauri/              # Rust 后端
│   ├── capabilities/       # 权限能力配置（JSON 或 TOML）
│   │   ├── default.json   # 默认能力（所有平台）
│   │   └── desktop.json   # 桌面平台特定能力
│   ├── permissions/        # 自定义权限定义（可选）
│   ├── src/
│   │   ├── lib.rs          # 应用入口（run() 函数）
│   │   ├── main.rs         # 桌面端入口（仅调用 lib::run()）
│   │   ├── commands/       # Tauri 命令模块
│   │   │   ├── mod.rs
│   │   │   └── *.rs
│   │   ├── state/          # 状态管理
│   │   │   ├── mod.rs
│   │   │   └── *.rs
│   │   └── models/         # 数据结构定义
│   ├── icons/
│   ├── Cargo.toml
│   └── tauri.conf.json     # 或 tauri.conf.json5
├── package.json
└── index.html
```

**正确**：将命令拆分到独立模块。

```rust
// src-tauri/src/commands/mod.rs
mod file_ops;
mod system;

pub use file_ops::*;
pub use system::*;
```

**错误**：将所有命令写在 `main.rs` 或 `lib.rs` 中。

```rust
// main.rs 中塞入所有命令 —— 不可维护
#[tauri::command]
fn cmd1() {}
#[tauri::command]
fn cmd2() {}
// ... 数十个命令 ...
```

## 2. 配置参考（实战模式）

### 2.1 tauri.conf.json5（JSON5 格式）

Tauri v2 支持 JSON5 格式（允许注释、尾逗号）。需在 `Cargo.toml` 中启用 `config-json5` feature，配置文件命名为 `tauri.conf.json5`。

```json5
{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "my-app",
  "version": "0.1.0",
  "identifier": "com.example.my-app",
  "build": {
    "beforeDevCommand": "pnpm dev",
    "devUrl": "http://localhost:1420",
    "beforeBuildCommand": "pnpm build",
    "frontendDist": "../dist",
    "removeUnusedCommands": true
  },
  "app": {
    "withGlobalTauri": true,
    "windows": [],
    "security": {
      "csp": ""
    }
  },
  "bundle": {
    "createUpdaterArtifacts": true,
    "active": true,
    "targets": "all",
    "icon": [
      "icons/icon.ico",
      "icons/icon.icns",
      "icons/32x32.png"
    ],
    "windows": {
      "wix": { "language": ["zh-CN"] },
      "nsis": {
        "languages": ["English", "SimpChinese"],
        "displayLanguageSelector": true
      }
    }
  }
}
```

**关键配置项说明**：

| 配置项 | 作用 | 使用建议 |
|--------|------|---------|
| `removeUnusedCommands` | 编译时剔除未被 `generate_handler!` 引用的命令 | 始终开启 |
| `withGlobalTauri` | 将 `invoke`、`event` 等 API 注入到 `window.__TAURI__` | 模块化前端项目中可设为 `false` |
| `windows: []` | 不在静态配置中定义窗口 | 窗口在 Rust `setup` 中动态创建更灵活 |
| `csp: ""` | 清空 CSP 策略 | 仅开发期。生产环境应设置具体规则 |
| `createUpdaterArtifacts` | 构建时生成更新签名文件 | 使用 updater 时启用 |

### 2.2 capabilities 的"通用 + 平台特定"拆分模式

**将权限拆分为 `default.json`（通用）和 `desktop.json`（桌面专有）两个文件。**

```json
// src-tauri/capabilities/default.json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "Default capability for all windows",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "core:window:default",
    "core:window:allow-close",
    "core:window:allow-hide",
    "core:window:allow-minimize",
    "core:window:allow-start-dragging",
    "autostart:allow-enable",
    "autostart:allow-disable",
    "autostart:allow-is-enabled",
    "os:default"
  ]
}
```

```json
// src-tauri/capabilities/desktop.json
{
  "identifier": "desktop-capability",
  "platforms": ["macOS", "windows", "linux"],
  "windows": ["main"],
  "permissions": [
    "updater:default",
    "window-state:default"
  ]
}
```

**优势**：
- `default.json` 被所有平台共享（包括未来的 Android/iOS）
- `desktop.json` 只影响桌面端，移动端构建时自动排除
- 权限按职责分离，不会在单文件内膨胀

### 2.3 updater 配置（含镜像回退）

**配置多个 endpoints 实现下载源回退，国内用户可使用镜像加速。**

```json5
{
  "plugins": {
    "updater": {
      "pubkey": "<your-public-key>",
      "endpoints": [
        "https://mirror.example.com/releases/latest/download/latest.json",
        "https://github.com/user/repo/releases/latest/download/latest.json"
      ]
    }
  }
}
```

Rust 侧也可以根据用户 locale 动态切换镜像源：

```rust
#[tauri::command]
async fn check_update(app: AppHandle) -> Result<Option<UpdateInfo>, String> {
    let updater = app.updater_builder().build().map_err(|e| e.to_string())?;
    match updater.check().await {
        Ok(Some(mut update)) => {
            // 根据用户语言切换镜像
            let locale = get_locale();
            if locale.starts_with("zh") {
                let mirror_url = format!("https://mirror.example.com/{}", update.download_url);
                update.download_url = Url::parse(&mirror_url).map_err(|e| e.to_string())?;
            }
            // 缓存 update 对象供后续 perform_update 使用
            *UPDATE_CACHE.lock().expect("Update cache lock poisoned") = Some(update);
            Ok(Some(extract_info(update)))
        }
        Ok(None) => { /* ... */ Ok(None) }
        Err(e) => Err(e.to_string())
    }
}
```

### 2.4 日志插件配置（生产级）

```rust
.plugin(
    tauri_plugin_log::Builder::new()
        .target(tauri_plugin_log::Target::new(
            tauri_plugin_log::TargetKind::Folder {
                path: log_path,      // 日志输出到文件
                file_name: None,     // 自动命名
            },
        ))
        .level(log::LevelFilter::Debug)
        .timezone_strategy(tauri_plugin_log::TimezoneStrategy::UseLocal)
        .max_file_size(1024 * 128)                // 单个文件 128KB
        .rotation_strategy(tauri_plugin_log::RotationStrategy::KeepAll)
        .with_colors(
            fern::colors::ColoredLevelConfig::new()
                .error(fern::colors::Color::BrightRed)
                .warn(fern::colors::Color::Yellow)
                .info(fern::colors::Color::Green)
                .debug(fern::colors::Color::Blue)
        )
        .build(),
)
```

### 2.5 Cargo profile 优化（生产构建）

Tauri 桌面应用的二进制体积直接影响分发体验。`opt-level = "s"`（大小优化）而非 `"z"` 是桌面应用的推荐平衡点——`"z"` 在某些平台可能导致性能退化。

> 完整 Cargo profile 规范见 [rust-guidelines](../rust-guidelines/) §8.3。此处仅列 Tauri 特化要点。

```toml
[profile.release]
opt-level = "s"      # 大小优化 —— 桌面应用分发的推荐选择
strip = "debuginfo"  # 剥离调试符号
lto = "thin"         # 平衡编译时间和运行时性能
panic = 'abort'      # panic 直接 abort，避免 unwind 代码塞入二进制
codegen-units = 1    # 最大跨模块内联优化
```

## 3. 命令系统（Commands）

### 3.1 命令定义规范

**每个命令必须是 `async fn` 或同步 `fn`，返回 `Result<T, E>`（T 和 E 必须实现 `serde::Serialize`）。**

**正确**：

```rust
#[tauri::command]
async fn get_user(id: String) -> Result<User, String> {
    // async 操作（文件/网络/数据库）防止 UI 阻塞
    Ok(User { id, name: "Alice".into() })
}
```

**错误**：

```rust
#[tauri::command]
fn get_user_blocking(id: String) -> User {
    // 同步阻塞 —— UI 线程会被阻塞
    std::thread::sleep(Duration::from_secs(5));
    User { id, name: "Alice".into() }
}
```

### 3.2 错误处理

**必须使用自定义错误类型（thiserror/custom enum），而非 `String` 类型错误。**

**正确**：

```rust
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("File not found: {0}")]
    NotFound(String),
    #[error("Permission denied")]
    Forbidden,
    #[error(transparent)]
    Io(#[from] std::io::Error),
}

impl serde::Serialize for AppError {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where S: serde::Serializer {
        serializer.serialize_str(&self.to_string())
    }
}

#[tauri::command]
async fn read_config(path: String) -> Result<String, AppError> {
    let content = std::fs::read_to_string(&path)?; // 自动 ? 转换
    Ok(content)
}
```

**错误**：

```rust
// String 错误 —— 前端无法区分错误类型
#[tauri::command]
async fn read_config(path: String) -> Result<String, String> {
    std::fs::read_to_string(&path).map_err(|e| e.to_string())
}
```

### 3.3 命令注册规范

**`generate_handler!` 宏只调用一次，包含所有命令。模块路径作为前缀传入。**

```rust
tauri::Builder::default()
    .invoke_handler(tauri::generate_handler![
        // 顶层命令（无模块前缀）
        open_url,
        check_update,
        shutdown_app,

        // 子模块命令（模块路径前缀）
        device::query_all,
        device::connect,
        config::load_config,
        updater::check_update,
    ])
```

**注意**：子模块中的 `pub fn` 命令需要模块路径前缀（如 `controller::query_devices`），而非扁平展开。

## 4. 状态管理

### 4.1 只读状态

**不需要修改的配置/data 直接使用 `tauri::State<T>`，不包裹 `Mutex`。**

```rust
struct AppConfig {
    app_name: String,
    version: String,
}

#[tauri::command]
fn get_config(config: tauri::State<AppConfig>) -> &AppConfig {
    config.inner()
}
```

### 4.2 可变状态

**可变共享状态使用 `std::sync::Mutex<T>`。如果读多写少，使用 `std::sync::RwLock<T>`。异步命令需要在 `.await` 点之间持有锁时，使用 `tokio::sync::Mutex<T>`。**

**正确（同步命令）**：

```rust
use std::sync::Mutex;

struct AppState {
    counter: Mutex<u32>,
}

#[tauri::command]
fn increment(state: tauri::State<AppState>) -> u32 {
    let mut c = state.counter.lock().expect("Counter lock poisoned");
    *c += 1;
    *c
}
```

**正确（异步命令，需跨 await 持锁）**：

```rust
use tokio::sync::Mutex;

struct AsyncState {
    data: Mutex<Vec<String>>,
}

#[tauri::command]
async fn process(state: tauri::State<'_, AsyncState>) -> Result<(), String> {
    let mut data = state.data.lock().await; // tokio::sync::Mutex
    some_async_fn().await; // 安全 —— tokio Mutex 不要求 .await 时释放
    data.push("done".into());
    Ok(())
}
```

**错误**：

```rust
// std::sync::Mutex 的锁在 .await 时仍被持有 —— 死锁
#[tauri::command]
async fn bad_cmd(state: tauri::State<'_, std::sync::Mutex<Vec<String>>>) -> Result<(), String> {
    let mut data = state.lock().unwrap();
    some_async_fn().await; // 锁在整个 .await 期间被持有！死锁
    data.push("done".into());
    Ok(())
}
```

> **锁中毒处理**：所有对 `.lock()` / `.read()` / `.write()` 的调用必须使用 `.expect("原因")` 而非 `.unwrap()`。锁中毒意味着持有锁的线程 panic 了，锁内数据完整性无法保证——崩溃重启比试图恢复更安全。详见 [rust-guidelines](../rust-guidelines/) §3.3。

### 4.3 状态注册方式

```rust
// 在 Builder 的 setup 或链式调用中注册
tauri::Builder::default()
    .manage(AppConfig { app_name: "MyApp".into(), version: "1.0".into() })
    .manage(Mutex::new(AppState { counter: Mutex::new(0) }))
```

### 4.4 在 `setup` 中初始化复杂状态

```rust
.setup(|app| {
    let app_handle = app.handle();
    // 创建复杂状态 —— Tauri 的 manage() 已自动包裹 Arc，无需手动再包
    let app_state = MyAppState::new();
    app.manage(app_state);

    // 动态创建窗口（而非在 tauri.conf.json 中静态定义）
    let start_minimized = false; // 可从持久化配置中加载
    let main_window = create_main_window(app_handle.clone());
    if !start_minimized {
        main_window.show().expect("Failed to show main window");
    }

    // 注册窗口事件
    main_window.on_window_event(move |event| match event {
        tauri::WindowEvent::CloseRequested { .. } => {
            app_handle.exit(0);
        }
        _ => {}
    });

    Ok(())
})
```

### 4.5 应用设置持久化

**使用 TOML/JSON 文件持久化设置，配合 `Lazy<RwLock<T>>` 实现高性能全局访问。**

对于需要读写频繁的应用配置（窗口状态、用户偏好、连接历史等），推荐以下模式：

```rust
use std::sync::{LazyLock, RwLock};
use serde::{Deserialize, Serialize};

/// 应用设置结构体
#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct AppSettings {
    #[serde(default = "default_theme")]
    pub theme: String,

    #[serde(default)]
    pub auto_start: bool,

    #[serde(default = "default_polling_frequency")]
    pub polling_frequency: u32,
}

fn default_theme() -> String { "light".to_string() }
fn default_polling_frequency() -> u32 { 125 }

/// 全局缓存 —— 读多写少用 RwLock；极端热路径可考虑 ArcSwap
static GLOBAL_SETTINGS: LazyLock<RwLock<AppSettings>> =
    LazyLock::new(|| RwLock::new(load_settings_internal()));

/// 从文件加载设置
fn load_settings_internal() -> AppSettings {
    let path = get_settings_path();
    if !path.exists() {
        let default = AppSettings::default();
        save_settings_to_disk(&default);
        return default;
    }
    // 反序列化，失败时 fallback 到默认值
    std::fs::read_to_string(&path)
        .ok()
        .and_then(|s| toml::from_str(&s).ok())
        .unwrap_or_else(|| {
            log::warn!("Failed to parse settings, using defaults");
            AppSettings::default()
        })
}

/// 以 command 暴露给前端
#[tauri::command]
pub async fn update_settings(app: tauri::AppHandle, new_settings: AppSettings) -> Result<(), String> {
    // 1. 数据验证
    if new_settings.polling_frequency == 0 {
        return Err("Invalid frequency".into());
    }

    // 2. 更新全局缓存 + 差异驱动的副作用
    let old_auto_start = GLOBAL_SETTINGS.read()
        .expect("Settings RwLock should not be poisoned").auto_start;
    *GLOBAL_SETTINGS.write()
        .expect("Settings RwLock should not be poisoned") = new_settings.clone();

    // 3. 异步持久化（不阻塞调用方）
    tokio::spawn(async move {
        save_settings_to_disk(&new_settings);
    });

    // 4. 触发副作用（如 auto_start 变更 → 调用 autostart 插件）
    if new_settings.auto_start != old_auto_start {
        if new_settings.auto_start {
            app.autolaunch().enable().map_err(|e| e.to_string())?;
        } else {
            app.autolaunch().disable().map_err(|e| e.to_string())?;
        }
    }

    Ok(())
}
```

**关键设计原则**：

| 原则 | 说明 |
|------|------|
| `#[serde(default)]` 逐字段 | 新增/删除字段不影响旧配置文件 |
| fallback 而非崩溃 | 文件损坏时使用默认值 + 日志警告，不 panic |
| 差异驱动副作用 | 只对实际变化的字段执行对应操作（如启动 autostart），避免重复注册 |
| `tokio::spawn` 写文件 | 设置更新后不阻塞前端，后台异步写盘。**注意**：应用退出时 spawn 的任务可能被丢弃，最后一次写入可能丢失——对关键数据应在退出前显式 `flush` |
| 全局缓存 + State 混合模式 | 高频读取走 `Lazy<RwLock<T>>`（无命令调用开销），复杂状态仍用 `State<T>` |

## 5. 权限模型（Capabilities & Permissions）

### 5.1 最小权限原则

**`capabilities/*.json` 只授予当前窗口/WebView 真正需要的权限。不要使用通配符 `"*"`。**

**正确**：

```json
{
  "identifier": "main-capability",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "fs:read-files",
    "fs:scope-home"
  ]
}
```

**错误**：

```json
{
  "identifier": "dangerous-capability",
  "windows": ["*"],
  "permissions": ["*"]
  // 所有窗口都能访问所有命令 —— 安全噩梦
}
```

### 5.2 权限作用域细化

**为每个命令定义精确的 allow/deny 作用域。**

```json
{
  "identifier": "fs-scoped",
  "windows": ["main"],
  "permissions": [
    {
      "identifier": "fs:allow-read-file",
      "allow": [{ "path": "$DOCUMENT/*" }],
      "deny": [{ "path": "$DOCUMENT/secret.txt" }]
    }
  ]
}
```

### 5.3 平台特定能力分发

**使用 `platforms` 字段区分桌面/移动端权限。**

```json
{
  "identifier": "desktop-features",
  "windows": ["main"],
  "platforms": ["macOS", "windows", "linux"],
  "permissions": [
    "global-shortcut:allow-register",
    "updater:default"
  ]
}
```

### 5.4 多窗口/多 WebView 权限分离

当应用有多个窗口（如主窗口 + 设置窗口）时，不同窗口应拥有不同级别的权限：

```json
{
  "identifier": "admin-panel",
  "windows": ["admin"],
  "permissions": [
    "core:default",
    "shell:allow-execute"
  ]
}
```

```json
{
  "identifier": "main-window",
  "windows": ["main"],
  "permissions": [
    "core:default"
    // 无 shell 权限
  ]
}
```

## 6. 插件集成（Plugin Integration）

**Tauri v2 的插件系统是功能扩展的核心方式。根据插件类型选择合适的注册方式。**

### 6.1 插件注册方式分类

Tauri v2 插件有 **三种注册形态**，取决于插件提供的 API 类型：

| 注册方式 | 适用插件 | 示例 |
|---------|---------|------|
| **Builder 链式** `.plugin(...)` | 有 Builder 构造器的插件 | `tauri-plugin-log`, `tauri-plugin-updater`, `tauri-plugin-window-state` |
| **Setup 内注册** `app.handle().plugin(...)` | 需访问 app handle 的插件 | `tauri-plugin-single-instance`, `tauri-plugin-autostart` |
| **函数式 init** `.plugin(init(...))` | 无 Builder，直接 init 的插件 | `tauri-plugin-os`, `tauri-plugin-autostart` |

### 6.2 插件注册完整示例

以下代码展示了 Tauri v2 中常见插件的完整注册方式（来源：[Tauri 官方文档 — 插件系统](https://v2.tauri.app/develop/plugins/)）：

```rust
#[cfg_attr(mobile, tauri::mobile_entry_point)]
pub fn run() {
    tauri::Builder::default()
        // --- Builder 链式注册（推荐用于有 Builder 的插件）---
        .plugin(tauri_plugin_window_state::Builder::new().build())
        .plugin(
            tauri_plugin_log::Builder::new()
                .target(tauri_plugin_log::Target::new(
                    tauri_plugin_log::TargetKind::Folder {
                        path: get_log_path(),
                        file_name: None,
                    },
                ))
                .level(log::LevelFilter::Debug)
                .timezone_strategy(tauri_plugin_log::TimezoneStrategy::UseLocal)
                .max_file_size(1024 * 128)
                .rotation_strategy(tauri_plugin_log::RotationStrategy::KeepAll)
                .build(),
        )
        .plugin(tauri_plugin_os::init())

        // --- Setup 内注册（插件需要 #[cfg(desktop)] 条件编译）---
        .setup(|app| {
            #[cfg(desktop)]
            app.handle().plugin(
                tauri_plugin_single_instance::init(|_app, _args, _cwd| {})
            );

            #[cfg(desktop)]
            app.handle().plugin(
                tauri_plugin_autostart::init(
                    tauri_plugin_autostart::MacosLauncher::LaunchAgent,
                    Some(vec!["--flag1", "--flag2"]),
                )
            );

            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### 6.3 插件安装方式

**方式一：CLI 命令（推荐，自动处理依赖和权限）**
```bash
# 使用包管理器安装
pnpm tauri add updater
pnpm tauri add autostart
pnpm tauri add log
pnpm tauri add window-state
pnpm tauri add single-instance
```

**方式二：手动安装**
需要在 `Cargo.toml`、`package.json`（若有前端 API）、capabilities 三处同步配置：

```toml
# Cargo.toml
[dependencies]
tauri-plugin-log = { version = "2", features = ["colored"] }
tauri-plugin-updater = "2"
tauri-plugin-autostart = "2"
tauri-plugin-single-instance = "2"
tauri-plugin-window-state = "2"
tauri-plugin-os = "2"
tauri-plugin-opener = "2"
```

```json
// npm 前端包 (package.json)
{
  "dependencies": {
    "@tauri-apps/plugin-log": "~2",
    "@tauri-apps/plugin-updater": "~2",
    "@tauri-apps/plugin-os": "~2",
    "@tauri-apps/plugin-window-state": "~2",
    "@tauri-apps/plugin-opener": "^2"
  }
}
```

```json
// capabilities/default.json — 仅授予当前窗口需要的权限
{
  "permissions": [
    "core:default",
    "log:default",
    "updater:default",
    "window-state:default",
    "opener:default",
    "os:default"
  ]
}
```

### 6.4 插件配置的 JSON5 端

部分插件需要在 `tauri.conf.json5` 的 `plugins` 字段中配置（来源：[Tauri 官方配置参考](https://v2.tauri.app/develop/plugins/)）：

```json5
{
  "plugins": {
    // updater 插件 —— 需要 pubkey + endpoints（双端配置模式）
    "updater": {
      "pubkey": "dW50cnVzdGVkIGNvbW1lbnQ6...",
      "endpoints": [
        "https://github.com/user/repo/releases/latest/download/latest.json"
      ]
    }
    // 其他插件如有配置需求也在此补充
  }
}
```

## 7. IPC 通信模式

### 7.1 Commands vs Events vs Channels 选择

| 通信模式 | 适用场景 | 类型安全 | 性能 |
|---------|---------|:-------:|:----:|
| Command (`invoke`) | 请求-响应，需要返回值 | ✅ 强类型 | 中 |
| Event (`emit`/`listen`) | 推送通知、生命周期事件 | ❌ JSON 字符串 | 高（单向） |
| Channel | 流式数据（下载进度、stdout、WebSocket） | ✅ 强类型 | 最高（有序） |

**选择规则**：
- 一次调用一次返回 → **Command**
- 后端→前端单向推送，低频（状态变化通知） → **Event**
- 后端→前端流式推送，高频（进度、实时数据） → **Channel**

**正确（Channel 流式传输）**：

```rust
use tauri::ipc::Channel;
use serde::Serialize;

#[derive(Clone, Serialize)]
#[serde(tag = "event", content = "data")]
enum DownloadProgress {
    Started { total: u64 },
    Progress { downloaded: u64 },
    Finished,
}

#[tauri::command]
async fn download_file(url: String, on_event: Channel<DownloadProgress>) -> Result<(), String> {
    on_event.send(DownloadProgress::Started { total: 1000 })
        .map_err(|e| format!("Failed to send progress: {e}"))?;
    on_event.send(DownloadProgress::Progress { downloaded: 500 })
        .map_err(|e| format!("Failed to send progress: {e}"))?;
    on_event.send(DownloadProgress::Finished)
        .map_err(|e| format!("Failed to send progress: {e}"))?;
    Ok(())
}
```

```typescript
// 前端
import { invoke, Channel } from "@tauri-apps/api/core";

type DownloadEvent =
  | { event: "started"; data: { total: number } }
  | { event: "progress"; data: { downloaded: number } }
  | { event: "finished" };

const channel = new Channel<DownloadEvent>();
channel.onmessage = (msg) => {
  if (msg.event === "progress") console.log(msg.data.downloaded);
};

await invoke("download_file", { url: "https://example.com/file", onEvent: channel });
```

### 7.2 事件监听生命周期管理

**使用 `listen()` 返回的 `unlisten` 函数在组件卸载/不再需要时取消监听。**

```typescript
import { listen } from "@tauri-apps/api/event";

// 开始监听
const unlisten = await listen("sync-started", (event) => {
  console.log("Sync started:", event.payload);
});

// 不再需要时取消
unlisten();
```

### 7.3 `use tauri::Emitter` 导入

**Tauri v2 中必须显式 `use tauri::Emitter` 才能调用 `app.emit()` —— 这是 v1 升级到 v2 最常见的遗漏错误。**

```rust
use tauri::Emitter; // 容易忘记 —— v2 中必须显式导入

#[tauri::command]
fn notify_frontend(app: tauri::AppHandle) -> Result<(), String> {
    app.emit("event-name", "payload").map_err(|e| e.to_string())
}
```

## 8. 前端调用规范

### 8.1 API 导入路径（v2 vs v1）

**Tauri v2 的 API 导入路径已变更。**

```typescript
// ✅ v2 正确路径
import { invoke, Channel } from "@tauri-apps/api/core";
import { listen, emit } from "@tauri-apps/api/event";
import { getCurrentWindow } from "@tauri-apps/api/window";
import { locale } from "@tauri-apps/plugin-os";

// ❌ v1 已废弃路径
import { invoke } from "@tauri-apps/api";           // v2 中不存在
import { listen } from "@tauri-apps/api/event";      // ✅ 这个没变
```

### 8.2 命令调用参数命名

**前端传递的参数名使用 camelCase，Tauri 自动映射到 Rust 的 snake_case。**

```rust
#[tauri::command]
async fn create_user(user_name: String, user_age: u32) -> Result<User, String> {
    // Rust 参数为 snake_case
}
```

```typescript
// 前端 —— camelCase 自动映射为 snake_case
await invoke("create_user", { userName: "Alice", userAge: 30 });
```

### 8.3 前端数据模型

前端应定义与后端返回值匹配的类型接口：

```typescript
interface User {
  id: string;
  name: string;
  email: string;
}

// 泛型 invoke 调用
const user = await invoke<User>("get_user", { id: "123" });
```

### 8.4 前端全局状态管理建议

Tauri 应用中，前端通过 `invoke` 与后端交互。对于高频实时数据（如设备状态推送、进度更新），建议使用集中式 store 管理全局状态。

**Vue 3 推荐模式：`reactive`**

```typescript
// stores/app_state.ts
import { reactive } from "vue";

export interface AppState {
  isConnected: boolean;
  deviceName: string;
  autoStart: boolean;
  theme: string;
  showModal: boolean;
  statusMessage: string;
}

export const appState: AppState = reactive({
  isConnected: false,
  deviceName: "",
  autoStart: false,
  theme: "light",
  showModal: false,
  statusMessage: "",
});
```

**React 推荐模式：zustand**

```typescript
// stores/app-store.ts
import { create } from "zustand";

interface AppState {
  isConnected: boolean;
  deviceName: string;
  autoStart: boolean;
  theme: string;
  statusMessage: string;
  setConnected: (name: string) => void;
  setDisconnected: () => void;
}

export const useAppStore = create<AppState>((set) => ({
  isConnected: false,
  deviceName: "",
  autoStart: false,
  theme: "light",
  statusMessage: "Ready",
  setConnected: (name) => set({ isConnected: true, deviceName: name }),
  setDisconnected: () => set({ isConnected: false, deviceName: "" }),
}));
```

**原则**：设备状态、应用设置、UI 状态放在集中 store 中；页面级临时状态用组件局部 state；不在 store 中存放后端数据的"镜像副本"（通过 invoke + 缓存更新）。

### 8.5 Tauri 前端构建配置（Vite + tsconfig）

**使用 Vite 作为 Tauri 前端的构建工具时，有若干 Tauri 特有的配置点**（来源：[Tauri 官方文档 — Frontend Configuration](https://v2.tauri.app/develop/frontend/)）：

```typescript
// vite.config.ts
import { defineConfig } from "vite";
import vue from "@vitejs/plugin-vue";  // 或 react、svelte 等

const host = process.env.TAURI_DEV_HOST;

export default defineConfig({
  plugins: [vue()],

  resolve: {
    alias: {
      "@": "/src",
    },
  },

  // Tauri 开发专用配置
  clearScreen: false,          // 防止 Vite 清屏遮盖 Rust 编译错误
  server: {
    port: 1420,                 // 必须与 tauri.conf.json5 中的 devUrl 一致
    strictPort: true,           // 端口占用时报错而非自动换端口
    host: host || false,
    watch: {
      ignored: ["**/src-tauri/**"],  // 避免 Rust 代码变更触发前端重编译
    },
  },
});
```

**tsconfig.json（Vite 项目）推荐配置**：

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",  // ⚠️ 非 nodenext！Vite 使用 bundler 模式
    "strict": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "noEmit": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*.ts", "src/**/*.vue"]
}
```

**关键配置项说明**：

| 配置项 | 值 | 理由 |
|--------|------|------|
| `clearScreen` | `false` | 让 Rust 编译错误在终端可见 |
| `server.port` | `1420` | Tauri 默认开发端口，与 `devUrl` 对应 |
| `server.strictPort` | `true` | 端口冲突时报错，避免 Tauri 连接错误端口 |
| `watch.ignored` | `["**/src-tauri/**"]` | Rust 变更不需要 Vite 重编译 |
| `moduleResolution` | `"bundler"` | Vite 打包器自行解析模块，不需要 `.js` 扩展名 |

## 9. Cargo.toml 依赖管理

### 9.1 核心依赖版本

| crate | 版本 | 说明 |
|-------|------|------|
| `tauri` | `"2"` | 启用 features: `tray-icon`, `config-json5` |
| `tauri-build` | `"2"` | 启用 features: `config-json5`（如使用 JSON5 配置） |
| `serde` | `"1"` | 启用 features: `derive` |
| `thiserror` | `"2"` | 自定义错误类型 |
| `tokio` | `"1"` | 启用 features: `full` |

### 9.2 插件安装方式

**CLI 方式**（推荐）：
```bash
pnpm tauri add fs
pnpm tauri add shell
pnpm tauri add updater
```

**手动方式**（需要在 Cargo.toml 和 capabilities 中同步配置）：
```toml
# Cargo.toml 添加依赖
tauri-plugin-fs = "2"
tauri-plugin-shell = "2"
```
```json
// capabilities/default.json 添加权限
"fs:default",
"shell:allow-open"
```

## 10. 动态窗口管理

**推荐在 Rust 代码中动态创建窗口，而非在 `tauri.conf.json` 中静态定义。**

```rust
fn create_main_window(app_handle: AppHandle) -> WebviewWindow {
    // 平台特定配置
    #[cfg(target_os = "windows")]
    let decorations = false;  // Windows 下使用自绘标题栏

    WebviewWindowBuilder::new(
        &app_handle,
        "main",
        WebviewUrl::App("index.html".into()),
    )
    .title("My App")
    .inner_size(1080.0, 720.0)
    .min_inner_size(800.0, 600.0)
    .resizable(true)
    .decorations(decorations)
    .center()
    .visible(false)    // 先隐藏，setup 完成后按需显示
    .build()
    .expect("Failed to create window")
}
```

### 10.1 系统托盘（System Tray）

**在 `setup` 钩子中初始化托盘，支持双击显示窗口和右键菜单。**

（来源：[Tauri 官方文档 — System Tray](https://v2.tauri.app/learn/system-tray/)）

```rust
use tauri::{
    menu::{Menu, MenuItem},
    tray::{MouseButton, TrayIconBuilder, TrayIconEvent},
    AppHandle, Manager,
};

fn init_tray(app: &AppHandle) -> tauri::Result<()> {
    let quit = MenuItem::with_id(app, "quit", "退出", true, None::<&str>)?;
    let menu = Menu::with_items(app, &[&quit])?;

    let _tray = TrayIconBuilder::new()
        .icon(app.default_window_icon()
            .expect("Default window icon not configured").clone())
        .menu(&menu)
        .show_menu_on_left_click(false)    // Linux 不支持；Windows/macOS 默认 true
        .on_menu_event(|app, event| match event.id.as_ref() {
            "quit" => {
                app.exit(0);
            }
            _ => {}
        })
        .on_tray_icon_event(|tray, event| {
            if let TrayIconEvent::DoubleClick {
                button: MouseButton::Left,
                ..
            } = event
            {
                let app = tray.app_handle();
                if let Some(window) = app.get_webview_window("main") {
                    let _ = window.show();
                    let _ = window.set_focus();
                }
            }
        })
        .build(app)?;

    Ok(())
}
```

**关键设计决策**：

| 配置项 | 推荐值 | 理由 |
|--------|:------:|------|
| `show_menu_on_left_click` | `false` | 左键双击显示窗口，右键显示菜单，体验更明确 |
| 托盘事件 vs 菜单事件 | 分离处理 | `on_tray_icon_event` 处理双击/单击，`on_menu_event` 处理菜单项 |
| 退出逻辑 | 菜单项 `quit` → `app.exit(0)` | 确保关闭所有资源后再退出 |

**前端侧（也可在 JS 中创建托盘）**：

```typescript
import { TrayIcon } from "@tauri-apps/api/tray";
import { Menu } from "@tauri-apps/api/menu";

const menu = await Menu.new({
  items: [{ id: "quit", text: "退出" }],
});

const tray = await TrayIcon.new({
  tooltip: "My App",
  menu,
  menuOnLeftClick: false,
});
```

## 11. 快速参考

| 规则 | 核心要求 | 主要例外 |
|------|---------|---------|
| 插件注册 | 按类型选择 Builder 链式 / Setup 内 / init 注册 | 原型阶段可简化 |
| 设置持久化 | TOML 文件 + `Lazy<RwLock<T>>` 全局缓存 | 简单配置可仅用文件 |
| 模块化命令 | 命令拆分到 `commands/` 目录 | 1-2 个简单命令可内联 |
| 自定义错误 | `thiserror` enum 替代 `String` | 原型阶段可临时用 `String` |
| 异步命令 | `async fn` 避免 UI 阻塞 | 纯内存计算用同步 |
| 状态注入 | `manage()` + `State<T>` | 高频只读配置可用全局 static |
| 最小权限 | capabilities 中不要用 `"*"` | 无 |
| Channel 流数据 | 高频推送用 Channel | 低频事件用 Event |
| 系统托盘 | `TrayIconBuilder` + `on_tray_icon_event` | 纯后台服务可省略 |
| 前端 API 路径 | `@tauri-apps/api/core` | v1 路径 `@tauri-apps/api` 已废弃 |
| `Emitter` 导入 | 必须 `use tauri::Emitter` | 无 |
| 动态窗口 | Rust 代码中创建 | 静态配置仅适合极简应用 |
| Vite 配置 | `clearScreen: false` + `strictPort` + `watch.ignored` | 非 Vite 项目不用 |
| JSON5 配置 | 文件名 `*.json5` + feature `config-json5` | 使用标准 JSON 可省略 |

## 12. 禁止事项

- 禁止使用 Tauri v1 的 `allowlist` 格式配置权限（v2 使用 capabilities）
- 禁止使用旧版 API 导入路径（`@tauri-apps/api` 顶层导入在 v2 中已废弃）
- 禁止在全局 `static` 中存储应用状态——应使用 `tauri::State`
- 禁止在异步命令中用 `std::sync::Mutex` 在 `.await` 点之间持有锁
- 禁止在 capabilities 中使用 `"*"` 通配符开放所有权限
- 禁止多次调用 `invoke_handler`——只调用一次，传入所有命令
- 禁止在前端组件卸载时不调用 `unlisten()`（造成内存泄漏）
- 禁止将窗口定义硬编码在 `tauri.conf.json` 中（失去动态控制能力）

## 13. 自检要求

输出 Tauri v2 项目代码或配置前，逐项检查：

- [ ] 插件按类型正确注册（Builder 链式 / Setup 内 / init），而非混用？
- [ ] 设置有完善的持久化逻辑（默认值生成、损坏 fallback、异步写盘）？
- [ ] 命令拆分到独立模块（`commands/` 目录），而非堆在 `main.rs` 中？
- [ ] 自定义错误类型（`thiserror` enum）替代 `String` 错误？
- [ ] 异步命令使用 `async fn`，非阻塞 UI 线程？
- [ ] `capabilities/*.json` 遵循最小权限原则，没有 `"*"` 通配？
- [ ] `std::sync::Mutex` 未在 `.await` 点之间持有？
- [ ] 前端 API 导入路径使用 `@tauri-apps/api/core` / `@tauri-apps/api/event`（v2 路径）？
- [ ] IPC 流式数据使用 `Channel` 而非 Event 传递频繁更新？
- [ ] `generate_handler!` 只调用一次且包含所有命令？
- [ ] 系统托盘正确初始化（如有需求），`show_menu_on_left_click` 按平台设置？
- [ ] 跨平台权限使用 `platforms` 字段区分？
- [ ] `tauri::Emitter` trait 已显式 `use`（v2 中 `emit()` 需要显式导入）？
- [ ] 窗口在 Rust 代码中动态创建，而非静态配置？
- [ ] Vite 配置中 `clearScreen: false`、`strictPort`、`watch.ignored` 已设置？
- [ ] tsconfig 中 `moduleResolution` 使用 `"bundler"`（Vite 项目）而非 `"nodenext"`？
- [ ] 日志配置使用 `TargetKind::Folder` 输出到文件？
- [ ] release profile 设置了大小优化（`opt-level = "s"`、`strip`、`lto`）？
