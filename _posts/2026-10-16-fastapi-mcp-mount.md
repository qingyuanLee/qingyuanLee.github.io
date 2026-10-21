---
layout: post
title: "存量 FastAPI 挂 MCP，先看懂总体设计再谈踩坑"
subtitle: "fastmcp 4.1.0 单进程挂载：总体架构、认证职责划分、请求链路，再到 8 个落地坑"
date: 2026-10-16 15:00:00 +0800
author: 清源
header-img: img/posts/fastapi-mcp-mount-cover.jpg
catalog: true
tags: [FastAPI, MCP, fastmcp, Python, 架构设计]
---

## 引子：业务方一句话，老服务不能动

![](/img/posts/fastapi-mcp-mount-cover.jpg)

业务方想用 MCP 客户端（Claude、Cursor 这类）直接调我们特征服务的接口。但服务是个跑了好几年的存量 FastAPI：协程范式、业务 JWT 认证、日志中间件、lifespan 里挂着一堆原系统业务，动哪一块都心惊胆战。

需求翻译成人话：**在不重构、不换认证、不接管 lifespan 的前提下，把 MCP 作为"挂载服务"接进来，工具只暴露该暴露的接口。**

这篇文章不讲零散的坑，先讲清楚**总体设计**——单进程里发生了什么、认证职责怎么划分、一条请求怎么走完全链路——最后才是落地时踩过的 8 个坑。完整可运行代码在公开库：[github.com/qingyuanLee/fastapi-mcp-demo](https://github.com/qingyuanLee/fastapi-mcp-demo)。

## 总体架构：单进程、单端口，MCP 是"挂"上去的

先建立总体认知。整套方案只有一个进程、一个端口、一个 Uvicorn，业务 API 和 MCP 服务共存：

```
┌──────────────────────── Uvicorn 单进程 ────────────────────────┐
│                  FastAPI 主应用                                 │
│  ASGI 中间件层（按注册顺序从外到内）                              │
│  TokenRefresh → Authentication → Access Log → CORS → Limiter  │
│  (业务路由)      (业务路由)      (全部)    (全部)  (业务路由)     │
│                                                                 │
│  ┌───────────────┐        ┌────────────────────────────────┐  │
│  │ 存量业务路由    │        │ MCP 子应用 (/mcp)              │  │
│  │ /api/v1/**    │        │ FastMCP 内置认证                │  │
│  │ (meta/realtime│        │ BusinessTokenVerifier          │  │
│  │  /offline)    │        │ · 复用 JWT 解码逻辑             │  │
│  │               │        │ · verify_exp: False            │  │
│  └───────────────┘        │ MCP 工具 (tools/list, call)    │  │
│                           └────────────────────────────────┘  │
│  内部调用：FastMCP → ASGITransport → 存量业务 FastAPI 应用      │
│  权限透传：MCP 签发 60s 内部 API JWT，业务接口独立校验           │
└────────────────────────────────────────────────────────────────┘
```

三个要点：

1. **存量业务路由和 MCP 子应用是并排的兄弟**，都挂在主应用下，互不干扰。MCP 子应用通过 `app.mount("/mcp", mcp_app)` 最后挂载。
2. **ASGI 中间件层在最外面**，对业务路由和 MCP 一视同仁地过一遍——但认证职责不同（下一节细讲）。
3. **内部调用不出进程**：MCP 工具调用业务接口时，FastMCP 通过 `ASGITransport` 进程内直连存量业务应用，不发真实 HTTP，数据链路完全复用。

## 认证职责划分：三条路径，三套职责

这是整个方案的**设计核心**。很多人以为"挂个 MCP 认证就行"，其实一条链路里有三种认证职责：

| 请求路径 | 认证执行者 | 过期校验 | 续期 |
|---|---|---|---|
| `/api/v1/**`（业务 API） | `AuthenticationMiddleware` + `JwtAuthMiddleware` | ✅ 校验 | ✅ `TokenRefreshMiddleware` 自动续期 |
| `/mcp`（MCP 服务） | FastMCP `BusinessTokenVerifier` | ❌ 跳过（verify_exp: False） | ❌ 不续期，客户端自管 |
| `/mcp` → 内部业务调用 | `InternalJWTAuth` 签发 60s 内部 JWT | 内部 JWT 自身有效期 | — |

![](/img/posts/fastapi-mcp-mount-auth.jpg)

（图：双轨 token——业务 JWT 管自己的过期与续期，MCP token 只管签名与用户有效性）

三条路径各干各的，但**共用同一套 JWT 密钥和解码逻辑**——这是"复用"而不是"新造"：

- **业务路径**：原有体系分毫不动。校验过期、自动续期、返回 `X-New-Access-Token`。
- **/mcp 路径**：ASGI 中间件直接放行，认证下沉到 MCP 协议层。`BusinessTokenVerifier` 复用 `JwtAuthMiddleware` 的解码和用户查询逻辑，唯一差异是 `verify_exp: False`。签名仍然严格校验，用户有效性仍然校验——**伪造 Token 和禁用用户照样进不来**。
- **内部调用路径**：MCP 工具要调业务接口时，**不透传客户端 Token**，而是用 `get_access_token()` 拿到会话用户，签发一张 60 秒的内部 API JWT（独立 issuer/audience）。业务接口看到的是"内部调用"，自己再校验一遍。

这个"双轨 + 内部 JWT"设计解决了一个隐蔽问题：MCP 的 token 生命周期由客户端管，业务 token 由服务端管——**两条轨道的时钟和策略完全不同**，强行共用一套会互相打架。

## 完整请求链路：一条请求的十步

把上面的设计串起来，AI Agent 调一次工具，完整走这十步：

1. AI 应用连接 `/mcp`，携带 `Authorization: Bearer <JWT>`
2. `TokenRefreshMiddleware` 检测到 `/mcp` 路径，**直接放行**（不续期）
3. `AuthenticationMiddleware` 检测到 `/mcp` 路径，**直接放行**（认证下沉）
4. Access Log、CORS、Limiter 按各自范围执行
5. 请求路由到 MCP 子应用
6. `BusinessTokenVerifier` 校验 JWT：签名 + 用户有效性，**跳过过期**
7. 校验通过，FastMCP 将用户身份注入 MCP 上下文
8. MCP 工具通过 ASGITransport 调存量业务接口，签发 60 秒内部 JWT
9. 业务接口独立校验内部 JWT，按原有权限、业务规则、数据链路返回结果
10. FastMCP 将结果按 MCP 统一格式返回 AI Agent

链路的关键在 6–9：**认证发生在 MCP 协议层（6），权限落在业务层（9）**——MCP 只负责"你是谁"，业务接口负责"你能干什么"。这正是"复用现有平台资产"的落点。

## 关键设计决策：为什么这么设计

### 为什么单进程不拆服务

拆一个独立 MCP 服务要面对数据链路、配置、部署、权限四套复制问题。单进程挂载让 MCP 工具调用业务接口走 `ASGITransport` 进程内直连——**业务逻辑、权限、数据链路零改动**。代价是同一个进程要兼容两套协议，中间件必须谨慎。

### 为什么 /mcp 跳过过期校验

MCP 客户端（AI 应用）自己管理 token 生命周期，服务端掺和续期会打破它的状态模型。所以服务端只做两件事：**严格校验签名、严格校验用户有效性**，过期与否交给客户端。这是个明确的取舍：泄露的过期 token 在用户被禁用前一直有效，缓解靠业务侧续期 + 客户端自管生命周期。

### 为什么内部调用不透传客户端 token

客户端 Bearer Token 是"你"，内部 JWT 是"你授权的服务替你去调"。透传客户端 token 会让业务接口误以为请求来自客户端，而且 token 一旦泄露就能直达业务层。内部 JWT 60 秒过期、独立 audience，**token 绑定 /mcp，不中继下游**——业务层永远只信任内部签发的短期凭证。

### 为什么工具用白名单模式

存量接口几十个，不可能全暴露给 AI。方案是显式白名单：路由打 `tags=["mcp_public"]`，MCP 侧 `mcp.enable(tags={"mcp_public"}, only=True)` 先禁用全部、再启用带标签的组件。AI 看到的工具清单，就是业务方画过圈的接口。

## 落地与验证：10 项全过

总体设计定完，落到真实环境（fastmcp 4.1.0 / fastapi 0.138.2 / httpx 0.28.1）逐项验证：

| # | 验证项 | 结果 |
|---|---|---|
| 1 | BusinessTokenVerifier 实例化（required_scopes 由 super().__init__ 初始化） | ✅ |
| 2 | RemoteAuthProvider（authorization_servers + base_url）构建 | ✅ |
| 3 | /health 公开访问 | ✅ 200 |
| 4 | /mcp/ 无 token | ✅ 401 |
| 5 | initialize 带业务 JWT | ✅ 200 |
| 6 | tools/list 只暴露 3 个 mcp_public 工具 | ✅ |
| 7 | tools/call 全链路（内部 JWT 携带 user_id） | ✅ 200 |
| 8 | 业务 API 续期头 X-New-Access-Token | ✅ |
| 9 | 过期 token：业务 API 401、/mcp 200（跳过 exp） | ✅ 机制差异 |
| 10 | 错误签名 token：/mcp 401（签名仍严格校验） | ✅ |

第 9、10 项恰好验证了设计里的"双轨"：**MCP 跳过过期校验但绝不放过错签名**。

## 踩过的 8 个坑

设计归设计，落地时每个环节都有对应的坑（完整代码见公开库，这里只列现象与解法）：

| 坑 | 现象 | 解决 |
|---|---|---|
| fastmcp 4.x 依赖 httpx2 | `import httpx` 不兼容 | 一律 `import httpx2`，由 fastmcp 自动带入 |
| TokenVerifier 不调 super().__init__ | `no attribute 'required_scopes'` / `'scopes_supported'` | 子类 __init__ 必须 super().__init__()（父类在实例上赋值） |
| from_fastapi 传 transport | TypeError | 内部已设 ASGITransport，不传 |
| from_fastapi 的 tags 当过滤用 | 工具列表全量暴露 | `mcp.enable(tags={"mcp_public"}, only=True)` 显式 allowlist |
| 默认 operation_id 长工具名 | `query_x_api_v1_x_post` | `generate_unique_id_function` 返回 `route.name` |
| 内部调用取 request 头业务 token | `_AuthenticationError(401)` | `get_access_token()` 取会话用户签内部 JWT |
| BaseHTTPMiddleware 用于 MCP | tools/call 报 cancel scope / BrokenResourceError | 全部中间件改纯 ASGI（SSE 长连接禁缓冲） |
| 日志中间件编造 disconnect | 客户端 SSE stream ended | 读 body 缓存 + 原样转发，绝不改变数据流 |

其中最后两个坑（纯 ASGI 铁律、日志只读旁观）我在另一篇里展开过，这里只想强调一句：**坑再多，只要总体设计里的职责划分是对的，每个坑都是局部修补，不会推翻架构。**

## 上生产后才补上的一课：nginx 多后端，会话必须粘住

本地单进程全绿，一旦在前面加了 nginx 反代、后端扩成多台，客户端调 MCP 工具就**偶尔**报 404。错误长这样：

```
Error POSTing to endpoint (HTTP 404):
{"jsonrpc":"2.0","id":null,"error":{"code":-32600,"message":"Session not found"}}
```

服务端日志更有意思：同一个会话 ID 下，`GET /mcp/` 一直 200，`POST tools/call` 偶尔 404。

### 根因：会话在内存，不在请求里

streamable-http 的 `Mcp-Session-Id` 是 **uvicorn 进程内存态**：

1. `initialize` 建会话时落在后端 A 的内存
2. GET（SSE 长连接）TCP 连接一直挂在 A → 所以 GET 永远 200
3. POST `tools/call` 是新 HTTP 请求，nginx upstream 轮询负载均衡，**偶尔被分到后端 B** → B 的内存里根本没这个会话 → 返回 404 `Session not found`

"偶尔"不是玄学，就是轮询命中率。我在验证项目里用两个独立实例复现了一遍：

| 路由方式（6 次 tools/call，会话建在 8001） | 结果 |
|---|---|
| 轮询（模拟 nginx upstream round-robin） | 打 8001 ✅200 / 打 8002 ❌404，约一半失败 |
| 按 `Mcp-Session-Id` 粘性 | 6/6 全 ✅200，始终命中建会话的后端 |

### 修复：nginx 按会话头做一致性哈希

核心一行：`hash $http_mcp_session_id$remote_addr;`——同一会话永远落同一台后端。initialize 时还没有 session id（为空），拼 `$remote_addr` 兜底，同客户端首包与后续请求也能对齐：

```nginx
upstream ffs_test {
    hash $http_mcp_session_id$remote_addr;   # ★ 会话粘性
    server backend1:8000;
    server backend2:8000;
    keepalive 16;
}

location /mcp/ {                          # ★ location 带尾斜杠，避免 /mcp 触发 307
    proxy_pass http://ffs_test;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_set_header Host $host;                  # ★ 别注释！否则上游拿到 host=ffs_test
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_buffering off;                   # SSE 必须关
    proxy_cache off;
    proxy_read_timeout 300s;               # ★ SSE 长连接，防默认 60s 断流
    proxy_send_timeout 300s;
}
```

顺带三个小坑，都是这次一起发现的：

| 坑 | 现象 | 解法 |
|---|---|---|
| upstream 轮询打散会话 | 偶尔 404 `Session not found` | `hash $http_mcp_session_id$remote_addr` 粘性 |
| `proxy_set_header Host $host` 被注释 | 上游日志里 host=ffs_test，重定向/URL 构造异常 | 取消注释 |
| `location /mcp` 无尾斜杠 | 客户端打 `/mcp` 时 307 一跳 | `location /mcp/` 带斜杠 |
| 缺 `proxy_read_timeout` | SSE 长连接 60s 被 nginx 切断 | 配 300s |

一个补充说明：如果 upstream 只有一台后端但 uvicorn 是 `--workers >1`，会话在各 worker 内存间同样不共享——nginx 层帮不上，要么 `--workers 1`，要么把会话外置。等客户端升级到无状态协议（2026-07-28）后每个请求自含上下文，粘性才不再是硬需求。

## 写在最后

回头看，这套方案最值钱的是那个"职责划分"——不是技术多难，而是想清楚了：

- **认证职责下沉**：ASGI 中间件放行 /mcp，认证交给 MCP 协议层——两边各管一段
- **token 双轨**：业务 JWT 管生命周期，MCP token 管身份有效性，内部调用用短期 JWT 隔离
- **单进程挂载**：业务逻辑、权限、数据链路零改动，MCP 只是"挂"上去的

**先把总体设计讲清楚，坑才有意义。** 否则看十个坑的解法，也不知道它们为什么是坑。

完整代码、README 和三个验证脚本（smoke / e2e_call / diff_check）都在 [github.com/qingyuanLee/fastapi-mcp-demo](https://github.com/qingyuanLee/fastapi-mcp-demo)，`uv run uvicorn app.main:app --port 8000` 直接跑。

---

> 架构图：ProcessOn [高清预览图](https://po-img.processon.com/chart_image/diss/file/full/img?imgId=6ad1db71360ffb2fde5282aa&from=po_tool_ai_dissfile) ｜ [在线可编辑版本](https://www.processon.com/thirds/dissociate/6ad1db71360ffb2fde5282aa?category=flow&partner=poToolAIFlow&poInfo=MqnsPSY4j0Kp2WZeeXdincG_AdWQ_TDcQHS0EcMKLL0LcsMwu-yQVw2TILHNWWorgpGhwisWfOniq191DxfbpUw_lbFCA6iBQAlD_3FJeQ15cQvLJaS-wqla5TcmsJRwzoCuoAI80R_oz8MuFkElhvoUAE-NIAa2V-z7gmQ6jno&partnerFlag=c94842ec551fd95a9119b766b03f6cce)
