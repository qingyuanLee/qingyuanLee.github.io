---
layout: post
title: "不写一行后端代码，我用 PWA + Gitee 仓库搭了个幕布平替"
subtitle: "从 Vercel 到 Netlify 全部注册失败，最后纯静态页面 + 浏览器直连 API 跑通了"
date: 2026-10-15 14:00:00 +0800
author: 清源
header-img: img/post-bg-github-cup.jpg
catalog: true
tags: [PWA, Gitee, 笔记工具, 纯前端, GitHub Pages]
---

## 引子：为什么要再造一个笔记工具

我用幕布记笔记三年了——工作日志、生活记录、读书笔记、项目复盘，两百多篇文档全在上面。之前我还写了个 [mubu-mcp](/2026/09/22/mubu-mcp-notes-to-ai/) 连接器让 AI 直接操作幕布。

但用得越久，越不放心：

- **数据在别人服务器上**：幕布说哪天关站或者改免费策略，我两百多篇笔记就悬了
- **导出格式不稳定**：从幕布导出来的 Markdown，图片路径、层级格式每次都有点小变化
- **移动端体验差**：幕布手机网页版很卡，又没有独立 App

我的目标很明确：**自己掌控存储，自己控制界面，免费部署，手机电脑都能用**。

## 技术路线探索：三条路都走不通

第一反应是"写个后端部署上去"。但现实给了我三巴掌：

### 1. Vercel：GitHub OAuth 登录失败

Vercel 是最好的 Serverless 平台，免费额度也够。但我用 GitHub 账号授权登录时，一直跳错误页——可能是网络问题，可能是账号问题，折腾半小时没进去。

### 2. Netlify：注册都注册不了

换 Netlify 试试，邮箱注册页面一直加载不出来。国内网络访问海外 Serverless 平台就是这么脆弱。

### 3. Gitee Pages：直接暂停服务了

想回国内方案——Gitee Pages。打开一看公告："因服务维护调整，Gitee Pages 暂停提供服务"。好嘛，连选项都没了。

三条路全堵死，逼着我想第四条路。

## 破局：纯静态页面 + 浏览器直连 API

思路突然通了：**为什么一定要有后端？**

Gitee API v5 本身就是 RESTful 的，浏览器直接 fetch 不就行了？token 存在 localStorage 里，不经过任何中间服务器。整个应用就是几个静态文件：

```
like-mubu-pwa/
├── index.html      # 入口
├── app.js          # 大纲编辑器（树渲染、键盘交互）
├── gitee-client.js # 浏览器端 Gitee API 适配层
├── style.css       # 样式（含移动端响应式）
├── sw.js           # Service Worker（PWA 离线缓存）
└── manifest.json    # PWA 清单
```

**核心就一个文件：`gitee-client.js`**。它在浏览器端覆盖了原来的 `window.api`，把所有 `/api/xxx` 请求转成对 `gitee.com/api/v5/...` 的 fetch 调用。没有 Node 服务器，没有 Express，没有任何后端。

部署就是把这堆文件推到 GitHub 仓库，开 Pages。完事。

## 关键设计决策

### 决策一：存储用 Gitee，不用 GitHub

国内访问 Gitee 比 GitHub 快太多。笔记仓库放在 `liqy153/daily_life`（私有），PWA 页面放在 `qingyuanLee/like-mubu-pwa`（公开）。

PWA 是纯静态的，不需要后端；Gitee API 通过浏览器 fetch 调用，token 在本地 localStorage，**代码里没有硬编码任何密钥**。仓库公开也安全。

### 决策二：图片怎么显示

这是踩坑最多的地方。Gitee 私有仓库的 raw URL（`gitee.com/xxx/raw/xxx`）直接访问会 403。试了三种方案：

1. **raw URL 加 `?access_token=`** → 还是 403
2. **fetch API contents → base64 → data URL** → 折叠再展开后时序问题导致不渲染
3. **Git Trees API raw endpoint**（`/api/v5/repos/xxx/raw/path?access_token=xxx`）→ ✅ 200，直接当 `<img src>` 用

最终方案就是第三种。在 `gitee-client.js` 里写了个 `fixGiteeImg` 函数，把所有 Gitee raw URL 自动转换成 API endpoint。

