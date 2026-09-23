---
layout: post
title: "把幕布变成大模型的第二大脑：我自研 mubu-mcp 的实战记录"
subtitle: "一个 MCP 连接器，让笔记从'沉睡的仓库'变成'AI 的工作记忆'"
date: 2026-09-22 10:00:00 +0800
author: 清源
header-img: img/post-bg-mubu-mcp.jpg
catalog: true
tags: [MCP, 幕布, AI Agent, 工具集成, 开源]
---

> 封面图：MCP 协议像一条数据高速公路，把大模型与笔记工具连成一体。

## 引子：一个笔记爱好者的痛点

![图：笔记散落在幕布里，AI 却"够不着"](/img/posts/mubu-mcp/pain-point.jpg)

过去三年，我把几乎所有的方法论、项目复盘、学习笔记都写进了幕布——需求实现五步法、DDD 笔记、Java 问题点、GRAI 复盘、个人商业系统……它们像一座越积越高的知识仓库，结构清晰，随时可查。

但有个问题越来越刺眼：**这些笔记大模型读不到**。

当我想让 AI 基于我的笔记做事时，只能手动复制粘贴——把整段大纲粘进对话，丢失层级；把文档导出成文本，再喂给模型。一次两次可以，但这不是"系统"，是"手工劳动"。

我需要的，是让 AI 能像我自己一样：**直接打开幕布，读文档、建文档、改文档**。这就需要一个协议层的桥梁——MCP。

## MCP 是什么：给 AI 装上的"USB 接口"

![图：MCP 就像 AI 的 USB 接口，即插即用接入各种工具](/img/posts/mubu-mcp/mcp-usb.jpg)

MCP（Model Context Protocol）是 Anthropic 在 2024 年底提出的开放协议，目标是为大模型提供一个标准化的"工具接口"。

类比一下：USB 协议统一了外设连接，让任何鼠标、键盘、U 盘都能即插即用。MCP 想做的事类似——**统一 AI 与外部工具/数据源的连接方式**。任何一个工具，只要实现一个 MCP Server，任何支持 MCP 的客户端（豆包、Claude Desktop、Cursor 等）就能直接调用它。

它有三个角色：

- **Host（宿主）**：运行大模型的应用，如豆包、Claude Desktop
- **Client（客户端）**：宿主内置的协议客户端，负责与 Server 通信
- **Server（服务端）**：把具体工具能力包装成标准接口的进程，如"幕布 MCP Server"

如果从协议栈的角度看，MCP 可以拆成自下而上的 **五层分层架构**：

| 层级 | 名称 | 职责 | 示例 |
|---|---|---|---|
| L1 | 应用层（Host） | 运行大模型、承载对话与业务逻辑 | 豆包、Claude Desktop、Cursor |
| L2 | 客户端层（Client） | 协议客户端，管理会话与连接生命周期 | MCP Client |
| L3 | 协议层（Protocol） | JSON-RPC 2.0 消息、生命周期管理、能力协商 | initialize / capabilities |
| L4 | 传输层（Transport） | 消息的物理通道，决定部署形态 | stdio、Streamable HTTP、SSE |
| L5 | 服务层（Server） | 暴露 Tools / Resources / Prompts 原语 | 幕布 MCP Server |

这五层各司其职：Host 负责"懂业务"，Client 负责"管连接"，协议层保证"能通信"，传输层决定"在哪跑"（本地进程还是远程服务），Server 最终把能力暴露成标准原语。理解分层后，很多困惑就清楚了——比如为什么我的两个 MCP 都要走 stdio：因为传输层选型（L4）决定了凭据、延迟与部署方式，本地工具天然适合进程管道。

