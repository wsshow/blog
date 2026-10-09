---
title: Tauri 后台进程管理：随应用启动 sidecar 并在退出时清理
date: 2023-01-01 22:02:30
updated: 2026-10-09
author: ws
description: 讲清 Tauri sidecar 机制与 Rust 侧进程生命周期管理
categories: ["Tauri"]
tags: ["Tauri", "Rust"]
cover:
---

## 引言

用 Tauri 做桌面应用时，常见做法是前端用 WebView，真正的"内核"（Python/Go 写的服务、算法程序）作为子进程在后台跑。问题往往不在启动，而在退出：应用关了，内核还挂在那里占着端口和文件锁。这篇文章从一段能跑但隐患不小的示例代码说起，讲清 Tauri 的进程模型，给出经过验证的清理方案，并整理 Tauri v2 的 sidecar 正式做法。文中的 Tauri API 以 tauri 2.12.1、tauri-plugin-shell 2.4.x 的官方文档为准；进程管理部分的代码在本机用 rustc 1.84（edition 2021）单独编译运行验证过。

## Tauri 架构与 sidecar 场景

Tauri 应用由两部分组成：

```text
┌─────────────────────────────── Tauri App ───────────────────────────────┐
│                                                                          │
│  Rust 宿主进程（main.rs 所在）                                            │
│   ├── 窗口/菜单/托盘事件循环（tauri crate）                               │
│   ├── IPC 命令（#[tauri::command]）                                       │
│   └── 子进程管理 ── spawn/kill/wait ──►  sdib-core（Python/Go 内核）      │
│                                                                          │
│  WebView（前端 HTML/JS）◄── IPC ──► Rust 宿主                             │
└──────────────────────────────────────────────────────────────────────────┘
```

把内核做成子进程（Tauri 官方叫法是 **sidecar**，"边车"）而不是动态库，好处是：内核可以用任意语言写、崩溃时不拖垮宿主、开发和调试时可以单独运行。代价是**进程生命周期要自己管**：谁启动它、谁在退出时收尸、它自己挂掉之后谁负责重启。

## 常见实现逐段解读

先看一段很容易写出来的实现：

```rust
#![cfg_attr(
    all(not(debug_assertions), target_os = "windows"),
    windows_subsystem = "windows"
)]

use std::{
    cell::RefCell,
    os::windows::process::CommandExt,
    process::{Child, Command},
};

use tauri::Manager;

fn run_core() -> Child {
    return Command::new("core/sdib-core")
        .creation_flags(0x08000000)
        .arg("-loglevel")
        .arg("debug")
        .spawn()
        .expect("failed to execute core process");
}

fn main() {
    let child = RefCell::new(run_core());

    tauri::Builder::default()
        .setup(|app| {
            let window = app.get_window("main").unwrap();

            window.on_window_event(move |event| {
                if let tauri::WindowEvent::Destroyed = event {
                    child
                        .borrow_mut()
                        .kill()
                        .expect("core process wasn't running");
                    child.borrow_mut().wait().unwrap();
                }
            });

            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

它要解决的问题很明确：主窗口关掉时，把内核进程杀掉。下面按从外到内逐段拆。

### `windows_subsystem = "windows"`

```rust
#![cfg_attr(all(not(debug_assertions), target_os = "windows"), windows_subsystem = "windows")]
```

这是一个 crate 级属性：在 **release 构建**、**Windows 平台**下，让程序以 GUI 子系统启动，不弹那个黑色控制台窗口。debug 构建保留控制台，方便看 `println!` 和 panic 信息。注意这段写在第一行、`#!` 开头，位置不能挪。

### `creation_flags(0x08000000)`

```rust
use std::os::windows::process::CommandExt;
...
.creation_flags(0x08000000)
```

`creation_flags` 是 `std::os::windows::process::CommandExt` 提供的 Windows 专用方法，把标志透传给 `CreateProcess`。`0x08000000` 就是 Win32 的 `CREATE_NO_WINDOW`：子进程不创建控制台窗口。

