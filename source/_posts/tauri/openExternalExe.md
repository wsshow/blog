---
title: 后台运行可执行文件
date: 2023-1-1 22:02:30
author: ws
description: tauri使用rust管理后外可执行程序
categories: tauri
tags: [tauri, rust]
cover:
---

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