> 🗺️ MCP 五层协议栈架构图（ProcessOn 可编辑原图，可放大查看分层容器与连线）：[查看高清架构图](https://www.processon.com/view/link/6ab340fb6a29601cdfc86a65)

架构上，Client 与 Server 通过 JSON-RPC 通信，可以走 **stdio**（本地进程管道，如本机运行）或 **SSE/HTTP**（远程服务）。我的两个 MCP 项目都用了 stdio 模式——因为笔记、画图这类工具往往需要本地凭据，且延迟低。

## mubu-mcp：把幕布变成 AI 能读写的知识库

![图：mubu-mcp 在 AI 与幕布之间建立双向通道](/img/posts/mubu-mcp/architecture-overview.jpg)

mubu-mcp 是我为幕布（mubu.com）开发的 MCP Server，通过豆包连接器以 STDIO 方式运行。它把幕布的 8 类核心操作全部封装成标准工具：

| 能力 | 工具 | 说明 |
|---|---|---|
| 创建 | `mubu_create_doc_from_markdown` | Markdown → 幕布大纲，目录不存在自动逐级创建 |
| 保存/写回 | `mubu_save_doc_markdown` | 按 doc_id 覆盖写入，内容真正落库 |
| 去重写入 | `mubu_upsert_doc_markdown` | 按「目录+名字」去重，同名覆盖 |
| 读取 | `mubu_get_doc` | Markdown 或原始 JSON 读回 |
| 搜索 | `mubu_search` | 关键词检索，可带内容 |
| 导出 | `mubu_export_markdown/opml/freeplane` | 三种大纲格式导出 |
| 图片 | `mubu_upload_image` | 本地图片自动托管到幕布存储 |
| 整理 | `mubu_move/rename/delete` | 文档与目录管理 |

这套能力意味着什么？举个例子：我这个博客里 9 篇文章，就是从幕布素材批量生成的——AI 直接读取幕布文档，提炼观点，写出文章。**笔记第一次成为 AI 可以自主使用的"内容供应链"**。

> 🗺️ 架构全景（ProcessOn 可编辑原图）：[查看高清架构图](https://www.processon.com/view/link/6ab25ad77783ce2a62c20247)

## 三个关键实现细节：难啃的骨头

![图：事件写入、TOS 直传、JWT 续期三大实现要点](/img/posts/mubu-mcp/implementation-details.jpg)

### 细节一：写入必须走事件接口

幕布没有现成的"创建文档并写内容"的公开 API。逆向后发现：内容只能通过 `POST /v3/api/colla/events` 以**事件流**方式写入——`create`（新节点，带 index/parentId/node/path）、`update`（改根文本）、`nameChanged`（重命名）。这些事件格式需要完全对齐网页版行为，否则前端不认。

### 细节二：图片直传火山引擎 TOS

幕布图片上传没有走普通表单，而是：`GET /v3/api/tos/sts` 取临时 AK/SK/SessionToken → 手写 **TOS4-HMAC-SHA256 签名**直传火山引擎 TOS（bucket `mubu-img`）→ `POST` 同步最近图片。签名计算是最容易踩坑的部分——头字段拼接顺序、日期格式、region 参数，错一个就 403。

> 顺带避坑：`POST /v3/api/document/upload_img_base64` 虽然能返回 fileId，但生成的对象无法通过公开 URL 访问（404），正式路径必须是 TOS 直传。

### 细节三：JWT 过期续期，避免撞限频

幕布登录 JWT 有效期约 30 天，但早期实现固定按 2 小时判过期，导致每 2 小时重登一次，直接撞上「Login Frequency」限频。修复方案：`_save_token` 解码 JWT 的 `exp` 作为过期时间；`_load_token` 发现本地记录过期但 JWT 仍有效时按 exp 自动续期，不再触发 login。

## 在豆包里怎么用：从"读笔记"到"写博客"的完整流水线

![图：幕布素材 → 博客文章的 AI 流水线](/img/posts/mubu-mcp/pipeline.jpg)

在豆包中接入 mubu-mcp 后，我沉淀出了一条可复用的工作流（已固化成 `mubu-blog-writer` 技能）：

1. **读**：`mubu_get_doc` 读取幕布文档，素材要点进入上下文
2. **想**：提炼 1 个主论点 + 3-5 个小节，规划文章结构
3. **写**：生成 Jekyll 文章（front matter + markdown 正文）
4. **配图**：题图 + 小节图（AI 生成）+ 架构图（ProcessOn MCP）
5. **发**：提交 GitHub Pages 部署并验证

整个流程中，幕布既是**素材库**，也是**结果归档库**——文章写完，还能用 upsert 把过程沉淀回幕布，形成"输入-输出-复盘"闭环。

## 已知的坑与边界：实测经验

![图：限频、缓存、重建——三个常见坑](/img/posts/mubu-mcp/pitfalls.jpg)

- **CLI 输出不稳定**：在 PowerShell 直接调用时 stdout 可能无输出，需重定向到文件再读
- **upsert 覆盖 = 删除+重建**：已有子节点的文档无法原地全量替换，doc_id 会变化
- **token 缓存编码**：Windows 下 `~/.mubu_token` 必须 UTF-8 读写，否则误判登录过期
- **markdown 层级映射**：幕布对"标题+列表"混用支持不佳，需用纯缩进嵌套列表才能保层级
- **只读不回写**：备注（`>` 引用）写回时暂不落库；checkbox 会映射为待办状态

## 开源：两个公开项目

![图：mubu-mcp 与 processon-mcp 两个开源项目](/img/posts/mubu-mcp/opensource.jpg)

这次实践沉淀出两个公开项目，都在我的 GitHub：

1. **mubu-mcp**（幕布 MCP 连接器）：https://github.com/qingyuanLee/mcp_mubu_latest
   能力：幕布文档的创建/写回/upsert/读取/搜索/导出/图片上传，含 JWT 自动续期、TOS 直传实现。

2. **processon-mcp**（ProcessOn 图形引擎连接器）：https://github.com/qingyuanLee/mcp_processon
   能力：让 LLM 主导画图——Mermaid 渲染可编辑架构图/流程图、思维导图、流程图直绘，返回预览图 + 在线编辑链接。

两个项目共同点：**把"创作型工具"变成 AI 可自主操作的能力**。幕布负责"文字与结构"，ProcessOn 负责"图形与表达"，合起来就是 AI 的内容生产闭环。

## 总结：笔记的终点是"被使用"

![图：从沉睡的仓库到被使用的工作记忆](/img/posts/mubu-mcp/conclusion.jpg)

记笔记的意义不在"记"，而在"用"。MCP 让这一层真正打通了：**笔记不再是沉睡的仓库，而是 AI 随时可读写的"工作记忆"**。

如果你也有大量沉淀在笔记工具里的知识，不妨试试用 MCP 把它接出来——你会发现，过去积累的每一条笔记，突然都有了被再次使用的机会。这大概就是 2026 年这个节点上，个人知识管理最有意思的打开方式。

---

*本文配图由 AI 生成；架构图由 ProcessOn 绘制（见文中链接）。项目地址：[mubu-mcp](https://github.com/qingyuanLee/mcp_mubu_latest) · [processon-mcp](https://github.com/qingyuanLee/mcp_processon)*
