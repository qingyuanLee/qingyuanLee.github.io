---
layout: post
title: "跑通之后才算账：grip-obscura 的六类边界修复、token 收益实测与两条查询通道的分工"
subtitle: "设计文档写「约 2000 tokens、相对原始 HTML 减少 19 倍」；实测首屏 355–659 tokens 确实更省，但把基线换成真实在用的阅读通道后只剩 3–4 倍 —— 真正的大头是骨架里根本没有的 DELTA"
date: 2026-11-12 10:00:00 +0800
author: 清源
header-img: img/post-bg-hacker.jpg
catalog: true
tags: [MCP, CDP, 无头浏览器, Python, 性能评估, 工程实践]
---

> **2026-11-12 16:00 更新（详见[第三期](/2026/11/12/grip-obscura-root-cause/)）**：本文的实测数字都是当期原值，不作改动；但三处结论后来变了 —— ①「元素发现率 0%–8.8%」被定位为 `offsetWidth`/`offsetHeight` 恒为 0（rect 其实是真实坐标），已可修复，只是全开会吃掉 token；②「SPA 误报 `no_content`」已用 DOM 正文复核消除误导；③「强杀客户端会残留 Obscura」已用 Windows Job Object 的 `KILL_ON_JOB_CLOSE` 兜住。第四节的验收表和第六节「这期没动稳定链路」据此更正。**工具分工的三级链不变**：WorkBuddy 里涉及交互仍一律走官方 `obscura` MCP。

## 引子：能跑，不等于值得用

[上一期](/2026/11/11/grip-obscura-mcp/)结束时，链路是通的：example.com 快照 355 tokens，HN 能输入并跳转，IANA 能点链接，截图也能以 image 内容块返回。但收尾时我留了三条「没解决」：元素漏检、SPA 误报 `no_content`、代理是硬依赖。

那三条是**能力边界**。还有另外三个问题当时根本没问：

1. 这条链路能长期跑吗？边界输入、断连、进程回收会不会慢慢把它磨死？
2. 方案文档里最诱人的那句「约 2,000 tokens、相比原始 HTML 中位数减少 19 倍」，到底成不成立？
3. 就算成立，它比**我本来就在用的那条查询通道**省多少？

这一期就是把这三笔账算完。顺序是：先体检并修掉六类问题，再建立同源基准量 token 与耗时，然后把基线换成真实替代品重新量一次，最后按初始需求逐条验收，并把结论落到工具分工上。

结论先放这儿：**绝对 token 量和稳态效率达标甚至超额，但 19× 这个口径站不住；真正值钱的收益来自实现里「单会话 + DELTA」这个设计文档骨架拿不到的改动；而交互能力不合格，最终让这条链路只承担「只读视图」这一层。**（后半句的项目自身能力在第三期补上了，但分工结论不变；19× 与 DELTA 两条结论至今成立。）

## 一、先做体检：一条能跑的链路里还埋着什么

方法还是上一期那套：不靠读代码下结论。先核对依赖的真实签名（grip 0.9.0 / mcp 2.3.0 / websockets），再跑 `--check` 和完整 stdio 探针，最后针对探针覆盖不到的边界写一次性诊断脚本去打真实端点。

结果是：**主链路没问题，`--check` 与 12 个工具的端到端探针全部通过**。出问题的全是边界：

