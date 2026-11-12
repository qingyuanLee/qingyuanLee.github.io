---
layout: post
title: "把「没解决」的三条真修掉：一次根因纠错、token 代价与内核级进程兜底"
subtitle: "第一期结尾的元素漏检、no_content 误报、代理硬依赖，加上第二期的孤儿进程；逐条复跑后发现根因判断错了、修法窄得多，而全量修复会把首屏 token 抬到 5.45 倍"
date: 2026-11-12 16:00:00 +0800
author: 清源
header-img: img/post-bg-hacker.jpg
catalog: true
tags: [MCP, CDP, 无头浏览器, Python, 性能评估, 工程实践]
---

## 引子：「没解决」有两种，别混在一张清单里

[第一期](/2026/11/11/grip-obscura-mcp/)把 CDP 兼容层做完，结尾留了三条：元素漏检、SPA 误报 `no_content`、代理是硬依赖。[第二期](/2026/11/12/grip-obscura-benefit/)做完边界体检和收益验收，又添了一条「客户端强杀服务时自动拉起的 Obscura 可能残留」，并顺手把这两组都归进「上游特性，这期没动稳定链路」。

这回的问题很直接：**这几条在当前项目里究竟能不能修？**

答案比分条判定更有意思：

- **元素漏检能修，而且修法比想象中便宜得多 —— 因为第一期写的根因是错的。**
- **SPA 误报能修**，只要在出快照前让 DOM 事实覆盖分类器意见。
- **代理硬依赖改不了根因**，只能改失败模式。
- **孤儿进程在 Windows 上能彻底解决**，但不能用更多 Python 代码，得把责任交给内核。

还有一个绕不开的代价：把元素发现补齐之后，复杂页首屏 token 会涨 3–5 倍，HN 从 643 涨到 3,507 —— 直接把这个项目最在意的指标吃回去。所以最终不是「修或不修」，而是「什么时候修」。

前两期一篇造「假象」让 grip 以为对面是 Chrome，一篇拆「口径」让自己别以为省了 19 倍。这一期拆的是**根因**，顺便发现：修复本身也有成本，得用同一把尺子量。

## 一、先把根因重新测一遍：不是「没有布局引擎」

第一期表格里那行是这么写的：

> 布局能力：`document.querySelector('a').getBoundingClientRect()` 全是 0 → grip 依赖几何过滤的元素发现大面积漏检。

现象没问题：HN 230 个候选只发现 1 个，IANA 70 个发现 0 个，example.com 1 个发现 0 个。但「全是 0」是从单个元素外推出来的。这次改成**逐条件计数**：把 grip 收集脚本里的 `gripIsHidden()` 拆成几条判断，对页面真实候选元素逐个跑，看看到底卡在哪一步。

```text
gripIsHidden(el) 的三道关
  1. el.offsetWidth === 0 || el.offsetHeight === 0
  2. el.checkVisibility() === false
  3. rect 宽高为 0 / 完全在视口之外
```

实测结果：

| 页面 | 候选 | 真实 rect | 卡在 `offset*` | 卡在 `checkVisibility` | 卡在 rect |
| --- | ---: | --- | ---: | ---: | ---: |
| example.com | 1 | 有效 | 1 | 0 | 0 |
| Hacker News | 230 | `[130, 10.6, 213, 25.6]` 等真实坐标 | **229** | 0 | 0 |
| IANA help | 70 | 有效 | **70** | 0 | 0 |
| Python docs | 113 | 有效 | 80 之外另有 33 个 `display:none` 祖先 | — | — |

**Obscura 返回的是真实坐标**；恒为 0 的是 `offsetWidth` / `offsetHeight` 这两个属性没实现。grip 那句 `el.offsetWidth === 0 || el.offsetHeight === 0` 就把几乎所有链接和按钮判成隐藏了。所谓「没有布局引擎」是个过度概括，准确说法是「几何属性只实现了一半」。

这个区别不是学术上的：如果真没有布局，就得伪造一套坐标体系、还要担心坐标点击对不上；而实际上**从已有的真实 rect 反推 `offset*` 就够了**，而且 grip 的 `click` 本来就走 JS `el.click()`，压根不依赖坐标。第一期设想的「`click_at` 走坐标点击」完全不需要。

