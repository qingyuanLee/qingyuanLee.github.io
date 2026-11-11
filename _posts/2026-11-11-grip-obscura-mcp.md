---
layout: post
title: "把 Grip 接上一个没有布局引擎的浏览器：一次 CDP 兼容层的完整实录"
subtitle: "方案文档里的三行骨架，跑起来三处都不成立；Obscura 的 page socket 是空壳、按键会吞回包、快照头部永远是空字符串"
date: 2026-11-11 10:00:00 +0800
author: 清源
header-img: img/post-bg-hacker.jpg
catalog: true
tags: [MCP, CDP, 无头浏览器, Python, 工程实践]
---

## 引子：一份看起来很顺的方案，和三个跑不起来的假设

项目目录里躺着一份方案：把 [Grip](https://github.com/nikolas-sapa/grip-browser)（一个 CDP-native 的 Python SDK，能把整页压成两千 tokens 左右的语义快照）接到 Obscura（一个 Rust 写的轻量无头引擎，对外声称自己是 Chrome 145）上，再套一层薄薄的 MCP 封装，让 LLM 用 `snapshot / click / type` 上网。

三层解耦，画出来很漂亮：

```text
MCP 客户端 ──stdio──> grip-mcp（自建薄封装）
                          │ ws
                     ObscuraBridge（本次真正的活儿）
                          │ ws
                     Obscura ── CDP ──> 网页
```

方案里甚至给了 `server.py` 骨架。我照着它写，然后一条一条撞墙：

1. `Browser(cdp_url="http://127.0.0.1:9222")` —— `InvalidURI: scheme isn't ws or wss`。grip 把这串原样丢给 `websockets`，它只认 `ws://`。
2. 骨架的 `click` / `type` 分支复用 `snapshot` 分支里的 `page`，而那个变量只在 `snapshot` 分支赋值 —— 走到第二、三个工具就 `AttributeError`。
3. 骨架默认 `/devtools/page/<id>` 这条 per-page socket 存在。Obscura 上它是空壳，裸探测直接回 `-32601 "No page for session"`。

这三条都属于「写完就跑不起来」，还算便宜。真正花时间的是第四条，方案里只用一句「Obscura 的 CDP 覆盖不完整」带过：

**grip 假设对面是 Chrome，而 Obscura 只是长得像 Chrome。**

仓库：[grip-obscura-mcp](https://github.com/qingyuanLee/grip-obscura-mcp)。下面是这层差异到底长什么样，以及我怎么补的。

## 一、先测清楚：Obscura 的 CDP 到底缺什么

动手之前先探端点：`/json/version` 回 `Chrome/145.0.0.0`、protocol `1.3`、V8 `14.5.0.0`，看上去人畜无害。接着一条条验：

| 探测项 | 结果 | 影响 |
| --- | --- | --- |
| `/devtools/page/<id>` 直连 | `-32601 No page for session` | grip 的 per-page socket 假设失效 |
| browser socket + `Target.attachToTarget{flatten:true}` | `Runtime.evaluate` / `Page.navigate` / `Emulation.*` 全部 OK | 唯一可用路径 |
| target 注册表作用域 | **按连接隔离**：A 连接建的 target，B 连接 attach 报 `Target not found` | 「一条下游连接配一条上游连接」的方案直接死掉 |
| 事件信封 | 全部带 `sessionId`（`Page.frameNavigated` / `lifecycleEvent` / `Runtime.executionContextCreated` / `Network.*`） | grip 的根监听器收不到任何导航事件 |
| `Target.getTargetInfo` | 返回 `{"targetId":"browser","title":"","url":""}` | 快照头部 `PAGE:` / `URL:` 永远空 |
| 布局能力 | `document.querySelector('a').getBoundingClientRect()` 全是 0（`innerWidth 1280`、`innerHeight 720`、`dpr 1`） | grip 依赖几何过滤的元素发现大面积漏检 |

第三条最难缠。我第一版桥是「每个下游连接各开一条上游连接」，逻辑上最干净，结果 `attach to target page-1 failed: Target not found`。

原因不复杂：Obscura 把 target 注册表挂在连接上，不是你挂在浏览器上。所以桥必须**只留一条上游连接**，把所有下游都多路复用到它上面。

## 二、桥的设计：一条上游，若干假 page socket

第二版 `bridge.py` 干的事，说穿了就是「伪装成 Chrome 的连接模型」：

- 下游按 `ws://127.0.0.1:<随机端口>/devtools/browser` 与 `/devtools/page/<targetId>` 两种路径接进来，桥从路径里解析出 target；
- 上行：把客户端自己的 `id` 换成桥的全局 id，记下 `(client, 原 id)`，并在信封里**注入** `sessionId`；
- 下行：按 id 还原成客户端原来的 id，并**剥掉** `sessionId` 再发下去；
- 事件：带 `sessionId` 的按 session 找到绑定的下游客户端，剥掉后下发；没有绑定的直接丢；不带 `sessionId` 的根事件广播给 browser 客户端。

再补两个「Obscura 不做但客户端非要用」的地方：

**会话从哪来。** 优先复用 Obscura 自己 `Target.setAutoAttach` 出来的会话（轮询最多约 2 秒等它到），实在没有才显式 attach。反过来的话，grip 会拿到一个「命令能答、事件不来」的哑会话。

**空壳命令。** 对 page 客户端发来的 `Target.getTargetInfo`，桥里直接拦下，用 `Runtime.evaluate` 跑一句：

```js
JSON.stringify({t: document.title || '', u: location.href || ''})
```

再拼成 `targetInfo` 回给 grip。补上这一句之后，快照头部才第一次出现 `URL: https://example.com/` 和 `PAGE: Example Domain`。

还有一个隐蔽的开关：`Target.setAutoAttach` 必须把 `waitForDebuggerOnStart` 降级成 `false`。不降级，Obscura 建出来的新页面会一直冻在「等调试器 resume」的状态，而那个 resume 永远不会来。

桥写完之后，example.com 的快照是 355 tokens，正文、标题、URL 全对。

## 三、两个只有实跑才暴露的坑

这部分我原以为没有。静态看代码看不出来，跑起来才现形。

### 3.1 导航型按键会吞掉回包

用 Hacker News 的搜索框试 `type("Search:", "lightpanda")` + `press("Enter")`。页面确实跳到了 `https://hn.algolia.com/?q=lightpanda`，但 grip 抛：

```text
TimeoutError: CDP command Input.dispatchKeyEvent timed out
```

原因：Enter 触发表单提交的瞬间，页面已经开始导航，Obscura 把还没回完的 keyUp 应答丢了。页面动了，客户端却以为命令失败。

处理方式不是加超时时间，而是把「超时 / 网络错误」归类成**应答已丢失**：等一下、重读页面状态、把这一句写进返回文本，不把工具调用判死。

```python
except Exception as exc:
    if not _is_ack_loss(exc):
        raise
    await asyncio.sleep(0.5)
    text = await asyncio.wait_for(self._finish(page), timeout=60)
    return f"[{label}: engine lost the acknowledgement, page state re-read]\n{text}"
```

这条判断的代价是可能掩盖真正的失败。所以我在返回里显式标注，让调用方看得见「这次没收到引擎确认」。

### 3.2 代理偶发的 TLS 重置

本机是 fake-ip DNS，Obscura 不带 `--proxy` 会直接 `SSRF blocked 198.18.x.x`；带上代理（7897）之后，第一次 `open("https://example.com")` 还是碰到了：

```text
RuntimeError: Network error: ... client error (Connect): tls handshake eof
```

属于瞬时网络问题，做法是 `open` 遇到 `Network error` / `tls` 重试一次。同时把 MCP 层的错误边界从「只映射自己的异常类型」放宽到「任何异常都透出原文」—— 否则客户端只看得到 `Error executing tool open`，完全无法行动。

## 四、MCP 层为什么必须自建

先确认一件事：grip 自己带了一个 `grip-mcp` 入口（19 个工具，`grip.mcp.server:main`）。但它里面的 `_ensure_browser()` 是直接 `Browser()` 本地拉起 Chrome，**没有接受 `cdp_url` 的入口**。

也就是说，官方入口和我需要的「连到已经跑着的 Obscura」是互斥的。这不是配置问题，是设计取舍，所以薄封装层只能自己写。

`mcp 2.3.0` 这边还有一个当期改名：`mcp.server.fastmcp` 已经不存在（导入即报错，提示 FastMCP renamed to MCPServer），可用的写法是

```python
from mcp.server import MCPServer
server = MCPServer("grip-obscura", lifespan=lifespan)

@server.tool(name="click", description="...")
async def _click(target: str) -> str: ...
```

最后落了 12 个工具：

| 工具 | 作用 |
| --- | --- |
| `open` / `goto` / `snapshot` | 打开、跳转、重新快照（有基线就只回 `DELTA v1->v2`） |
| `click` / `type` / `press` / `scroll` | 交互，target 支持 ref（`e2`）或文本子串 |
| `wait_for` | 等文本/ref/选择器出现 |
| `read` / `links` | 正文分块、真实链接列表 |
| `screenshot` / `status` | JPEG 截图、端点与当前 URL |

其中 `links` 是一个绕道设计：grip 的链接列表来自元素发现，而元素发现依赖几何，Obscura 无布局时基本是空的。所以 `links` 不走 grip，改成桥上的 DOM 直查：

```js
JSON.stringify(Array.from(document.querySelectorAll('a[href]'), a => {
  const t = (a.innerText || a.textContent || '').replace(/\s+/g, ' ').trim();
  return [t.slice(0, 120), a.href];
}).filter(r => r[1] && r[1].indexOf('http') === 0))
```

在 example.com 上，grip 报 0 个交互元素，`links` 却能正常吐出 `Learn more  https://iana.org/help/example-domains`。

## 五、交付：uv 管环境、wheel 打包、stdio 配置

环境按约束走 uv，不碰系统 Python：

```powershell
uv python install 3.13      # 落到 uv 自己的目录
uv init --package --name grip-mcp --python 3.13
uv add grip-browser mcp     # 0.9.0 / 2.3.0
uv build                    # dist/grip_mcp-0.1.0-py3-none-any.whl（约 20 KB）
```

为了不出现「在我机器上能跑」，我把 wheel 装进一个全新的空 venv 再验一次：

```text
grip-obscura-mcp 0.1.0
CDP upstream : http://127.0.0.1:9222
CDP bridge   : ws://127.0.0.1:60951/devtools/browser
snapshot     : PAGE: Example Domain | URL: https://example.com/ | VIEWPORT: y=0/820
OK: grip -> Obscura chain works
```

自动拉起也单独验过：把端点指到 9333，服务自己 `obscura serve --port 9333 --proxy ...`，连上、抓页、退出时再把那个进程收回（端口确认释放）。只在自己启动的情况下才回收，外部已经在跑的实例不动。

MCP 标准配置：

```json
{
  "mcpServers": {
    "grip-obscura": {
      "command": "C:\\Users\\18133\\work\\headless-browser\\grip-mcp\\.venv\\Scripts\\grip-obscura-mcp.exe",
      "args": [],
      "env": {
        "GRIP_CDP_URL": "http://127.0.0.1:9222",
        "GRIP_AUTOSTART": "1",
        "GRIP_OBSCURA_PROXY": "http://127.0.0.1:7897"
      }
    }
  }
}
```

`GRIP_AUTOSTART=1` 表示端点没人监听时自己拉起；`GRIP_OBSCURA_PROXY` 只在自动拉起时生效。

## 六、验证记录与没解决的问题

跑通的链路（都是对着真实站点、走真实 stdio 协议）：

| 场景 | 结果 |
| --- | --- |
| example.com 快照 | `PAGE: Example Domain`、`URL: https://example.com/`、355 tokens |
| screenshot | 81868 字符 base64 JPEG，以 image 内容块返回 |
| HN 搜索框 type + Enter | 跳到 `hn.algolia.com/?q=lightpanda` |
| IANA 页 click `e1` | 跳到 `https://www.iana.org/help` |
| links | DOM 直查拿到真实链接 |

没解决的，也说清楚：

- **元素漏检**。没有布局引擎，`getBoundingClientRect` 恒为 0，grip 的几何过滤会把大部分链接、按钮筛掉 —— HN 只识别出搜索框，IANA 页只识别出一个链接。能识别到的 ref 点了确实会跳转，识别不到的就只能靠 `links` 或 `goto` 绕。
- **误报 `no_content`**。部分 SPA 结果页会被 grip 的分类器判成 `STATUS: no_content (recovery: retry)`，正文其实在。
- **代理是硬依赖**。fake-ip DNS 下不给代理，Obscura 直接 SSRF 拦截，这不是代码能修的。

下一步如果还要用这套东西，最值钱的一件事是给 `click` 补一条兜底：DOM 直查 + 文本匹配拿到元素，再 `click_at` 走坐标点击。现在的 `click` 只能点「恰好被 grip 看见」的那几个元素。

## 写在最后

这次最直接的收获不是「MCP 又做了一个」，而是重新认识了「兼容层」这三个字。

以前我会把它理解成 API 适配：字段名不同、格式不同，写个 adapter 就完事。这次不是。差异在**连接模型**上：人家按连接隔离 target、按 session 路由事件、per-page socket 是空壳。这类差异不是映射字段能解决的，你需要在对端造一套完整的假象，还包括伪造一条「本来不存在」的 per-page socket。

以及一个老规律又被验证：**方案文档里的骨架，通常只验证了「思路」，没有验证「跑得起来」**。三行代码里藏着三处假设，每一处都需要用真实端点去打一次才知道成不成立。这活儿没有捷径，只能一条一条探，然后一条一条补。

代码和全部探测脚本都在 [grip-obscura-mcp](https://github.com/qingyuanLee/grip-obscura-mcp)，README 里把这几条 CDP 差异，以及每个坑的复现方式都记了下来。