| # | 问题 | 实测影响 |
| --- | --- | --- |
| 1 | CLI 硬覆盖 `GRIP_AUTOSTART` | `autostart=not args.no_autostart` 恒为 True；设 `GRIP_AUTOSTART=0` 仍自动拉起 Obscura，且没带 `--proxy`。环境变量形同虚设 |
| 2 | 桥的路径注册表泄漏 | `process_request` 用 `id(connection)` 存 path；一次普通 HTTP `/json/version` 探测就永久留下条目，id 被复用时会把真实 socket 误判为未知路径关掉 |
| 3 | 丢应答 / 断连后状态泄漏 | `_pending` 只增不减；上游断开后 `_futures` 没人唤醒，`attachToTarget` 白等 15 秒，还继续往死连接 `send` |
| 4 | 工具契约缺陷 | `read()` 返回全文无上限、传 `max_chars` 被静默忽略；`wait_for`/`scroll` 非法参数抛 `ValueError` 并在 stderr 打整段 traceback；`wait_for` 返回标 `Any` 导致 outputSchema 为 null；`screenshot` 降级写临时文件不清理，还把路径当成功结果 |
| 5 | 孤儿进程 | `start()` 超时/失败后留下无主 Obscura；`close()` 只 `terminate()` 不 `wait()` |
| 6 | 文档与实现脱节 | README 称 `read`「按块读取」、配置说明里的「形态二」无对应内容 |

挑两个值得展开的。

**第一个是配置被代码覆盖。** 上一期专门验过「外部托管 Obscura 时不要自动拉起」，结果 CLI 里那行 `BrowserSession(args.cdp_url, autostart=not args.no_autostart)` 把环境变量彻底吃掉了：`--no-autostart` 没传，就恒为 True。这类 bug 的讨厌之处在于它不报错，只是让你以为自己关掉了某个行为。修法是让 `autostart` 缺省为 `None`，此时才去读环境变量，只有 `--no-autostart` 强制关闭。

**第二个是桥里那个 `id()`。** 原实现为了把 websocket 路径传给 handler，在 `process_request` 里做了这么一句：

```python
def _remember_path(self, connection, request):
    self._paths[id(connection)] = getattr(request, "path", "/")
    return None
```

`_paths` 只在 `_handle` 里 `pop`。可任何 CDP 客户端都会先探一次 `/json/version`，那是个普通 HTTP 请求，不带 Upgrade，websockets 直接回 426 —— `process_request` 却照常执行，于是**一条 `/json/version` 记录永久留在字典里**。更糟的是 CPython 会复用对象 id：等哪次新连接恰好拿到同一个 `id`，它会被当成「上次那个 HTTP 探测」处理，直接关掉。

修法不是加清理，是把注册表整个删掉。websockets 的 `ServerConnection` 本来就带 `request.path`，`_handle` 里直接读就行。顺带把状态清理补齐：120 秒 `_PENDING_TTL` 惰性回收丢应答的请求、`_upstream_lost` 让断连后 fail-fast、`stop()` 与断连路径清空 pending/futures/sessions 并 cancel 还在等的内部请求。

工具契约那组改动看起来琐碎，但对 Agent 很实际：`read(max_chars)` 默认 20000 字符（0 为全文）并在截断时给出提示；非法参数统一转 `SessionError`，不再污染 stderr；`wait_for` 返回类型改成 `str`，outputSchema 才不为空；`screenshot` 降级不再泄漏临时文件，而是明确报错。

为了把这些钉死，新增了 `scripts/test_regressions.py`：27 项检查（当期值，第三期后已扩到 51 项），用纯 Python 假上游模拟 CDP，不需要 Obscura 二进制也不需要网络。修完重跑 `--check`、`mcp_probe.py`（12 工具全过）、27/27 回归、`compileall`，并用 `uv build` 重建了 wheel 和 sdist —— 原来的 `dist/` 里装的还是修复前代码，这种「交付物比源码旧一版」的事值得单独记一笔。

## 二、算 token 这笔账：先把口径定死

方案文档里两句话是这期的验收目标：

> Grip：CDP-native 的 Python SDK……将页面转换为约 **2,000 tokens 的语义快照**（相比原始 HTML 中位数减少 **19 倍**）。

> 操作后返回**新快照**，让 Agent 在每轮交互中只接收约 2,000 tokens 的语义视图，而非重新拉取全量 HTML。

