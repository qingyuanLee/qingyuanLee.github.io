---
layout: post
title: "把存量 FastAPI 挂进 MCP，我踩了 8 个坑才跑通"
subtitle: "fastmcp 4.1.0 从 from_fastapi 到纯 ASGI 中间件：零重构接入 MCP 服务的完整实战"
date: 2026-10-16 14:00:00 +0800
author: 清源
header-img: img/posts/fastapi-mcp-mount-cover.jpg
catalog: true
tags: [FastAPI, MCP, fastmcp, Python, 工程化]
---

## 引子：业务方一句话，老服务不能动

![](/img/posts/fastapi-mcp-mount-cover.jpg)

事情是这样的：业务方想用 MCP 客户端（Claude、Cursor 这类）直接调我们特征服务的接口，要求**服务必须暴露成 MCP 协议**。但我们的服务是个跑了好几年的存量 FastAPI：协程范式、业务 JWT 认证、日志中间件、lifespan 里挂着一堆原系统业务，动哪一块都心惊胆战。

需求翻译成人话就是：**在不重构、不换认证、不接管 lifespan 的前提下，把 MCP 作为"挂载服务"接进来，工具只暴露该暴露的那几个接口。**

我用 fastmcp 4.1.0 走通了一条路，全程踩了 8 个坑。完整可运行代码在公开库：[github.com/qingyuanLee/fastapi-mcp-demo](https://github.com/qingyuanLee/fastapi-mcp-demo)，这篇是它背后的完整思路。

## 先想清楚三件事（都是认知纠偏）

动手之前，有三个"你以为"必须纠正：

1. **MCP 的 token 和业务 token 不是一回事。** 业务 JWT 要校验过期时间（exp）并自动续期；但 MCP 的 token 由客户端自管生命周期——服务端**跳过 exp 校验、不续期**，只严格校验签名和用户有效性。两条轨，互不干扰。
2. **MCP 不能接管主应用的 lifespan。** 老项目 lifespan 逻辑复杂，指望 `combined_lifespan` 把它缝进去不现实。正解是"惰性启动"：第一个 `/mcp` 请求进来时，幂等进入 MCP 子应用的 lifespan，之后保持到进程结束。
3. **中间件必须全部纯 ASGI，禁用 BaseHTTPMiddleware。** 这条铁律排在最前——MCP 的 streamable-http 是 SSE 长连接，`BaseHTTPMiddleware` 会缓冲响应 body，和 SSE 天然冲突。这条坑最后再说，但它决定了整个中间件链的写法。

## 依赖与版本：caret 规范 + 一个隐藏的 httpx2

依赖就三件，版本声明和本地项目规范一致，用 `^`（uv 的 PEP 621 里写成上下限）：

| 依赖 | 声明（caret 等价） | 实际解析 |
|---|---|---|
| fastmcp | `>=4.1.0,<5.0.0`（^4.1.0） | 4.1.0 |
| fastapi | `>=0.138.2,<0.139.0`（^0.138.2） | 0.138.2 |
| httpx | `>=0.28.1,<0.29.0`（^0.28.1） | 0.28.1 |

```toml
[project]
dependencies = [
    "fastmcp>=4.1.0,<5.0.0",
    "fastapi>=0.138.2,<0.139.0",
    "httpx>=0.28.1,<0.29.0",
]
```

**坑 1（隐藏依赖）**：fastmcp 4.x 内部依赖的是 **httpx2**——httpx 2.x 的新包名，实测 2.13.1。它由 fastmcp 自动带入，你**不需要显式声明**，但写代码时必须用 `import httpx2`，不是 `import httpx`。`ASGITransport`、`InternalJWTAuth` 这些一律走 httpx2。第一次在这儿卡住的人不少。

## 认证：RemoteAuthProvider + 自定义 TokenVerifier

![](/img/posts/fastapi-mcp-mount-auth.jpg)

（图：双轨 token——业务 JWT 管自己的过期与续期，MCP token 只管签名与用户有效性）

认证是这套方案的核心。MCP 侧的配置长这样：

```python
auth = RemoteAuthProvider(
    token_verifier=BusinessTokenVerifier(),
    authorization_servers=settings.mcp_authorization_servers,  # 必填
    base_url=settings.mcp_resource_server_url,                 # 必填（4.x 替代 resource_server_url）
)
```

这里有两个认知纠偏：

**`authorization_servers` 和 `base_url` 跟服务器 IP 没关系。** 它们只服务 OAuth 元数据和 audience 描述，实际认证走的是自定义 `TokenVerifier`。服务器启动时根本不用知道自己 IP——配置里填部署方给的公网域名就行。这俩参数在 4.x 都是必填、没有默认值。

**坑 2（必须 super().__init__）**：`BusinessTokenVerifier` 继承 `TokenVerifier`，如果你子类的 `__init__` 不调 `super().__init__(...)`，运行期必然报 `'BusinessTokenVerifier' object has no attribute 'required_scopes'`——补上 `required_scopes` 后还会缺 `scopes_supported`，一样是这行解决的。因为 `required_scopes` / `base_url` / `resource_base_url` 这三个属性是父类 `AuthProvider.__init__` 在**实例上赋值**的，子类不调父类构造器，实例上就永远没有这些属性。

```python
class BusinessTokenVerifier(TokenVerifier):
    def __init__(self, base_url=None, required_scopes=None, resource_base_url=None):
        super().__init__(                     # ★ fastmcp 4.1.0 必备
            base_url=base_url,
            required_scopes=required_scopes,
            resource_base_url=resource_base_url,
        )

    async def verify_token(self, token: str) -> AccessToken | None:
        payload = jwt.decode(
            token, settings.jwt_secret, algorithms=[settings.jwt_algorithm],
            options={"verify_exp": False, "verify_aud": False, "verify_iss": False},
        )  # 跳过 exp，但签名仍严格校验
        ...
        return AccessToken(token=token, client_id=str(user_id),
                           scopes=payload.get("scopes", []),
                           expires_at=None, claims=payload)
```

**坑 3（from_fastapi 没有 auth 形参）**：fastmcp 4.1.0 的 `FastMCP.from_fastapi` 签名里没有显式 `auth`，但它的 `**settings` 会透传给 `FastMCP.__init__`——实测 `from_fastapi(..., auth=auth)` 是生效的（`mcp.auth == auth`）。同时**千万别传 `transport`**：`from_fastapi` 内部已经设了 `transport=httpx2.ASGITransport(app=app)`，你再传就 TypeError。

## 工具暴露：两个名字相关的坑

MCP 挂上之后，第一反应是"所有接口都变成工具了"——这就是坑 4。

**坑 4（tags 不是过滤器）**：fastmcp 4.x 的 `tags` 参数（旧版叫 `include_tags`）语义是"给全部工具附加标签"，是 union 不是过滤。你给路由打了 `mcp_public` 标签，工具列表照样全量暴露。要过滤必须显式 allowlist：

```python
mcp.enable(tags={"mcp_public"}, only=True)   # ★ 先禁用全部，再启用带标签的组件
```

标签在 FastAPI 路由上声明：`router = APIRouter(tags=["mcp_public"])`。

**坑 5（工具名又长又丑）**：FastAPI 默认的 operation_id 是 `{函数名}_{路径}_{method}`，比如 `query_naming_demo_api_v1_naming_demo_post`，OpenAPIProvider 直接拿它当工具名。解决：

```python
def _short_generate_unique_id(route):
    return route.name   # APIRoute.name 默认 = 端点函数名

business_app = FastAPI(
    title="...",
    generate_unique_id_function=_short_generate_unique_id,  # ★ 0.138.x 参数名
)
```

注意：FastAPI 0.138.x 的参数名是 `generate_unique_id_function`，旧版叫 `generate_unique_id`，传错会被静默忽略，工具名还是老样子。

## 内部调用链路：async_auth_flow + get_access_token

MCP 工具内部要调业务接口，业务接口得认出"这是 MCP 内部调用"。方案是签发一个 60 秒的内部 JWT。

**坑 6（内部调用没有业务 token）**：老项目最初的做法是 `request.headers.get('Authorization')` 取业务 token——但 MCP 工具内部调业务接口时，请求头里**根本没有业务 token**，直接报 `_AuthenticationError(401, 'request need token authentic')`。必须换成从 fastmcp 的会话上下文取用户：`get_access_token()` 拿到会话用户 `sub`，再签内部 JWT。

```python
class InternalJWTAuth(httpx2.Auth):
    async def async_auth_flow(self, request):   # ★ 协程范式 → 必须 async_auth_flow
        access_token = get_access_token()       # fastmcp 会话 token
        user_id = (access_token.claims.get("sub") if access_token else None) or "system"
        internal_token = jwt.encode(
            {"sub": user_id, "iss": "demo-feature-mcp", "aud": "demo-feature-api",
             "exp": now + timedelta(seconds=settings.internal_jwt_ttl_seconds)},
            settings.internal_jwt_secret, algorithm=settings.jwt_algorithm,
        )
        request.headers["Authorization"] = f"Bearer {internal_token}"
        yield request
```

原项目是协程范式，所以这里必须是 `async_auth_flow`，不是同步的 `auth_flow`——这也是当时排查了很久的一个点。

## 中间件铁律：纯 ASGI + 读 body 缓存转发

![](/img/posts/fastapi-mcp-mount-middleware.jpg)

（图：日志中间件是旁观者——读取数据流，但绝不改变它）

这是最后一个大坑，也是耗时最长的。

**坑 7（BaseHTTPMiddleware 与 SSE 冲突）**：老项目日志中间件继承 `BaseHTTPMiddleware`，挂上 MCP 后现象很诡异：initialize 正常，tools/call 报 `ReadError` + `RuntimeError: Attempted to exit a cancel scope`。根因是 `BaseHTTPMiddleware` 缓冲响应 body，跟 SSE 无限流冲突。整个中间件链必须全部改纯 ASGI 实现（类名、日志字段、注册方式不变）：

```
CORSMiddleware（expose Mcp-Session-Id / WWW-Authenticate / X-New-Access-Token）
→ AuthenticationMiddleware(JwtAuthMiddleware，/mcp 放行)
→ McpLifespanMiddleware（首个 /mcp 请求幂等启动）
→ TokenRefreshMiddleware（跳过 /mcp 不续期）
→ AccessLogMiddleware → RateLimitMiddleware → McpAccessLogMiddleware
→ 业务路由 → mount /mcp
```

**坑 8（读 body 不能吞、更不能编造 disconnect）**：MCP 的所有请求 URL 都是 `/mcp/`，普通访问日志只能看到"调用了 mcp"，看不到具体工具。工具名藏在 JSON-RPC body 里。日志中间件读 body 解析时，**必须"缓存 + 原样转发"**：

```python
async def receive_wrapper() -> Message:
    message = await receive()
    if message["type"] == "http.request":
        body_chunks.append(message.get("body", b""))   # 缓存
    return message                                     # ★ 原样转发，绝不吞/绝不编造
```

如果你吞了消息、或者编造 `http.disconnect`，下游读 body 后再次 receive() 会**误判客户端断开**，SSE 响应被中断——客户端报 `SSE stream ended without a response`，服务器端报 `ASGI callable returned without completing response`。这个坑我们一度误判成"官方客户端兼容差异"，最后定位到是自己中间件的 bug。

修好之后日志长这样，一眼能看到调了哪个工具、谁调的、哪个会话：

```
[mcp-log] 请求IP:127.0.0.1, 用户:u_demo_001, MCP方法:tools/call,
工具:query_realtime_feature, 会话:xxx, 响应code:200, 响应时间:0.012s
```

## 端到端验证：10 项全过

真实环境（fastmcp 4.1.0 / fastapi 0.138.2 / httpx 0.28.1）跑下来的验证表：

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

第 9、10 项最能说明双轨设计：**MCP 跳过过期校验但绝不放过错签名**——客户端自管 token 生命周期，服务端只管身份有效性。

## 踩过的坑清单

| 坑 | 现象 | 解决 |
|---|---|---|
| fastmcp 4.x 依赖 httpx2 | `import httpx` 报错/不兼容 | 一律 `import httpx2`，由 fastmcp 自动带入 |
| TokenVerifier 不调 super().__init__ | `no attribute 'required_scopes'` / `'scopes_supported'` | 子类 __init__ 必须 super().__init__() |
| from_fastapi 传 transport | TypeError | 内部已设 ASGITransport，不传 |
| from_fastapi 的 tags 当过滤用 | 工具列表全量暴露 | `mcp.enable(tags={"mcp_public"}, only=True)` |
| 默认 operation_id 长工具名 | `query_x_api_v1_x_post` | `generate_unique_id_function` 返回 `route.name` |
| 内部调用取 request 头业务 token | `_AuthenticationError(401)` | `get_access_token()` 取会话用户签内部 JWT |
| BaseHTTPMiddleware 用于 MCP | tools/call 报 cancel scope | 全部中间件改纯 ASGI |
| 日志中间件编造 disconnect | 客户端 SSE stream ended | 读 body 缓存 + 原样转发 |

## 写在最后

回头看，这套方案的本质是把"接入 MCP"这件事的**侵入面压到最小**：不重构业务、不让出 lifespan、不换认证体系，只加了一层"挂载 + 过滤 + 内部转发"。

- **双轨 token**：业务 JWT 管自己的生命周期，MCP token 管自己的，互不掺和
- **惰性 lifespan**：主应用动都不动，第一个请求来了再启动
- **纯 ASGI 铁律**：中间件全部避开 BaseHTTPMiddleware，SSE 才能活

最值钱的经验其实是那个"缓存 + 原样转发"——**日志中间件只是旁观者，绝不能改变数据流**。这句同样适用于所有要给 SSE 长连接加日志/监控的场景。

完整代码、README 和三个验证脚本（smoke / e2e_call / diff_check）都在 [github.com/qingyuanLee/fastapi-mcp-demo](https://github.com/qingyuanLee/fastapi-mcp-demo)，直接 `uv run uvicorn app.main:app --port 8000` 就能跑。

---

> 架构图：ProcessOn [高清预览图](https://po-img.processon.com/chart_image/diss/file/full/img?imgId=6ad1db71360ffb2fde5282aa&from=po_tool_ai_dissfile) ｜ [在线可编辑版本](https://www.processon.com/thirds/dissociate/6ad1db71360ffb2fde5282aa?category=flow&partner=poToolAIFlow&poInfo=MqnsPSY4j0Kp2WZeeXdincG_AdWQ_TDcQHS0EcMKLL0LcsMwu-yQVw2TILHNWWorgpGhwisWfOniq191DxfbpUw_lbFCA6iBQAlD_3FJeQ15cQvLJaS-wqla5TcmsJRwzoCuoAI80R_oz8MuFkElhvoUAE-NIAa2V-z7gmQ6jno&partnerFlag=c94842ec551fd95a9119b766b03f6cce)
