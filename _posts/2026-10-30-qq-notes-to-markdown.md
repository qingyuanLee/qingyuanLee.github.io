---
layout: post
title: "十年 815 条记事：把 QQ 邮箱记事本整个搬进 Markdown"
subtitle: "从没文档的私有接口、1283 张会烂掉的外链图片，到一个能装进 Obsidian 的本地笔记库"
date: 2026-10-30 18:00:00 +0800
author: 清源
header-img: img/posts/qq-notes-to-markdown-cover.jpg
catalog: true
tags: [数据迁移, Markdown, Python, Agent Skill, 工程实践]
---

## 引子：一个没有导出按钮的记事本

我有个习惯，从 2016 年开始，所有临时想到的东西都往 QQ 邮箱的「记事本」里丢：一段排查思路、
一个订阅地址、某台服务器的登录信息、半夜想到的一句话、粘贴过来的半篇文章。

十年下来攒了 **815 条**。

直到某天我想在里面搜一个关键词，才发现这东西有多脆：网页版只能一条条点开看，没有导出，
没有全文搜索，图片还是挂在 `iwx.mail.qq.com` 上的临时链接——`notekey` 一变，图就永远回不来了。

这不是"想整理一下"的冲动，是一次**数据自救**：再过几年，这些链接会一个个烂掉，
而里面躺着的是我十年的技术轨迹。