`scripts/benefit_benchmark.py` 的设计只有一条原则：**同源**。四个页面（example.com / Hacker News / IANA help / Python docs）都走同一条 Obscura CDP 连接，分别取原始 HTML（`outerHTML`）、朴素正文（`innerText`）和 grip 快照，差异只来自后处理，不掺网络、引擎和时序。token 一律用 tiktoken `cl100k_base`，和 grip 内部估算用的是同一个编码器。

| 页面 | 原始 HTML | 朴素正文 | grip 首屏快照 | vs HTML | vs 正文 |
| --- | ---: | ---: | ---: | ---: | ---: |
| example.com | 755 | 330 | **355** | 2.1× | 0.9× |
| Hacker News | 11,467 | 1,132 | **659** | 17.4× | 1.7× |
| IANA help | 2,501 | 524 | **387** | 6.5× | 1.4× |
| Python docs | 7,513 | 659 | **420** | 17.9× | 1.6× |

**绝对量超额**：355–659 tokens，比 2,000 还省得多。**19× 没达**：2.1 / 6.5 / 17.4 / 17.9，中位数约 12×，简单页只有 2×。

效率这一侧更漂亮：Obscura 冷启 2.6s、桥启动 0.029s、grip 连接 0.002s、引擎工作集 60.1 MB、首屏 `open` 0.38–2.75s（网络主导）、稳态重快照 **4–17ms**、截图 0.06–0.16s。CDP 直连不依赖 Playwright/Puppeteer，「轻量低内存」这条是成立的。

但这里有个反直觉的发现。方案文档的骨架写着：

> 每个工具调用创建独立的 `Browser` 实例……避免会话交叉污染。

照这个骨架，每次工具调用都要重新 `open`，也就是每轮都要付 0.38–2.75s 和一次全量快照。而实际实现用的是**单会话 + baseline**：页面没变化时 `payload()` 只回一句

```text
DELTA v2->v3: no change
```

10 tokens。连续五轮读同一页，实测 395 / 699 / 427 / 460 tokens，比每轮全量快照省约 **78%**；HN 上 `open → type → re-read` 三步是 700 vs 1,977 tokens。稳态重读也从「重新抓页」降到毫秒级，差约两个数量级。

**也就是说，这期量到的最大收益，恰恰来自实现偏离了设计骨架的地方。** 骨架每次新建 `Browser`，没有 baseline，压根拿不到 DELTA。

不过第二节的结论还有个明显漏洞：`vs HTML` 这个基线是纸上对手。真实 Agent 不会把原始 HTML 塞进上下文，替代方案是已经转成正文或 Markdown 的阅读通道。而 `vs 正文` 只有 0.9–1.7×，example.com 上 grip 甚至比 `innerText` 还大。第三节就得把它换掉。

## 三、把基线换成真实替代品：两条通道同源对打

`innerText` 也不是真实替代品。我本来就在用的 `obscura-query` MCP，它的 `read` 返回的是**带完整链接的 Markdown**：HN 一页 10,255 字符、3,713 tokens（默认截到 6,000 字符仍有 2,162 tokens），而 `innerText` 只有 1,132 tokens —— 差的那部分正是链接。

于是写了 `scripts/compare_channels.py`：同一个 Obscura 二进制、同样四个页面、同一个编码器，把两条通道放在一张表上。

| 页面 | `obscura-query read`（默认 6000 字符） | 它的 `grep` 模式 | grip 快照 | grip 收益 |
| --- | ---: | ---: | ---: | ---: |
| example.com | 342 | 92 | **355** | 0.96×（倒亏） |
| Hacker News | 2,162 | 173 | **659** | **3.28×** |
| IANA help | 1,275 | 53 | **387** | **3.29×** |
| Python docs | 1,664 | 1,229 | **420** | **3.96×** |

结论比第二节诚实：**相对真实在用的阅读通道，grip 在复杂页省 3–4 倍**，简单页持平略亏。19× 是拿原始 HTML 当对手算出来的，不能这么记账。