### 决策三：搜索怎么实现

Gitee 公开 API **没有**仓库内全文搜索。一开始我写了个"逐个读文件内容搜索"的版本，215 个文档触发了 Gitee 限速，全挂了。

优化了两步：

1. **Git Trees API 一次拿全量目录**：`/git/trees/main?recursive=1` 一次请求拿到所有文件路径，缓存到内存
2. **搜索只匹配标题/路径**：不读文件内容，纯前端过滤，毫秒级响应

现在搜索是即时的。如果以后要全文搜索，可以加个"全文搜索"按钮，在后台限速批量加载。

### 决策四：移动端适配

手机浏览器打开就是一坨——侧边栏占满半屏，文档挤在右边。加了响应式布局：

- 侧边栏变成抽屉，默认隐藏，点左上角 ☰ 滑出
- 文档头部按钮缩小、换行
- 图片 `max-width: 100%` 适配屏幕
- 字号调到 16px（iOS Safari 不会自动缩放）

Safari"添加到主屏幕"后就是个 PWA App，全屏运行，体验接近原生。

## 踩过的坑清单

| 坑 | 原因 | 解决 |
|---|---|---|
| 图片不渲染 | markdown 里 URL 前后有空格（`]( url )`）→ `%20` 404 | fixGiteeImg 里 trim |
| 改了代码不生效 | Service Worker 缓存旧 JS | 升级 CACHE_NAME + network-first 策略 |
| 搜索慢 | 逐个读文件触发 API 限速 | Git Trees API 一次拿目录 + 前端缓存 |
| 手机端看不到目录 | PWA standalone 模式下 prompt 输入 token 后没刷新侧边栏 | 加 token 就绪回调，输入后自动 refreshSidebar |
| 文档头部显示第一行内容 | parseMd 把 `# 标题` 同时当 title 和大纲节点 | 第一个 `#` 只作标题，不入树；标题用文件名 |

## 最终架构图

```
┌─────────────────────────────────────────┐
│  用户手机 / 电脑浏览器                     │
│  ┌───────────────────────────────────┐  │
│  │  PWA (GitHub Pages 静态托管)      │  │
│  │  ├── index.html / app.js         │  │
│  │  ├── gitee-client.js (API 适配)  │  │
│  │  └── localStorage: gitee_token   │  │
│  └───────────────┬──────────────────┘  │
└──────────────────┼──────────────────────┘
                   │ fetch (带 access_token)
                   ▼
        ┌───────────────────────┐
        │  Gitee API v5         │
        │  liqy153/daily_life   │
        │  (私有仓库)            │
        │  ├── notes/           │
        │  ├── journal/YYYY/   │
        │  └── images/          │
        └───────────────────────┘
```

**零后端、零服务器成本、零数据库**。所有数据在自己的 Gitee 仓库里，Markdown 格式，随时可以迁移。

## 反思：纯前端能做什么、不能做什么

### 能做的

- 个人笔记、文档管理、日记
- 任何"读多写少、数据量在几百个文件以内"的场景
- 不想花钱买服务器、不想维护后端的小工具

### 不能做的

- 多人协作（没有鉴权和冲突合并）
- 大数据量（215 个文档还好，几千个就不行了）
- 全文搜索（Gitee API 不支持）
- 实时同步（PWA 刷新才同步）

### 为什么这条路能走通

本质上是**把 Git 仓库当成了数据库**。Git 本身就是版本控制、就是文件存储、就是历史记录。我只需要一个界面来读写 Markdown 文件而已。

以前我总觉得"做个应用一定要有后端"，这次被逼着绕了一圈，发现对于个人工具来说，纯静态 + API 直连反而更简单——没有部署烦恼，没有服务器维护，没有数据库备份。GitHub Pages 一分钱不要，Gitee API 个人版免费。

**最好的后端，就是没有后端。**

---

> 项目地址：[qingyuanLee/like-mubu-pwa](https://github.com/qingyuanLee/like-mubu-pwa)（前端）
> 在线体验：`https://qingyuanlee.github.io/like-mubu-pwa/`（需要自备 Gitee PAT）