这里有两个细节：

- 它是**纯 Windows API**。`std::os::windows::process::CommandExt` 在 macOS/Linux 上不存在，所以这段 `use` 必须套 `#[cfg(windows)]`，否则项目只能编译 Windows 版——这是一个很常见的疏漏。
- Tauri 的 shell 插件内部也是这么干的：`tauri-plugin-shell` 源码里对 Windows 子进程固定加了 `creation_flags(CREATE_NO_WINDOW)`，用插件就不用手写这个常量。

### 为什么需要 `RefCell`

`window.on_window_event` 接收的是一个 `Fn` 闭包（可能被多次调用，但只能共享捕获环境）。而 `Child::kill(&mut self)` 需要可变借用，闭包体里不能直接拿到 `&mut child`。`RefCell` 提供"运行时借用检查 + 内部可变性"，让不可变闭包也能改里面的值：

```text
on_window_event(move |event| { ... })   闭包捕获 child
        │  Fn：只能 &self
        ▼
RefCell<Child>::borrow_mut()            运行时取得 &mut
        │
        ▼
Child::kill(&mut self)                  修改成功
```

这里之所以不用 `Rc<RefCell<...>>`，是因为闭包只有一个所有者，不需要共享所有权；如果后续要在多个事件处理器之间共用同一个子进程句柄，才需要 `Rc`（或者干脆换成 Tauri 的托管状态，见下文）。

### `kill()` 与 `wait()`

```rust
child.borrow_mut().kill().expect("core process wasn't running");
child.borrow_mut().wait().unwrap();
```

- `kill()`：Unix 上等价于发 `SIGKILL`，Windows 上调用 `TerminateProcess`——**强杀**，不给子进程任何收尾机会。Rust 1.84 的文档明确：如果子进程已经退出，`kill()` 会返回 `Ok(())`（不会报错）。
- `wait()`：等待子进程真正结束并回收内核数据结构。这一步不能省：Unix 上已退出但没被 `wait` 的子进程会变成僵尸进程（zombie），长期留着会耗尽进程号等资源——这正是标准库文档专门警告的点。

## 这段实现的五个问题

把上面这段代码放在生产环境，会遇到这些情况：

1. **清理时机绑死在主窗口上。** `on_window_event` 只监听 `"main"` 窗口的 `Destroyed` 事件。多窗口应用里关掉别的窗口、从托盘菜单退出、`app.exit()`、macOS 上"关闭窗口不等于退出应用"这些路径都绕过了这段清理，内核成了孤儿进程。
2. **强杀不给内核收尾机会。** 内核可能在写数据库、导出文件、释放端口。`kill()` 一发起，这些动作全部中断，留下半写状态。
3. **`wait()` 跑在 UI 线程上。** 窗口事件回调在主线程执行，虽然强杀后一般很快就退出，但在慢机器/杀软扫描/子进程卡死的情况下，事件循环会被阻塞，界面卡住。
4. **子进程自己退出时无人知晓。** 如果内核崩溃或自行退出，宿主不关心，也没有重启/告警逻辑；清理时 `kill()` 虽然不报错，但"子在不在"这个事实需要靠 `try_wait()` 或事件流去发现。
5. **相对路径 + Windows 专用导入。** `Command::new("core/sdib-core")` 依赖当前工作目录——从 Finder、桌面快捷方式或计划任务启动时，工作目录不一定是程序目录，找不到文件；而 `os::windows::process::CommandExt` 没做 cfg，macOS/Linux 编译不过。

另外，`get_window("main")` 是 Tauri v1 的常用写法；v2 里窗口是 WebView 窗口，应该用 `get_webview_window("main")`（`Manager` trait 提供）。

## 清理时机：在 `RunEvent::Exit` 统一收口

Tauri 的事件循环会给宿主一个全局事件回调，v2 的用法是：`.build(ctx)` 拿到 `App`，再用 `App::run(|app_handle, event| ...)` 注册回调。`RunEvent` 的变体包括 `Exit`、`ExitRequested`、`Ready`、`WindowEvent` 等（枚举是 `#[non_exhaustive]`，匹配时必须留一个 `_` 兜底）。其中：