`grep` 那列也值得看：按行过滤能把事实查询压到 53–173 tokens，比 grip 快照还省；但前提是关键词有选择性 —— Python docs 上用泛词「Python」命中 69 行，仍要 1,229 tokens。

**多轮才是分水岭。** `obscura-query` 每次调用都新起一个 `obscura.exe fetch`，没有会话、没有缓存、没有增量，「再看一眼」就要重抓整页：

| 页面 | grip 五轮累计 | `obscura-query` 五轮累计 | 倍数 |
| --- | ---: | ---: | ---: |
| example.com | 395 | 1,710 | 4.3× |
| Hacker News | 699 | 10,810 | 15.5× |
| IANA help | 427 | 6,375 | 14.9× |
| Python docs | 460 | 8,320 | 18.1× |
| **合计** | **1,981** | **27,215** | **13.7×** |

HN 那五行还实测了耗时：五轮全量重读 7.53 秒（单次 1.26–1.83s），grip 侧只是会话内五次重快照，按单次 17ms 计约 0.085 秒。**首次读页两边几乎一样快**（0.31–3.50s vs 0.38–2.75s，都由网络主导），差的是稳态。

反方向的能力差异同样重要。`obscura-query` 的 `search` 一次返回 1,425 tokens / 30.4s，`verify` 一次 1,981 tokens / 50.8s，且带对象相关性闸门和「≥2 个独立域命中 / 单一来源 / 未核实」的结论分级；grip 这边**没有任何检索或核查工具**，要完成「查资料并回答」，Agent 得自己拼 `goto 搜索引擎 → snapshot → links → goto 结果页`，而当期测到的元素发现率只有 0%–8.8%（第三期已定位并修复，但**无检索工具这一条到今天仍成立**）。

所以这两条通道**不是竞品**：

```text
obscura-query  优化「查询任务层」
   search / verify → 证据 → 结论分级
   省的是 Agent 的决策链与返工轮次；代价是单次 30–51s、无会话

grip-obscura   优化「页面状态表示层」
   语义快照 + DELTA
   省的是页面状态在多轮间的重复传输；代价是激进截断、无检索
```

还得给 grip 的收益打个折：`CONTENT` 被硬截到 2,000 字符，且当期 HN 229 个链接只发现 1 个。省下来的 token 有一部分是**信息截断和丢链接**换的，不是纯压缩。如果 Agent 要拿链接做下一跳，额外一次 `links` 会抵消一部分收益；而 Markdown 天然带 URL。反过来，`obscura-query` 的 `search`/`verify` 单次 30–51 秒是它最明显的效率短板。

第三期把「丢链接」这笔账量化了：把 229 个链接补回快照后，HN 从 643 tokens 涨到 3,507（**5.45×**），四页里复杂页普遍涨 3–5 倍。也就是说，这里的省 token 确实**可以**用信息量换回来，只是代价高到不适合默认开启。

## 四、逐条验收初始需求

把两轮实测合起来，对着方案文档逐条判：

| 初始需求 | 判定 | 依据 |
| --- | --- | --- |
| 首屏约 2,000 tokens | ✅ 超额 | 355 / 659 / 387 / 420 tokens |
| 相比原始 HTML 中位数减少 19× | ⚠️ 部分 | 2.1 / 6.5 / 17.4 / 17.9，中位约 12× |
| 每轮只回语义视图、不重拉全量 | ✅ 超额 | 无变化轮只回 10 tokens 的 DELTA；HN 三步 700 vs 1,977 |
| 轻量、低内存引擎 | ✅ | Obscura 工作集 60.1 MB，冷启 2.6s |
| CDP 直连、不依赖 Playwright/Puppeteer | ✅ | 桥 0.029s、grip 连接 0.002s，开销可忽略 |
| 三层解耦 + 统一 MCP 接入面 | ✅ 超额 | 3 个核心工具 → 12 个，并多出一层桥解决 CDP 差异 |
| `snapshot` / `click` / `type` 核心操作可用 | ⚠️ **当期不达标，第三期已修** | 当期元素发现率 0%–8.8%（HN 1/230、IANA 0/70、example.com 0/1）；补 `offset*` 后成为 230/230、70/70、1/1，但全量开启要付 3–5× token，故改为点击失败时按需兜底 |
| 豆包自定义连接器（HTTP / Streamable HTTP） | ❌ 未实现 | 项目只有 stdio；`mcp 2.3.0` 的 `MCPServer.run("streamable-http")` 其实已支持 |
| Grip 用 `Browser(cdp_url=...)` 直连 Obscura | ⚠️ 假设不成立 | 必须 ws 规范化 + 单连接多路复用桥接，否则连不上（上一期已证） |
| WorkBuddy STDIO 接入 | ⚠️ | 评估时记为「未启用」，后经核实是笔记误记，配置本就在线 |

