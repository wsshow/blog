---
title: git提交规范
date: 2022-12-07 22:20:30
author: ws
description: git commit规范
categories: git
tags: [git]
cover: false
---

## commit message格式

`<type>(<scope>): <subject>`

## type(必须)

| 关键字   | 描述                                    |
| :------- | :-------------------------------------- |
| feat     | 功能新增                                |
| fix      | BUG修复                                 |
| docs     | 文档更新                                |
| style    | 不影响程序逻辑的代码修改                |
| refactor | 重构代码(既没有新增功能，也没有修复BUG) |
| perf     | 性能、体验优化                          |
| test     | 新增测试用例或更新现有测试              |
| build    | 修改项目构建系统配置(Makefile等)        |
| ci       | 项目自动构建流程相关的提交              |
| chore    | 构建过程或辅助工具的变动                |
| revert   | 回滚到早前提交                          |

## scope(可选)

用于说明commit影响的范围

## subject(必须)

用于说明commit目的的简短描述