于是有了这个项目：[qq-notes-exporter](https://github.com/qingyuanLee/qq-notes-exporter)。

![](/img/posts/qq-notes-to-markdown-cover.jpg)

## 先想清楚：这次迁移真正的敌人是谁

动手前我列了一下，阻碍只有三个，但每个都能让项目烂尾：

1. **没有公开接口**。记事本的读写走的是网页版自己的 `webnote` 私有接口，没有文档、没有 SDK，
   只有浏览器里那一串 Cookie 和一个 `sid`。
2. **图片是外链且会死**。815 条记事里 221 条带图，共 1283 个唯一图片链接，分布在
   `iwx.mail.qq.com`、CSDN、知乎、搜狐、博客园……每一个都是定时炸弹。
3. **数据是私人的**。这意味着任何"传到云端转换"的在线工具都不能用，
   也意味着这个项目的产出必须默认不进公开仓库。

对应的三个目标是：**可离线**（断网也能看图和搜索）、**可检索**（`grep` 能搜到）、
**可重复**（跑一次和跑十次结果一样，随时补跑）。

## 三步流水线

最终的实现是三个职责单一的脚本，串成一条流水线：

![](/img/posts/qq-notes-to-markdown-flow.jpg)

<small>图：三步流水线与各自的落盘产物。可编辑版：[ProcessOn](https://www.processon.com/view/link/6ae465922c1aa61cb2aa49ea)</small>

### 第一步：把数据先落盘，别急着处理

`POST /webnote/list` 分页拿元数据（`noteid` / `subject` / `ctime` / `cata_name`），
再 8 线程并发 `POST /webnote/read` 拿正文 HTML。

这里有个不起眼但关键的细节：记事本接口要求每次请求带一个 `r` 参数——
**毫秒时间戳拼 5 位随机数**。不带你不会报错，只会偶发拿到空 body，然后你会在
"为什么 815 条只拉到 700 条"上耗掉一晚上。

还有终止条件。`total_affected_count` 看着很美好，但它有时只在第一页返回。
第一版用 `len(collected) >= total` 判断，`total` 为 0 时就死循环了。
可靠的做法只有一个：**本页返回数量小于 `page_size` 就停**。

跑完先落盘成 `qq_notes_raw.json`，之后所有环节都从这份文件出发。
原始数据永不修改——这是后面一切"可回滚"的前提。

### 第二步：图片本地化，这是整个项目 80% 的复杂度

1283 个链接，两类来源，处理方式完全不同：

- **记事本自带的图**：`iwx.mail.qq.com/note/viewfile?...&notekey=...`，
  只要 URL 完整**不依赖登录态**就能取到
- **外站复制进来的图**：CSDN、知乎、搜狐、博客园……要防防盗链

设计上做了一个决定，事后证明是整件事的分水岭：**用清单（manifest）做唯一事实来源，
而不是边下载边替换正文**。

```python
def repl(m):
    rec = manifest.get(norm_url(m.group(2)))
    return m.group(1) + rec["path"] + m.group(3) if rec and rec.get("ok") else m.group(0)

new_content = IMG_SRC.sub(repl, content)
```

`manifest` 是 `url → {ok, size, path}` 的字典，正文改写**每次都从 raw 重建**。
于是整个流程天然幂等：跑 1 次和跑 10 次结果完全一致，中途失败直接重跑，
只重试失败的用 `--retry-failed`。

对比第一版"下载一张、`content.replace()` 一处"，那版跑到一半失败就再也对不上账了。

### 第三步：HTML 转 Markdown，别迷信现成库

记事正文是富文本粘贴产物，标签集合其实很窄：`p` / `br` / `h1-h6` / `ul` / `ol` / `li` /
`strong` / `pre` / `code` / `a` / `img` / `blockquote` / `table`。

最后没有引现成的 `html2text` 之类的库，而是用 `html.parser` 写了一个 150 行的状态机。
理由很实在：记事正文的标签集合就这么窄，通用库为了覆盖全部 HTML 会带来一堆用不上的分支，
反而在中文标点、列表缩进这些地方不好控制。手写的好处是完全可控，
坏处是你得自己处理 `<a href><img src></a>` 这种嵌套——图片已经单独输出了，
再套一层链接会被 Markdown 解析器截断。

输出按 `ctime` 归档成 `YYYY/MM/标题.md`，带 front matter：

```markdown
---
title: "一次线上 502 的排查记录"
date: 2020-11-03 22:41:10
category: 默认分类
star: false
noteid: _1024
---
```

图片路径用 `os.path.relpath` 按实际目录深度算，不写死 `../../../`——
这样把整个文件夹搬到任何地方都不会裂。

## 三个把项目救回来的细节

### 一、`&not` 不是实体，别 `html.unescape`

这是全场最坑的一个 bug，也是我认为最值得单独讲的一个。

记事本图片 URL 长这样：

```
https://iwx.mail.qq.com/note/viewfile?sid=...&notekey=xxx&name=1.jpg
```

看起来很普通。直到我对 HTML 属性做了一次 `html.unescape()`：

```python
html.unescape("...&notekey=abc")   # -> "...¬ekey=abc"
```

`&not` 是 **¬ 的合法 HTML 实体**。一次 unescape，所有 359 张记事本图片同时失效，
服务端返回 `token not valid`——报错信息完全指向不了编码问题，
你会以为是 Cookie 过期、接口改版、风控升级，唯独想不到是一个 ¬。

正确做法只还原真正会出现的那两种：

```python
u = raw.replace("&amp;", "&").replace("&quot;", '"')
```

**教训**：处理"看起来像 URL"的字符串时，任何通用解码函数都是危险品。

### 二、按属性替换，不要做字符串 replace

同一篇记事里常常同时存在 `//host/x.gif`（协议相对）和 `https://host/x.gif`。
用 `content.replace(url, path)` 替换时，前者是后者的**子串**，结果拼出：

```
https:qq_notes_images/_external/x.gif
```

图片全裂，而且只在部分文章出现，极难复现。

正确姿势是按 `src` 属性逐处匹配：

```python
IMG_SRC = re.compile(r'(<img\b[^>]*?\bsrc=")([^"]+)(")', re.I)
```

顺便说，文件名里的括号也会截断 Markdown 图片语法：`252(1).jpg` 会生成
`![](path/252(1).jpg)`，解析器在第一个 `)` 就结束了。所以非法字符集必须是：

```python
ILLEGAL = re.compile(r'[\\/:*?"<>|\r\n\t()\[\]]')
```

### 三、防盗链和"假 200"

外站图片最终失败 80 个，占 6.2%：51cto 反爬 567（23）、稀土掘金 502（14）、
yiibai/csdn 404（12）、搜狐 `p*.itc.cn` 403（约 20）、手机本地路径 `file://localhost/`（2）。

两个坑：

1. **反爬页经常是 `200 + text/html`**。只判断状态码，你会把一堆 HTML 存成 `.jpg`。
   必须连 `Content-Type` 一起判，`html` / `json` 一律算失败。
2. **三级退避能救回 94 张**：无 Referer → 站点根 Referer → 去掉 query
   （OSS 图片的水印参数常触发 567）→ http↔https。

## 数字说话

| 指标 | 数值 |
|---|---|
| 记事 | 815 条，2016-05-17 ~ 2026-03-18，覆盖 504 个日期 |
| 图片 URL | 1283 个唯一链接 |
| 下载成功 | 1203 张 / 116.4 MB（93.8%） |
| 正文改写 | 1339 处引用，221 条记事带图 |
| Markdown | 815 篇，本地图链接 1231 处，**缺链 0** |
| 耗时 | 拉数据约 1 分钟，图片约 6 分钟（8 线程），转换约 3 秒 |

体检报告现在是我每次跑完必看的东西，只盯一个指标：**本地图缺链 = 0**。
它意味着随便抽一篇打开，图都在。

## 把流程沉淀成 Skill，而不是留一堆脚本

脚本躺在本地，三个月后我一定忘了 `--retry-failed` 是干嘛的，也忘了 Cookie 该从哪个请求里复制。

所以这次多做了一步：把整条流程写成一个 **Agent Skill**（`SKILL.md` + `scripts/` + `references/`），
并且单独开了一个公开仓库。

Skill 和 README 的区别，我体感是这样：

- **README 写给"想用的人"**：怎么装、怎么跑、参数是什么
- **SKILL 写给"要替你干活的 Agent"**：什么时候该触发、必须遵守什么边界、验收标准是什么

所以 `SKILL.md` 里我特意写了几样 README 里不会有的东西：

- **触发条件**：用户说到"导出 QQ 记事本""图片链接本地化"就该命中
- **不可省略的步骤**：Cookie 只走环境变量，不写盘
- **九条踩坑清单**：表格形式，一条一行，避免 Agent 重走一遍我的弯路
- **验收清单**：缺链必须为 0、随机抽 3 篇能看图、`git status` 里没有凭证

装到 WorkBuddy 就是一行：

```bash
cp -r qq-notes-exporter ~/.workbuddy/skills/
```

之后说一句"把我的 QQ 记事本导成 markdown，图片一起下"就会自动走完整条流程。
`references/` 里还放了三份文档：接口字段与错误码、10 个坑的完整复盘、脱敏正则清单。

## 隐私是这类项目的第一性问题

这是自己的十年记录，误发一次就收不回来。所以三条硬规则写进了项目本身：

1. **`.gitignore` 默认拉黑所有产出**：`qq_notes_*.json`、`qq_notes_images/`、
   `qq_notes_md/`、`_cookie.txt` 全在里面。仓库里只有代码和文档。
2. **需要二次利用（写总结、喂给 AI、发文章）先脱敏**。`sanitize.py` 处理五类：
   手机号、邮箱、身份证/长数字串、`token=` 订阅链接、裸口令。
   一个容易漏的点：**文件名本身可能带手机号**（标题里就有），所以目录和文件名也要一起改。
3. **提交前自检**：

```bash
grep -rIn -E "skey|p_skey|sid=|1[3-9][0-9]{9}" . --exclude-dir=.git
```

还有一个边界：这个项目只调用网页版自身的接口，**只用于导出自己账号的数据**，
默认 8 并发已经很克制。技术上能做的事和应该做的事，中间隔着一条线。

## 写在最后：三条能复用的经验

这活儿看着是"写个爬虫导点数据"，实际上踩的坑分布在编码、正则、HTTP、文件系统、
隐私五个层面。抽出来看，有三条经验在任何数据迁移里都成立：

1. **先落盘原始数据，再谈处理**。`raw.json` 是整个项目的锚点，
   所有转换都是它的纯函数，出了任何问题都能回到原点重来。
2. **外部资源用清单驱动，不要流程驱动**。"下载-替换"这种耦合流程一旦中断就无法续跑；
   把状态抽到 manifest 里，流程就变成幂等的。
3. **一次性劳动要沉淀成 Skill**。脚本解决这一次，Skill 解决下一次——
   而且它记录的不是代码，是"当初为什么这么做"。

项目开源在这里：[github.com/qingyuanLee/qq-notes-exporter](https://github.com/qingyuanLee/qq-notes-exporter)（MIT）。
如果你也有一个十年没导出过的记事本、备忘录、收藏夹，希望这份折腾能让你少走点弯路。

十年前随手记的那句话，现在终于能被 `grep` 到了。