- `ExitRequested`：用户请求退出时触发（点关闭按钮、Cmd+Q 等），可以用 `api.prevent_exit()` 拦截；
- `Exit`：事件循环即将退出，是**最后一个统一清理点**，不管退出路径是关窗、托盘还是 `app.exit()`，都会走到这里。

所以更稳妥的做法是把清理逻辑放在 `RunEvent::Exit`，它是所有退出路径的统一收口点，一次性覆盖前面列出的问题 1。

## 句柄保存：用托管状态管理子进程

用闭包捕获 `RefCell` 在简单场景里能工作，但当清理逻辑进入全局事件回调、而且以后可能从命令或后台线程访问子进程时，更合适的做法是 Tauri 的托管状态（managed state）：

```rust
use std::sync::Mutex;

struct CoreProcess(Mutex<Option<Child>>);

// 注册
app.manage(CoreProcess(Mutex::new(Some(child))));

// 任意拿到 AppHandle 的地方取用
let state = app_handle.state::<CoreProcess>();
let mut guard = state.0.lock().unwrap();
if let Some(child) = guard.as_mut() { /* ... */ }
```

几个来自官方文档（State Management 一节，2026-10 核对）的要点：

- `State` 内部已经用 `Arc` 共享，**不需要**自己写 `Arc<Mutex<T>>`；
- 用 `std::sync::Mutex` 就够，除非要跨 `await` 持有锁，才考虑 async mutex；
- 类型必须完全匹配，注册的是 `CoreProcess` 就取 `CoreProcess`，写错类型时 `Manager::state` 会直接 panic（可以用 `try_state` 做防御）；
- `Option<Child>` 让"当前有没有内核在跑"成为显式状态，清理后置为 `None`，逻辑天然幂等。

## 退出协议：先优雅退出，再兜底强杀

跨平台最省事的"优雅退出协议"是 **stdin 命令**：宿主往子进程的 stdin 写一行 `quit`，内核自己决定怎么收尾（写盘、关端口、 flush 日志），超时没退再强杀。下面这段代码用 rustc 1.84 编译并实际运行过：

```rust
use std::io::Write;
use std::process::{Child, Stdio};
use std::time::{Duration, Instant};

/// 先写一行 quit 让内核自己收尾，限时等待；超时或出错则强杀。
/// 返回 true 表示优雅退出成功。
fn shutdown_child(child: &mut Child, timeout: Duration) -> bool {
    if let Some(stdin) = child.stdin.as_mut() {
        let _ = stdin.write_all(b"quit\n");
        let _ = stdin.flush();
    }

    let deadline = Instant::now() + timeout;
    loop {
        match child.try_wait() {
            Ok(Some(_)) => return true,               // 已退出，且已回收，不留僵尸
            Ok(None) if Instant::now() >= deadline => break,
            Ok(None) => std::thread::sleep(Duration::from_millis(50)),
            Err(_) => break,                          // 状态查询都失败了，直接走强杀
        }
    }

    let _ = child.kill();
    let _ = child.wait();                              // 必须回收
    false
}
```

关键点：`try_wait()` 非阻塞地检查子进程是否退出，退出时在 Unix 上顺带回收进程；循环里用 50ms 的间隔轮询，配合总超时，既不需要引入 `wait_timeout` 依赖，也不会忙等。实测两种情况：

```text
# 内核正常响应 quit
child exited gracefully: ExitStatus(unix_wait_status(0))

# 内核不读 stdin（模拟卡死），800ms 超时后强杀
child killed after timeout
stubborn shutdown took 805.898334ms
```

### 跨平台的差异