## 二、修完立刻撞上新问题：补齐发现率会把省下的 token 还回去

修法落在 `session.py` 的 `LAYOUT_SHIM_JS`：一段注入到页面里的 JS，重定义 `HTMLElement.prototype.offsetWidth/offsetHeight` 的 getter，优先返回引擎原值，为零时用 `getBoundingClientRect()` 的宽高兜底，rect 也为空才合成一个最小盒子。四个细节是刻意的：

- 先看 `getComputedStyle()` 的祖先链，`display:none` / `visibility:hidden` / `opacity:0` 一律保持隐藏，不做无差别放行；
- `checkVisibility()` 只在原生判定为 false 且确实有几何时纠偏；
- 脚本幂等（`window.__gripLayoutShim`），每次快照都重装一次 —— 导航会重建 JS 上下文，补丁会掉；
- 只在候选元素里确实存在零几何时安装，接真实 Chrome 时 `not-needed` 直接返回。

`scripts/verify_first_post_fixes.py` 用真实链路复跑（`cl100k_base` 统一计数）：

| 页面 | DOM 候选 | 修复前发现 | 修复后发现 | 精简快照 | 全量发现 | token 代价 |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| example.com | 1 | 0 | **1** | 355 | 369 | 1.04× |
| Hacker News | 230 | 1 | **230** | 643 | 3,507 | **5.45×** |
| IANA help | 70 | 0 | **70** | 387 | 1,376 | 3.56× |
| Python docs | 113 | 10 | **80** | 420 | 1,364 | 3.25× |

Python docs 停在 80/113，剩的是被 `display:none` 祖先藏起来的元素，shim 不放行。（表中「精简快照」是 `verify_first_post_fixes.py` 这一次运行的口径：HN 643 tokens；第二期 `benefit_benchmark.py` 那次是 659，复跑是 642。页面内容与脚本采样时点不同会有几十 tokens 的波动，本表的对比始终在同一脚本、同一次运行内完成。）

这张表同时坐实了第二期那句猜测：HN 229 个链接只发现 1 个，省下的 token 里确实有一部分是**丢链接**换的。把它们补回来，首屏就要付 5.45 倍 —— 3,507 tokens 已经超过方案文档「约 2,000 tokens」的目标。项目的初始需求是「减少 Agent tokens」，把一条修复默认打开反而违背它。

所以 `GRIP_LAYOUT_SHIM` **默认关闭**，改成第一期建议的那条兜底：`click` / `type` 抛 `ELEMENT_NOT_FOUND` / `STALE_REF` 时，才注入 shim → 重拍一次快照 → 重试同一动作。平时保持精简视图，只有真的要交互、且现有 ref 不够用时才付这笔钱。

实测：example.com 首屏仍是 987 字符、`INTERACTIVE:` 为空，`click("Learn more")` 自动兜底并跳到 `iana.org/help/example-domains`。需要每轮全量 ref 时手动设 `GRIP_LAYOUT_SHIM=1`，接受上表代价。

按第二期定下的全局共识，WorkBuddy 里涉及交互仍一律走官方 `obscura` MCP 的 `browser_*`。这次补的是项目自身的能力，不改变分工 —— 一条修复不该推翻一个基于任务链做的决策。

## 三、SPA 误报：让 DOM 事实覆盖分类器意见

`STATUS: no_content (recovery: retry)` 是 grip 分类器的**意见**，不是 DOM 的事实。第二期把它列为「上游特性」，其实在 MCP 层就能拦住误导性输出。

`scripts/mcp_probe.py` 里正好有稳定复现：HN 输入 `lightpanda` + Enter 跳到 `hn.algolia.com/?q=lightpanda`，grip 判成 `no_content`，而页面正文有 **7,757 字符 / 67 个内容块**。Agent 看到 `recovery: retry` 会去重新导航 —— 一次纯浪费的轮次。

`_correct_no_content()` 的做法是只在状态行出现时多跑一次只读 JS：

```python
_TEXT_PROBE_JS = "JSON.stringify({chars: ..., blocks: ..., title: ...})"
# 正文 >= 200 字符且存在内容块 -> 换掉 STATUS 行
# 否则保留分类器原判
```

