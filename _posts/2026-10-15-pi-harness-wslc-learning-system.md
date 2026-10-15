---
layout: post
title: "不重写一行核心引擎：用 WSLc 把开源编码 Agent 封装成可动手的学习系统"
subtitle: "镜像内 pi CLI + JupyterLab，统一入口/关闭脚本 + 后端就绪检测控制台，一套可复用的 Agent 学习 Harness 思路"
date: 2026-10-15 15:30:00 +0800
author: 清源
header-img: img/post-bg-github-cup.jpg
catalog: true
tags: [WSLc, pi, Agent, JupyterLab, 容器, 学习环境]
---

## 引子：读懂一个 Agent，最好的姿势是"动手"

![pi-harness 系统题图](/img/posts/pi-harness-wslc-learning-system/1-hero.jpg)

最近在研究 [pi](https://zread.ai/earendil-works/pi)——一个开源的终端编码 Agent。它最吸引我的地方不是"能写代码"，而是它的分层架构：`agent-loop` 管理对话循环、`durable` 用任务状态机把"一次生成拆成多个工具子任务再合并结果"、`pi-ai` 把 30 多家 LLM 的协议归一化成一套事件流。

源码读得再细，都是"纸上谈兵"。真正理解一个 Agent 的执行机制，最好的姿势是**把它跑起来、给它任务、观察它内部怎么拆解和合并**。

于是问题变成：怎么给一个开源 Agent 搭一个"可以动手"的学习环境？我的答案是——**不重写核心引擎，用 WSLc 把它封装成一个学习型 Harness 系统**。这篇文章记录完整思路。

## 一、先想清楚：学习一个 Agent，到底要准备什么

拆开需求，一套 Agent 学习环境至少要满足四件事：

1. **可执行**：Agent 本身要能跑，最好带一个能写代码的运行时
2. **可观察**：能打开 Notebook 一类交互界面，实时操作、随时实验
3. **可进入**：一键启动、一键关闭，入口要傻瓜化，不然学习热情会被"配置地狱"消耗光
4. **可复用**：换一个开源工具（比如别的 Agent、别的引擎），这套封装思路能直接搬

围绕这四条，我选了三个组件：

- **pi CLI**（`@earendil-works/pi-coding-agent`）：学习对象本体，提供 `pi -p`、`pi --json` 等模式，能输出结构化事件流观察执行过程
- **JupyterLab**：学习界面，Notebook 里能跑 pi 命令、Python、Node，还可以把实验过程留存成可复用的笔记
- **WSLc**：Windows 下的镜像管理工具（docker 风格的 CLI），负责把上面两者打包成镜像、启停容器

目标很明确：**双击一个脚本，环境就绪；学习完双击另一个脚本，环境安全关闭。中间的全部复杂度都收进镜像。**

## 二、系统设计：Harness 三件套

整个系统就三个文件加一个控制台，逻辑非常薄。

![Harness 四层壳：镜像、入口、控制台、关闭](/img/posts/pi-harness-wslc-learning-system/2-four-shells.jpg)

### 1. 镜像层（Dockerfile）：把环境固化成制品

```
FROM node:22-bookworm
npm install -g @earendil-works/pi-coding-agent@1.2.0   # pi CLI
pip install jupyterlab notebook bash_kernel ipykernel # 学习界面
COPY notebooks/ /notebooks/                           # 预置示例
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--port=8888", "--allow-root", ...]
```

几个关键选择：

- **基础镜像选 Node 22**：pi 要求 Node ≥ 22.19，官方 Node 镜像直接满足，还自带系统包管理可以装 Python
- **Notebook 里装 Bash 内核**：这是点睛之笔——学习 Agent 不只是"看文档"，而是真的在 Notebook 里跑 `pi --json "..."`，观察 `agent_start → turn_start → tool_execution → agent_end` 的事件流
- **预置三个示例 Notebook**：快速上手、核心机制实验、SDK 集成，用户打开就有事可做

### 2. 统一入口（start-pi-harness.ps1）：幂等四步

入口脚本只做四件事，且每一步都幂等（重复执行不会出错）：

```
① 检查 wslc → ② 镜像缺失则 wslc build → ③ 容器未运行则 wslc run → ④ 轮询健康端点，就绪后打开控制台
```

关键点：**启动后不急着打开界面，先轮询 `http://127.0.0.1:8888/api/status` 直到返回 200**，确保用户打开的是"能用"的界面而不是"还在启动"的界面。

### 3. 学习控制台（index.html）：就绪检测 + 锁定/解锁

控制台是用户唯一看到的界面，设计成"状态驱动"：

- 页面每 3 秒轮询后端健康端点，顶部状态胶囊显示「检测中 / 已就绪 / 未就绪」
- **未就绪时，进入 Jupyter 的按钮和"添加页"按钮全部锁定**，并给出明确引导——避免用户点进一个白屏
- **就绪后**，点击按钮即以 iframe 打开 JupyterLab，可以直接操作执行计算
- 学习区支持**动态添加更多 Notebook 标签页**（预置路径或自定义路径），多开几个实验并行操作

### 4. 统一关闭（stop-pi-harness.ps1）

`wslc stop` 安全停止容器；默认保留容器（Notebook 数据还在），加 `-Remove` 才删除。学习中断的成本降到最低。

这套系统的整体架构浓缩成一张图（接入层 → 容器层 → 核心引擎层 → 模型层）：

[查看 pi-harness 学习系统架构图（ProcessOn 在线可编辑）](https://www.processon.com/view/link/6ad082004f05267c494c97ba)

## 三、两个最关键的坑：CORS 与 CSP

![浏览器安全策略的两道墙](/img/posts/pi-harness-wslc-learning-system/3-cors-csp-walls.jpg)

整个系统最隐蔽的坑不在 pi，而在**浏览器安全策略**。控制台是本地 `file://` 打开的纯静态页面，要访问容器里的 JupyterLab 服务，撞上两道墙。

### 坑一：跨源检测被 CORS 拦

`file://` 页面（origin 是 `null`）用 `fetch` 访问 `http://127.0.0.1:8888`，浏览器默认拦截。解法是在 Jupyter 启动参数里放开来源：

```
--ServerApp.allow_origin='*'
```

验证：响应头出现 `Access-Control-Allow-Origin: *`，就绪检测才能工作。

### 坑二：iframe 嵌入被 CSP 拦

控制台用 iframe 嵌入 JupyterLab，但 Jupyter 默认响应头是 `Content-Security-Policy: frame-ancestors 'self'`——**只有同源页面才能嵌入它**，`file://` 页面直接被拒。查源码发现官方留了后门：`settings['headers']` 里显式给出 `Content-Security-Policy` 就能覆盖默认值。于是加一份配置文件：

```python
# jupyter_server_config.py
c.ServerApp.tornado_settings = {
    "headers": {"Content-Security-Policy": "frame-ancestors *"}
}
```

验证：重启后响应头变成 `frame-ancestors *`，iframe 嵌入成功。

**经验**：这类配置尽量用配置文件而不是命令行参数——Windows 下 PowerShell → wslc exec → bash → jupyter 层层引号嵌套，命令行传 JSON 参数是引号地狱，配置文件一劳永逸。

## 四、验证闭环：从 build 到 200

![全链路验证闭环](/img/posts/pi-harness-wslc-learning-system/4-verify-checklist.jpg)

整个系统不是"写完就算"，而是每一步都实测：

| 验证项 | 方法 | 结果 |
|---|---|---|
| 镜像构建 | `wslc build -t pi-harness:latest .` | 导出并命名成功 |
| 容器运行 | `wslc run -d -p 127.0.0.1:8888:8888` | Running，端口映射正确 |
| 后端就绪 | `GET /api/status` | 200 |
| 学习界面 | `GET /lab`、Notebook API | 均 200 |
| 环境完整 | 容器内 `pi --version` | 1.2.0 |
| 浏览器策略 | 响应头检查 | CORS `*`、CSP `frame-ancestors *` |

有一个细节值得说：本机没有 Python，博客配图压缩脚本跑不了。我的替代做法是**换验证通道但不降验证标准**——用 `node --check` 校验控制台内联 JS 语法，用标签配对计数检查 HTML 结构，一样能发现问题。

## 五、总结：这套思路怎么复用到其他开源工具

回头再看，这个系统的本质是：**给任意一个开源 CLI/工具包上"镜像 + 入口 + 关闭 + 就绪检测"四层壳**。换个工具（比如别的 Agent、数据引擎、模型服务），只需要替换三处：

1. **镜像层**：换基础镜像和要安装的 CLI，JupyterLab 部分原样保留
2. **健康端点**：换成新工具的 HTTP 健康检查路径（很多工具都有 `/api/status` 或 `/health` 之类）
3. **预置 Notebook**：换成新工具的实验脚本

入口脚本、控制台的就绪检测/锁定逻辑、关闭命令——**全部原样复用**。

可复用的核心清单：

- 幂等入口：构建 → 启动 → 等待 → 打开，四步全部可重复执行
- 就绪驱动 UI：健康端点轮询 + 状态胶囊 + 未就绪锁定，杜绝"白屏式"学习体验
- 配置走文件：CORS/CSP 等安全策略用配置文件，避开命令行引号地狱
- 验证换通道不降标准：没有 Python 用 node 校验，没有浏览器用响应头检查

这套系统跑通后，我对 pi 的理解反而比读源码更扎实——看着它在 Notebook 里把一次任务拆成工具调用、再把结果合并回上下文，机制一下就立体了。**最好的学习环境，是让工具本身变成你的实验台。**

如果你也在学习某个开源 Agent 或工具链，这套"四层壳"思路可以直接抄作业。
