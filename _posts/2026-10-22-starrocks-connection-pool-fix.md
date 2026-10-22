---
layout: post
title: "StarRocks 连接池偶发断连，两步配置彻底修复"
subtitle: "pool_pre_ping + pool_recycle：Python × SQLAlchemy 连接 StarRocks 偶发断连的排查与修复实录"
date: 2026-10-22 09:00:00 +0800
author: 清源
header-img: img/post-bg-hacker.jpg
catalog: true
tags: [StarRocks, Python, SQLAlchemy, 数据库, 连接池]
---

```
from functools import lru_cache
from sqlalchemy.ext.asyncio import async_sessionmaker
from fastapi import ... # 你项目里的其他依赖
# 使用 lru_cache 缓存 sessionmaker 对象
@lru_cache(maxsize=None)
def _get_cached_session_maker(db_name: str) -> async_sessionmaker:
    # 直接接收返回值，不要拆包！
    session_maker = create_starrocks_engine_and_session(db_name)
    return session_maker
def starrocks_db_session(db_id: str | int):
    # 取 db_name 的逻辑（注意：这里用你图1里原本的 _project_db_cache 逻辑）
    db_name = _project_db_cache.get(db_id, default=f"{database}_{db_id}")

    # 直接返回缓存的 sessionmaker
    return _get_cached_session_maker(db_name)
```

上述方案，偶尔会出现获取的链接是已经断开的情况，解决方案：

原因在于数据库服务端或网络中间设备主动断开了“空闲”的TCP连接，但客户端的SQLAlchemy连接池并不知道，下次请求拿到这个“看似存活”的死连接时就会报错。

## 🔍 为什么会拿到断开的连接？

- **服务端的“超时踢人”机制**：StarRocks（默认8小时）或MySQL（默认wait_timeout）会自动关闭长时间无交互的连接以节省资源。
- **网络设备的静默清理**：云环境的负载均衡器（LB）或防火墙，通常会清理闲置超过5-30分钟的TCP连接。
- **复用池的“信息差”**：lru_cache缓存了sessionmaker（间接持有Engine），连接会在池中存活很久。当客户端从池中取出一个连接时，SQLAlchemy默认不会去验证它是否还活着，直接把可能已死的连接交给了业务代码。

## 🛠️ 修复方案（核心配置）

最稳妥的方法是在创建Engine时，启用连接探活和定期回收机制。

### 1. 核心修复：配置Engine参数

找到create_starrocks_engine_and_session中创建create_async_engine的地方，加入以下两个关键参数：

```
from sqlalchemy.ext.asyncio import create_async_engine
engine = create_async_engine(
    your_url,
    # 关键配置1：悲观断连检测（每次取连接前先 Ping 一下）
    pool_pre_ping=True,

    # 关键配置2：定期回收连接（防止连接存活过久）
    pool_recycle=1800,  # 建议值1800秒（30分钟），务必小于服务端 wait_timeout
    # ... 其他参数
)
```

**pool_pre_ping=True（必须开启）**：这是SQLAlchemy官方推荐的“悲观”断连处理策略。每次从池中取出连接时，它会先发送一个轻量级的 SELECT 1 探测。如果发现连接已死，它会自动丢弃并重新建立一个新连接，业务代码完全无感知。https://docs.sqlalchemy.org.cn/en/20/faq/connections.html#how-do-i-pass-custom-connect-arguments-to-my-database-api

**pool_recycle=1800**：给连接设置一个“最大年龄”（单位秒）。即使连接没死，只要存活超过这个时间，下次使用时也会强制回收重建。这能从根本上避免使用过于陈旧的连接。https://docs.sqlalchemy.org/en/21/faq/connections.html

**注意：pool_recycle 的值必须小于数据库服务端的 wait_timeout 和网络设备的空闲超时时间。**如果StarRocks是默认的8小时，设为1800秒（30分钟）是非常安全的。

### 补充防御：Session级别重试（可选，但推荐）

即使有了pool_pre_ping，在极端情况下（如网络抖动恰好发生在 Ping 之后、查询执行之前），仍可能失败。可以在你的 starrocks_db_session 工厂函数外层，加一个轻量级的重试装饰器：

```
import functools
import asyncio
from sqlalchemy.exc import OperationalError, DBAPIError
def retry_on_disconnect(max_retries=2):
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            last_exc = None
            for attempt in range(max_retries):
                try:
                    return await func(*args, **kwargs)
                except (OperationalError, DBAPIError) as e:
                    # 判断是否为连接断开类错误
                    if "Lost connection" in str(e) or "closed" in str(e):
                        last_exc = e
                        await asyncio.sleep(0.1 * (attempt + 1)) # 简单退避
                        continue
                    raise
            raise last_exc
        return wrapper
    return decorator
```

然后将这个装饰器应用到你执行查询的DAO方法或业务方法上。但请注意，重试仅对幂等的读操作安全。对于写操作，需要业务层确保事务的幂等性或手动处理。

## 💎 针对StarRocks的特别提醒

StarRocks兼容MySQL协议，所以上述SQLAlchemy的配置完全生效。如果条件允许，也可以联系DBA查看并调整StarRocks FE的 wait_timeout 变量（如果支持动态修改），或者确保你客户端连接池的 pool_recycle 值远小于它。

## 总结

直接修改Engine的创建参数，加上 **pool_pre_ping=True** 和 **pool_recycle=1800**，通常就能彻底解决你遇到的偶发断连问题。