- **Unix（macOS/Linux）**：除了 stdin 协议，还可以用信号：`libc::kill(pid, SIGTERM)`（需要 `libc` 依赖），内核注册 `SIGTERM` 处理器做清理。`SIGKILL` 则完全不可捕获。
- **Windows**：没有 `SIGTERM`。可选路径是 stdin 协议、`GenerateConsoleCtrlEvent(CTRL_BREAK)`（需要控制台进程组，Tauri GUI 场景不适用）或 `WM_CLOSE` 消息。**stdin 命令协议是双平台通用的，推荐统一用它**，内核里实现成"读到 quit 就退出主循环"即可。

## 完整的 main.rs（Tauri 2.12.x）

把上面的方案拼起来，就是一份可以直接使用的完整实现：

```rust
// 适用版本：tauri 2.12.x；进程管理部分已在 rustc 1.84 (edition 2021) 下编译运行验证
#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]

use std::{
    io::Write,
    path::PathBuf,
    process::{Child, Command, Stdio},
    sync::Mutex,
    time::{Duration, Instant},
};
use tauri::{Manager, RunEvent};

/// 托管状态：Mutex 提供内部可变性，Option 表示"当前没有在跑的内核"
struct CoreProcess(Mutex<Option<Child>>);

/// 优雅退出的等待上限，按内核实际收尾时间调整，不要设太大
const GRACE_TIMEOUT: Duration = Duration::from_secs(3);

/// 与主程序同目录定位内核（Tauri 打包后 sidecar 就放在这里）
fn core_binary_path() -> PathBuf {
    let mut path = std::env::current_exe()
        .expect("无法定位当前可执行文件")
        .parent()
        .expect("可执行文件没有父目录")
        .to_path_buf();
    path.push(if cfg!(windows) { "sdib-core.exe" } else { "sdib-core" });
    path
}

fn spawn_core() -> std::io::Result<Child> {
    let mut command = Command::new(core_binary_path());
    command
        .arg("-loglevel")
        .arg("debug")
        .stdin(Stdio::piped())    // 优雅退出协议走 stdin
        .stdout(Stdio::null())
        .stderr(Stdio::inherit());

    #[cfg(windows)]
    {
        use std::os::windows::process::CommandExt;
        const CREATE_NO_WINDOW: u32 = 0x0800_0000;
        command.creation_flags(CREATE_NO_WINDOW);
    }

    command.spawn()
}

fn shutdown_child(child: &mut Child, timeout: Duration) -> bool {
    if let Some(stdin) = child.stdin.as_mut() {
        let _ = stdin.write_all(b"quit\n");
        let _ = stdin.flush();
    }

    let deadline = Instant::now() + timeout;
    loop {
        match child.try_wait() {
            Ok(Some(_)) => return true,
            Ok(None) if Instant::now() >= deadline => break,
            Ok(None) => std::thread::sleep(Duration::from_millis(50)),
            Err(_) => break,
        }
    }

    let _ = child.kill();
    let _ = child.wait();
    false
}

fn main() {
    let app = tauri::Builder::default()
        .setup(|app| {
            let child = spawn_core().expect("内核进程启动失败");
            app.manage(CoreProcess(Mutex::new(Some(child))));
            Ok(())
        })
        .build(tauri::generate_context!())
        .expect("Tauri 应用初始化失败");

    app.run(|app_handle, event| {
        // 只在最后一个清理点动手；该回调覆盖关窗、托盘退出、app.exit()
        if let RunEvent::Exit = event {
            let state = app_handle.state::<CoreProcess>();
            if let Some(mut child) = state.0.lock().unwrap().take() {
                let graceful = shutdown_child(&mut child, GRACE_TIMEOUT);
                println!("core exited gracefully: {graceful}");
            }
        }
    });
}
```

使用说明：

- `App::run` 的签名是 `FnMut(&AppHandle<R>, RunEvent)`，回调不可返回、不可恢复（事件循环结束后进程由框架直接退出），所以清理必须在这里同步做完；`GRACE_TIMEOUT` 控制在几秒内，别在这里做重活。
- `RunEvent` 是 `#[non_exhaustive]`，这里用 `if let` 只匹配 `Exit`，其他事件自动忽略，等价于留了兜底分支。
- 如果同时要在 `ExitRequested` 里拦"用户点关闭"做二次确认，注意不要在 `ExitRequested` 和 `Exit` 里重复清理——`Option::take()` 保证了幂等，重复调用是安全的。