**总评：部分达成。** token 的绝对量和执行效率成立，但收益口径被高估、交互能力不合格、接入范围只落地了一半。

## 五、从数据落到决策：一条三级工具链

数据摆完，工具分工就不需要拍脑袋了。

交互那条短板最直接：grip 靠几何过滤，而 Obscura 的 `offsetWidth`/`offsetHeight` 恒为 0（**勘误**：不是 `getBoundingClientRect()` 恒为 0，rect 返回真实坐标，见第三期），`click`/`type` 在多数页面无 ref 可用；Agent 得靠 `links` + `goto` 兜底，多出来的轮次会把 DELTA 省下的 token 还回去。所以先定了一条硬规矩：**凡涉及交互的浏览器需求，一律用官方 `obscura` MCP 的 `browser_*`，grip 不作交互通道。** 这条是任务链上的分工决策，第三期虽然补齐了项目自身的发现能力，也没有改它。

接着，既然 grip 在只读视图和多轮增量上确实领先，就把它提为**信息查询的默认工具**，形成三级链：

```text
第 1 级  默认      grip-obscura MCP
                   已知 URL 的读取 / 快照 / 多轮 DELTA
        │ 需要交互
        ▼
第 2 级  交互退化  官方 obscura MCP（browser_*）
                   点击 / 填表 / 登录 / 截图 / SPA 等待
        │ 同一目标试过 ≥3 种不同方案仍失败
        ▼
第 3 级  兜底      平台其他工具（内置检索、浏览器控制等）
                   必须说明已降级、列出试过的方案与失败原因
```

这里特意把「3 次」写成了**3 种不同方案**：换定位方式、换导航入口、换等待策略、换操作序列各算一种，同一个操作反复重试只算 1 个；计数跨轮累计，成功清零，换目标重新计数。否则「重试三次就换工具」会变成最容易的偷懒路径。

还补了一条能力边界：`grip-obscura` 没有检索工具，只能处理已知 URL。开放式关键词查询在第 1 级根本无法启动，也不算交互、不会触发第 2 级 —— 这类任务先用平台检索拿到 URL，**再回到第 1 级**读取。少了这一条，三级链在最常见的查询场景里会卡死。

顺带纠了个自己的错：笔记里一直写着「`grip-obscura` 保持 disabled」，实际去读 `~/.workbuddy/mcp.json`，它**根本没有 `disabled` 字段**，`status` 也确实返回了 CDP 9222、桥地址和 `obscura_managed: true`；反倒是 `obscura-query` 带着 `disabled: true`，当次会话里按全名查不到它的工具、调用直接报 `not found in the deferred tools index`。也就是说，兜底那一层当时得靠平台内置检索，不能指望自建通道。

**配置的真实状态和笔记对不上，本身就是个教训**：连「哪个工具能用」这种事实，都要打一次端点确认，不能靠上次写下的结论。

## 六、交付物与没解决的问题

这一期落地的是这些（未提交 git，工作区改动）：

