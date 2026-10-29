---
layout: post
title: "OpenIDEA v0.1.23 调试 Python 项目：debugpy + .openidea/launch.json 两步配置"
subtitle: "开源 Rust IDE 的 Python 调试实测记录：装对调试器、写对配置，断点才能生效"
date: 2026-10-29 14:35:00 +0800
author: 清源
header-img: img/post-bg-debug.png
catalog: true
tags: [OpenIDEA, Python, debugpy, 调试, IDE]
---

## 背景

[OpenIDEA](https://github.com/l0ng-ai/openidea) 是一个开源的、IntelliJ 风格的 IDE，使用 Rust + Tauri 构建，支持 Windows / macOS / Linux（x86-64 与 Apple Silicon），提供代码编辑、调试、Git、终端等能力，主打轻量与低资源占用。

我最近用它的 v0.1.23 版本调试 Python 项目时发现：直接点调试并不能工作，必须手动做两步配置（安装 `debugpy` + 提供 `launch.json`）后，断点调试才可用。以下为 v0.1.23 上的实测记录。

## 问题现象

在 OpenIDEA 中打开一个带 `.venv` 的 Python 项目，直接启动调试不可用。排查后发现有两个前提没有满足：

1. 项目 venv 中没有安装 Python 调试适配器 `debugpy`；
2. 项目根目录缺少 OpenIDEA 读取的调试配置文件 `.openidea/launch.json`。

## 第一步：在项目 venv 中安装 debugpy

OpenIDEA 的 Python 调试走的是 `debugpy`（与 VS Code 的 Python 调试适配器同源），所以必须先把它装进**项目自己的虚拟环境**里。

Windows（PowerShell）：

```
.venv\Scripts\python.exe -m pip install debugpy
```

Linux / macOS：

```
.venv/bin/python -m pip install debugpy
```

验证是否安装成功：

```
.venv\Scripts\python.exe -m pip show debugpy
```

## 第二步：创建 .openidea/launch.json

在**项目根目录**（与 `.venv` 同级）下创建 `.openidea` 目录，并在其中新建 `launch.json`，内容如下：

```json
{
    "name": "Python: Current File",
    "type": "debugpy",
    "request": "launch",
    "program": "${file}",
    "python": "${workspaceFolder}/.venv/bin/python",            // Linux/macOS
    // "python": "${workspaceFolder}/.venv/Scripts/python.exe", // Windows
    "console": "integratedTerminal"
}
```

各字段说明：

| 字段 | 值 | 说明 |
| --- | --- | --- |
| name | Python: Current File | 调试配置的显示名称，可在调试菜单里识别 |
| type | debugpy | 调试适配器类型，对应第一步安装的 `debugpy` |
| request | launch | 采用启动式调试（launch 模式） |
| program | ${file} | 调试当前打开的 `.py` 文件 |
| python | ${workspaceFolder}/.venv/... | 指定解释器，**必须指向装了 debugpy 的那个 venv** |
| console | integratedTerminal | 在集成终端中运行，便于 `input()` 等交互输入 |

## 关键点与常见坑

1. **目录名带点**：是 `.openidea`，不是 `openidea`，不要漏掉开头的点。
2. **launch.json 实为 JSONC**：虽然扩展名是 `.json`，但它支持 `//` 注释（与 VS Code 的 launch.json 一致）；如果某个工具要求严格 JSON，把注释行去掉即可。
3. **python 路径必须与安装 debugpy 的环境一致**：`debugpy` 装在哪个 venv，`python` 就指向哪个 venv，否则启动调试会因找不到调试适配器而失败。
4. **venv 目录名按实际调整**：示例默认目录名是 `.venv`；如果你的项目用的是 `venv` 或其他名字，同步修改 `python` 路径。
5. **跨平台路径差异**：Linux/macOS 用 `bin/python`，Windows 用 `Scripts/python.exe`。两份配置同时保留、按平台注释切换即可。
6. **program 可换**：`${file}` 调试当前文件；若想固定调试入口脚本，可改为 `${workspaceFolder}/main.py` 这类具体路径。

## 验证

配置完成后：

1. 打开要调试的 `.py` 文件；
2. 在代码行上打断点；
3. 选择 `Python: Current File` 启动调试；
4. 断点命中、变量面板可正常查看，即配置成功。

## 参考

- [l0ng-ai/openidea 项目仓库](https://github.com/l0ng-ai/openidea)
- [OpenIDEA v0.1.23 Release](https://github.com/l0ng-ai/openidea/releases)
- [OpenIDEA: An open-source IntelliJ-style IDE built with Rust](https://dev.to/l0ngai/openidea-an-open-source-intellij-style-ide-built-with-rust-36lp)
- [debugpy（PyPI）](https://pypi.org/project/debugpy/)