替换后的提示是明确可执行的：`[note] grip classified this page as no_content, but the DOM has 7757 chars of text across 67 content blocks; the page is not empty. Use read() for the body; do not retry the navigation.`

三个边界都保住了：页面真的空、探针本身失败、状态行压根不存在时都保持原样，且**干净的快照不额外消耗任何 CDP 调用**。这比「调分类器的阈值」便宜，也没有动上游包。

## 四、代理硬依赖：改不了根因，只能改失败模式

这条是四条里唯一真正「改不了」的。本机是 fake-ip DNS，Obscura 不带 `--proxy` 会直接 `SSRF blocked 198.18.x.x`，这是引擎侧的安全策略，MCP 层没有绕过它的正当理由。

能做的是把失败从「两次工具调用后才 TLS 报错」提前成一条明确告警：

- 自动拉起前用 `socket.create_connection()` 探代理端口，不通就 warning，并把原因写清楚；
- `status` 报出 `proxy` / `proxy_source` / `proxy_reachable`，排障时不用猜；
- 可选 `GRIP_PROXY_INHERIT=1` 从 `HTTPS_PROXY` 等环境变量继承。

这里踩到一个配置坑：宿主的 `HTTPS_PROXY` 指向 `127.0.0.1:53621`，而 Obscura 实际用的是 `7897`。**默认继承会把本来正确的链路带偏**，所以必须是显式 opt-in，而且非 HTTP 的 `socks5://` 系统代理要跳过 —— `serve --proxy` 只吃 HTTP 正向代理。

## 五、第二期的孤儿进程：让内核而不是 Python 兜底

第二期修过一次进程回收：`start()` 超时/失败时补 `_reap_obscura()`，`close()` 从只 `terminate()` 改成 `terminate() + wait()`（超时 `kill()`）。但那只在 `close()` 被正常调用时有效。MCP 客户端强杀服务进程 —— 编辑器关闭、`taskkill /F`、stdio 管道断开、宿主崩溃 —— Python 没有任何机会执行清理，那个 `obscura.exe serve` 就留着占 CDP 端口，后面每次连接都撞上它。

加更多 Python 代码解决不了「Python 根本没运行」的问题。Windows 上有现成机制：

```text
CreateJobObjectW(NULL, NULL)
  -> SetInformationJobObject(..., JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE)
  -> AssignProcessToJobObject(job, Popen._handle)
  -> 进程退出时 OS 关闭最后一个 job handle
  -> 内核杀掉 job 内所有进程（含孙进程）
```

关键在「最后一个 handle」：job handle 由本进程持有、默认不可继承（不会被 Obscura 拿到），父进程无论怎么死，OS 关闭句柄时兜底一定发生，不依赖任何 Python 清理逻辑。正常退出时顺序仍是先 `terminate + wait` 再 `close()` job，后者顺手清掉可能存在的孙进程。

`process_guard.py` 把它封装成 `KillOnCloseJob`，`_start_obscura()` 在 `Popen` 成功后 attach 一次，`status` 新增 `job_object` / `job_object_supported` / `job_object_attached` / `job_object_error` 四个字段。降级路径也写死了：非 Windows、父进程已被宿主放进不允许嵌套的 job（`ERROR_ACCESS_DENIED` / `ERROR_NOT_SUPPORTED`）、或 `GRIP_JOB_OBJECT=0` 时，自动回退到原来的回收方式并给出 warning，不影响主链路。

`scripts/verify_kill_on_close.py` 用 9333 / 9334 两个专用端口起真实 MCP 式父进程，等它报出自动拉起的 Obscura，然后 `parent.kill()` 硬杀（无 `close()`、无 `atexit`），再看引擎是否还活着：

| 配置 | 强杀前引擎存活 | 强杀后残留 |
| --- | --- | --- |
| `GRIP_JOB_OBJECT=1`（默认） | 是 | **否** |
| `GRIP_JOB_OBJECT=0`（对照） | 是 | 是（脚本随后清理） |

对照组那一行很重要：它证明不是宿主环境顺带把进程树收走了，而是 Job Object 真的在起作用。`scripts/test_regressions.py` 里另有 8 项纯 Python 检查，其中一项直接验证「关闭 job handle 会杀掉 attach 进去的子进程」这一内核语义。