| 产出 | 内容 |
| --- | --- |
| `src/grip_mcp/{cli,bridge,session,server}.py` | 六类问题修复：环境变量、路径注册表、状态清理、工具契约、进程回收 |
| `scripts/test_regressions.py` | 27 项（第三期后 51 项），纯 Python 假上游，不依赖 Obscura 二进制与网络 |
| `scripts/benefit_benchmark.py` | 同源 token / 耗时基准，含元素发现率统计 |
| `scripts/compare_channels.py` | 两通道对比，含 `read` / `grep` / 五轮重读 / `search` / `verify` |
| `.runtime/*.json` | 两轮原始实测数据，可复跑核对 |
| `README.md` | 新增「收益实测」「与 `obscura-query` MCP 的对比」两节 |
| `dist/` | wheel 与 sdist 重建（原包是修复前代码） |

没解决的，按优先级：

- **P0 让收益可达**：补 Streamable HTTP 传输，兑现方案文档推荐的豆包路径；工具默认接入状态也要落到实际配置上，而不是停在代码就绪。
- **P1 修交互短板**：给 `click` 补 DOM 直查 + CDP 派发（`links` 已验证这条路可行），或换有布局的引擎。否则「执行效率高」只在只读场景成立。
- **P1 校正验收口径**：把 19× 改成按基线分列的实测值，并在方案文档里补上「必须桥接」「单会话 + DELTA」这两条实测结论。
- **P2**：`CONTENT` 截断与 `read(max_chars)` 已配套，可再做自适应 token 预算。

**更正（第三期已动稳定链路）**：当初写「另外三个仍是上游特性，这期没动稳定链路」—— 元素发现漏检、SPA 误报 `no_content`、强杀时残留 Obscura —— 这三条随后都处理了：前两条在 MCP 侧修好，第三条在 Windows 上用 Job Object 的 `KILL_ON_JOB_CLOSE` 交给内核兜底（实测硬杀父进程后无孤儿）。唯一没变的是**代理硬依赖**，那确实是引擎侧策略。`GRIP_AUTOSTART=0` 修完之后才真正生效这一条仍然成立。

## 写在最后

上一期的教训是：**方案文档里的骨架，通常只验证了「思路」，没验证「跑得起来」**。三行代码里藏着三处假设，得拿真实端点一条条打。

这一期的教训是：**设计文档里的数字，通常只验证了「算式」，没验证「对手是谁」**。

「相比原始 HTML 减少 19 倍」不是假的，把 HTML 塞进上下文确实那么大。但真实 Agent 的对手是已经转成 Markdown 的阅读通道，换掉基线，收益从 19× 变成 3–4×，简单页还会倒亏。更有意思的是，那点「更省」里还有一部分来自把 `CONTENT` 截到 2,000 字符和丢掉 228 个链接 —— 它是信息取舍，不全是压缩算法的功劳（第三期把那些链接补回来后，HN 快照涨到 5.45×，坐实了这个判断）。

反倒是真正稳的收益不在任何压缩比上，而在架构：**单会话 + baseline 让「没变化」这件事只值 10 tokens**。方案文档骨架里「每次工具调用新建 `Browser`」这一句，恰好把这个收益排除掉了。

所以量收益之前，先花时间定义基线；判断一个工具该不该默认接入，也别只看它最擅长的那个指标，还得看它在整条任务链里能不能被直接用 —— 有没有检索入口、交互能不能完成、配置是不是真的开着。第一期造的是一套「假象」，好让 grip 以为对面是 Chrome；这一期拆的是另一套「口径」，好让自己别以为省了 19 倍。

代码、两个基准脚本和全部原始数据都在 [grip-obscura-mcp](https://github.com/qingyuanLee/grip-obscura-mcp)，README 里记了复跑方式和每一条实测结论。

---

> **续篇（第三期）**：[把「没解决」的三条真修掉：一次根因纠错、token 代价与内核级进程兜底](/2026/11/12/grip-obscura-root-cause/) —— 回到第一期结尾的三条问题，逐条复跑、修复、量化代价，并补上强杀场景的进程回收。
