# 五个限流配置，只有一个端点用上了

做安全自查的时候，我把"限流"这一栏先打了勾。因为登录接口确实有一行 `rate_limiter.enforce(...)`，注释写着"防止暴力破解"。真正逐行对下来，这个勾打不住。

## 现象：配置写了五档，只有一档有人读

`core/config.py` 里排得很整齐：

```python
RATE_LIMIT_LOGIN: int = 5        # 登录接口：5 次/分钟（防爆破）
RATE_LIMIT_GENERATE: int = 10    # 研报生成：10 次/分钟（LLM 推理昂贵）
RATE_LIMIT_INTERPRET: int = 30   # 研报解读：30 次/分钟
RATE_LIMIT_UPLOAD: int = 10      # 文件上传：10 次/分钟
RATE_LIMIT_DEFAULT: int = 120    # 默认：120 次/分钟
```

全仓 `grep RATE_LIMIT_`，命中六行：五个定义，加一处读取。`auth.py:45` 的 `limit=settings.RATE_LIMIT_LOGIN`。另外四档在整个 backend 和 frontend 里一次都没被读。

同理，`rate_limiter.enforce` 全仓只有一个调用点，就是 `auth.py:42` 的登录。注释里点名要保护的"LLM 推理昂贵"的研报生成、上传、解读，全部零限流。

于是最该挨打的地方反而敞着：`POST /auth/register` 是公开端点，既不要求登录，也没有任何限流，每调一次就往 `users` 表里插一行、算一次 `get_password_hash`。攻破登录要绕着 5 次/分钟的墙走，批量造账号这条路一步都不用绕。

## 定位：标识是请求方自己填的

带着"那这道墙到底能不能拦住"的疑问去看标识怎么来的：

```python
# middleware.py:304
def get_client_ip(request: Request) -> str:
    xff = request.headers.get('X-Forwarded-For')
    if xff:
        return xff.split(',')[0].strip()      # 取第一个
    xri = request.headers.get('X-Real-IP')
    if xri:
        return xri.strip()
    return request.client.host if request.client else '-'

# auth.py:42
client_ip = get_client_ip(http_request)
rate_limiter.enforce(..., identifier=f'login:{client_ip}', limit=settings.RATE_LIMIT_LOGIN)
```

限流的桶是按 `login:{client_ip}` 分的。而 `X-Forwarded-For` 的首元素，是最不可信的那一段。

`nginx/nginx.conf` 里每一处 API 反代都是 `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`——注意是 `proxy_add_`，语义是"在客户端传来的值后面追加 `$remote_addr`"，不是覆盖。所以客户端自己发一个 `X-Forwarded-For: 1.2.3.4`，nginx 转给后端的就是 `1.2.3.4, 真实IP`，而后端 `split(',')[0]` 正好取走了伪造的那个。

结果是每换一次请求头就换一个桶。5 次/分钟的限额，在伪造请求头面前等于没有上限。

同一份 nginx 配置里，`X-Real-IP` 是被 `$remote_addr` 覆盖写的，那才是可信的；可 `get_client_ip` 把它排在 `X-Forwarded-For` 后面当备胎。`docker-compose.yml` 里 backend 只 `expose: 8000`、不映射宿主端口，唯一入口是 nginx 的 `8088:80`——也就是说，唯一可信的来源恰好被优先级压到了最后。

## 另外三处小账

`RateLimiter.check()` 里有 `reset_at = int(now + window_seconds)`，`Retry-After` 就是拿它减当前时间。它算的是"从现在起再等一整个窗口"，而不是"窗口里最早那条记录什么时候过期"。最坏情况下对方只差一秒就能重试，我们让他等六十秒——这个数只会高估，不会低估。

`X-RateLimit-*` 系列响应头只在 429 分支里出现。正常响应一个都没有，客户端在挨打之前不知道自己还剩几次。而且 `X-RateLimit-Remaining` 那里硬编码了 `'0'`，`check()` 明明算了 `remaining` 却没传出来。

降级是放行：`if not redis_client.is_available: return True, limit, limit, ...`。Redis 一挂，限流整体消失，只留一行 warning。这条本来是合理的取舍，但配合"`_available` 置回 True 只有 `connect()`/`ping()` 两条路、而 healthcheck 打的是不查依赖的 `/livez`"，就变成了：一次读超时之后，这个进程余生都不再限流。

最后是测试。`tests/conftest.py:71` 直接把 `rate_limiter.enforce` 换成了 no-op，注释写明是为了避免三个 session 级 fixture 依次登录时踩到 5 次/分钟。全 `backend/tests/` 里搜 `429`，零命中。所以"限流生效"这件事，从来没有被任何一条用例证明过。

## 我会怎么改

1. 标识换成 `X-Real-IP` 优先，`X-Forwarded-For` 只取最右一段（也就是最近一跳代理写入的那个），并且在非代理直连场景下直接落 `request.client.host`。
2. `enforce` 挂到生成、上传、解读三个端点，用的是已经写好却没人读的那三个配置。
3. `/register` 补一条比登录更严的限流——它写库、算哈希，成本比一次失败的密码校验更高。
4. `reset_at` 改成查 ZSET 里最小 score 加窗口，`Retry-After` 才是真话；成功响应也带上 `X-RateLimit-Remaining`。
5. 至少一条用例断言第 6 次登录返回 429，并且要在 `conftest` 里只对业务 fixture 关限流，不是全局关。

## 一句可以搬走的

限流的三件事必须同时成立才算数：限额配了且有人读、标识不可伪造、生效有测试兜。缺一条，代码里那行 `enforce` 就只是注释的一部分。