## 六、交付物与仍未解决的

这一期动的是稳定链路，落地清单：

| 产出 | 内容 |
| --- | --- |
| `src/grip_mcp/session.py` | `LAYOUT_SHIM_JS`、`_correct_no_content()`、`_act_discoverable()` 按需兜底、`_dom_probe()`、代理探测与继承、`status` 新字段 |
| `src/grip_mcp/process_guard.py` | 新增，`KillOnCloseJob` 封装 Job Object 内核兜底与降级 |
| `scripts/verify_first_post_fixes.py` | 真实链路复跑：发现率、token 代价、按需点击、代理状态；数据 `.runtime/first_post_fixes.json` |
| `scripts/verify_kill_on_close.py` | 硬杀父进程的对照实验；数据 `.runtime/kill_on_close.json` |
| `scripts/test_regressions.py` | 27 → 43 → **51 项**，新增 shim / no_content / 代理 / Job Object 四组 |
| `README.md` | 「遗留问题的评估与处理」重写，新增第 4 节孤儿进程；两期文章同步勘误 |
| `dist/` | wheel 与 sdist 重建，已验证包内是修复后代码 |

回归：`compileall` 通过、**51/51 回归通过**、`--check` 通过、`mcp_probe.py` 12 工具全通（含 `no_content` 误报被改写后的实际返回）、`benefit_benchmark.py` 复跑确认默认模式 token 无退化（355 / 642 / 387 / 420）。改动未提交 git。

**仍未解决的**，按优先级：

- **P0 收益可达**：Streamable HTTP 传输没做，方案文档推荐的豆包自定义连接器路径仍未兑现；工具默认接入状态在评估时还被笔记误记过一次。
- **代理硬依赖**：引擎策略，只能提前探测和明确报错。
- **验收口径**：方案文档里的「19×」仍是相对原始 HTML 的算式，没按实测基线分列改；「单会话 + DELTA」「必须桥接」这两条实测结论也还没回写进设计文档。
- **非 Windows / 不允许嵌套 job 的环境**：强杀仍可能残留，建议外部托管 `obscura serve` 并设 `GRIP_AUTOSTART=0`。
- **全量发现的 token 代价**：`GRIP_LAYOUT_SHIM=1` 下复杂页 3–5×，还没做自适应 token 预算来缓解。

另外，前两期文章都加了勘误块并互引到这一期，把「没有布局引擎」「元素发现率 0%–8.8%」「这期没动稳定链路」这几处过时结论标清楚。标题保留原样（已发布文章的标识不动），但在文首明确说明它被推翻 —— 与其悄悄改掉，不如留一份可见的纠错记录。

## 写在最后

三期的教训可以串成一条：第一期是**方案文档里的骨架只验证了思路，没验证跑得起来**；第二期是**数字只验证了算式，没验证对手是谁**；这一期是**「没解决」清单只标了状态，没区分「没查清」和「查清了但改不了」**。

元素漏检属于前者。当时从单个元素外推出「没有布局引擎」，于是连修法都往复杂的方向想（伪造布局、坐标点击）；真去逐条件计数，发现只是两个属性没实现，二十行 JS 就够了。代理依赖属于后者，再怎么改代码也绕不过引擎的安全策略，能做的只有把失败提前、说清楚。

还有一层更实际的：**修复要用项目自己的核心指标再量一遍**。补发现率无疑是「修好了」，可它让 HN 首屏涨到 5.45 倍，直接违背「减少 Agent tokens」这个初始需求。最后选的不是最强的修复，而是「默认不修、需要时才修」——把成本推迟到真正付得起的那一次调用上。判断一个 bug 该不该现在修，光看能不能修是不够的。

代码、两个新验证脚本、51 项回归和全部原始数据都在 [grip-obscura-mcp](https://github.com/qingyuanLee/grip-obscura-mcp)，README 里记了每一个开关和每一条实测结论。

---

> **前两期**：[第一期 · 把 Grip 接上一个没有布局引擎的浏览器](/2026/11/11/grip-obscura-mcp/)（CDP 兼容层实录，文首已加勘误） · [第二期 · 跑通之后才算账](/2026/11/12/grip-obscura-benefit/)（边界修复与收益验收，文首已加更新）
