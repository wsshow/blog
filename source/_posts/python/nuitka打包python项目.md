---
title: nuikta打包python项目
date: 2024-06-22 10:18:30
author: ws
description: 使用nuikta打包工具打包python项目
categories: python打包
tags: [python, 打包]
cover:
---
## 创建虚拟环境
`python -m venv nuitka-venv`
## 激活虚拟环境
`nuitka-venv\Script\Activate.ps1`
## 切换镜像源
`pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple`
## 下载nuitka
`pip install nuitka`
## 下载项目依赖
`pip install -r requirements.txt`
## nuitka命令行打包
`nuitka --show-progress --standalone --plugin-enable=numpy --disable-console --onefile --remove-output D:\PyProjects\pyocr\main.py --output-dir=D:\PyProjects\pyocr\out`