## Tauri v2 的正式方案：`externalBin` + shell 插件

上面用 `std::process::Command` 直接启动，适合"内核对 Tauri 没有感知"的简单场景。Tauri v2 提供了正规的 sidecar 通道，能把它作为应用资源一起打包。

### 1. 在 `tauri.conf.json` 声明 externalBin

```json
{
  "bundle": {
    "externalBin": ["binaries/sdib-core"]
  }
}
```

Tauri 要求同名文件按**目标平台三元组**命名，例如：

```text
src-tauri/binaries/sdib-core-aarch64-apple-darwin
src-tauri/binaries/sdib-core-x86_64-pc-windows-msvc.exe
src-tauri/binaries/sdib-core-x86_64-unknown-linux-gnu
```

用这个命令查当前平台的三元组（Rust 1.84 起支持 `--print host-tuple`）：

```bash
rustc --print host-tuple
# 输出示例：aarch64-apple-darwin
```

### 2. Rust 侧用 shell 插件启动

```bash
cargo add tauri-plugin-shell
```

```rust
use tauri_plugin_shell::ShellExt;

fn main() {
    tauri::Builder::default()
        .plugin(tauri_plugin_shell::init())
        .setup(|app| {
            let (_rx, child) = app
                .shell()
                .sidecar("sdib-core")          // 只写文件名，不要写 binaries/ 前缀和三元组后缀
                .expect("sidecar 未在 externalBin 中声明")
                .args(["-loglevel", "debug"])
                .spawn()
                .expect("sidecar 启动失败");
            // child 是 tauri_plugin_shell::process::CommandChild
            app.manage(Mutex::new(Some(child)));
            Ok(())
        })
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

`spawn()` 返回 `(rx, child)`：`rx` 是输出事件流（`CommandEvent::Stdout/Stderr/Terminated`），`child` 上有三个方法（tauri-plugin-shell 2.4.1 文档）：

- `write(&mut self, buf: &[u8])`：写 stdin，正好用来做上面约定的 `quit` 协议；
- `kill(self)`：强杀，注意**会消费 `self`**，所以要先把句柄从 `Mutex<Option<_>>` 里 `take()` 出来；
- `pid(&self) -> u32`。

`CommandChild` 没有 `try_wait`，子进程是否退出要看 `rx` 里的 `CommandEvent::Terminated` 事件。清理时的写法可以这样组织：用一个 `AtomicBool`（放在托管状态里）由读 `rx` 的后台任务在收到 `Terminated` 时置位；`RunEvent::Exit` 里先 `write(b"quit\n")`，轮询该标志最多 3 秒，然后对还没退出的句柄 `take()` + `kill()`。

### 3. capabilities：前端调用才需要

如果 sidecar 是由**前端 JS** 通过 `Command.sidecar()` 调起的，需要在 `src-tauri/capabilities/default.json` 里授权：

```json
{
  "permissions": [
    "core:default",
    {
      "identifier": "shell:allow-execute",
      "allow": [{ "name": "binaries/sdib-core", "sidecar": true }]
    }
  ]
}
```

用 `spawn` 而不是 `execute` 就换成 `shell:allow-spawn`。**从 Rust 侧调用不经过 IPC，不受 capabilities 约束**——官方 sidecar 文档里 "Running it from Rust" 一节也没有权限步骤。这条经常被反过来理解，白白卡半天。

### 4. 打包后的路径问题

两个被源码证实的细节：

- `tauri-build` 在构建时会把 `binaries/sdib-core-<target-triple>` 复制到**主程序所在目录**，并去掉 `-<target-triple>` 后缀；dev 和打包产物行为一致。
- shell 插件的 `sidecar()` 定位方式就是 `current_exe().parent().join("sdib-core")`（Windows 自动补 `.exe`）。

所以如果要绕过插件用 `std::process::Command`，`core_binary_path()` 按"主程序同目录"拼路径是等价的；但插件还额外处理了输出事件流、Windows 隐藏窗口、非 UTF-8 编码（`Encoding`）等问题，能用插件就用插件。

### 两条路线怎么选

| 维度 | shell 插件 `sidecar()` | `std::process::Command` |
| --- | --- | --- |
| 打包集成 | `externalBin` 自动收集、随包分发 | 需要自己确保文件进包 |
| 输出处理 | `rx` 事件流，异步不阻塞 | 自己管管道/线程 |
| 退出等待 | 无 `try_wait`，靠 `Terminated` 事件 | `try_wait` 轮询，同步逻辑直观 |
| 权限模型 | 前端调用需 capabilities | 不受 capabilities 影响 |
| 适用场景 | 标准 sidecar，尤其是前端交互多的 | 极简内核、或想完全掌控进程语义 |

## 边界与坑

1. **子进程自己退出**：宿主必须能感知。用 `std::process` 时在事件循环里周期性 `try_wait()`；用 shell 插件时监听 `CommandEvent::Terminated`。感知之后才谈得上重启或弹窗告警。Unix 上别忘了 `wait` 回收僵尸进程。
2. **宿主被强杀**（任务管理器结束进程、崩溃）：任何写在 `RunEvent` 里的清理都不会执行。要兜底得靠操作系统：Windows 用 **Job Object** 并设置 `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE`，父进程一死，作业里的子进程被系统连带终止；Linux 可以用 `prctl(PR_SET_PDEATHSIG, SIGKILL)`。这超出了 Tauri 的范畴，但对"内核绝不能残留"的产品值得做。
3. **不要绑定单个窗口事件**：多窗口、托盘、`app.exit()` 各有各的路径，`RunEvent::Exit` 是唯一能统一收口的地方。托盘程序尤其容易踩——点"退出"菜单时主窗口可能早已销毁。
4. **`ExitRequested` + `prevent_exit()` 要小心**：它允许你在退出前弹确认框，但如果拦下之后忘记再次触发退出（或清理逻辑挂在 `ExitRequested` 里而本次退出被拦截），应用会"退不干净"。建议清理只写在 `Exit`，`ExitRequested` 只做拦截决策。
5. **sidecar 命名冲突**：`tauri-build` 源码明确规定，sidecar 文件名不能与 Cargo 包名相同，否则构建直接报错（`Cannot define a sidecar with the same name as the Cargo package name`）。给内核起个不同于应用包的名字。
6. **版本门槛**：tauri-plugin-shell 2.4.x 的官方文档标注要求 Rust ≥ 1.90；Rust 版本低于 1.90 时先 `rustup update`。
7. **调试建议**：开发期让子进程 stdout/stderr 继承到宿主控制台（本机验证代码里用 `Stdio::inherit()`），release 再按需重定向；Windows release 用 `windows_subsystem = "windows"` 后没有控制台，`println!` 是不可见的，排错阶段可以用文件日志代替。

## 总结

- 只监听窗口销毁、直接强杀子进程的思路，只能覆盖"主窗口关闭"这一条路径，多窗口、托盘退出、崩溃、自退出都会漏，而且 `kill()` 不给内核收尾机会。
- 正确做法是三层：**启动**用 `current_exe()` 同目录定位；**托管**用 `app.manage(Mutex<Option<Child>>)`（Tauri 内部已有 Arc，不要再包一层）；**清理**统一放在 `RunEvent::Exit`，先通过 stdin 协议请求优雅退出，限时等待，超时强杀并 `wait()` 回收。
- Tauri v2 的正式 sidecar 路线是 `bundle.externalBin` + `tauri-plugin-shell`：文件要带目标三元组后缀命名，`.sidecar()` 只传基础文件名，前端调用才需要 capabilities。
- 想覆盖"宿主被强杀"的极端情况，要靠 Windows Job Object / Linux `PR_SET_PDEATHSIG` 这类操作系统级机制，应用层代码无能为力。
