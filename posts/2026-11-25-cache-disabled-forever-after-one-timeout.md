# 一次 3 秒读超时之后，缓存对整个进程关闭了

## 现象：数字停住，接口照常 200

知识库首页那张状态卡挂了 60 秒缓存。有一段时间，重建索引结束后它的「已完成文档数」还停在旧值，但接口一直是 200，日志里除了偶尔一行 `Redis GET失败` 没有别的异常。

第一反应是「缓存没失效」，可失效逻辑确实写了、也确实在被调用。往下翻一层才发现问题不在失效：那次读超时之后，这个进程的缓存就再也没启用过。停住的数字不是缓存的旧值，是缓存已经不存在了，而它退场时只留下一行 warning。

## 三个数：1 处定义，1 处使用，2 处失效

整个 backend 里 `@cached` 只有一次真实使用：

```python
@router.get("/status", summary="知识库状态查询")
@cached(prefix='cache:kb:status', ttl=settings.CACHE_KB_STATUS_TTL, include_role=False)
async def knowledge_status(
    request: Request,
    current_user: User = Depends(require_role("admin", "analyst")),
    db: Session = Depends(get_db)
):
```

`grep -rn "@cached"` 共 3 处命中，另两处一处是装饰器自身的定义，一处是它 docstring 里的用法示例。`CACHE_KB_STATUS_TTL` 在 `config.py:148` 是 60。

`invalidate_cache` 有 2 个调用点：`api/knowledge.py:68` 在重建索引之后，`api/reports.py:86` 在上传后处理的 `finally` 里。后者写法是对的；前者在 `await rag_service.rebuild_index(db)` 之后，一旦重建抛异常就跳过失效，只剩 TTL 兜底。

装饰器模块的 docstring 声明了两条设计原则，逐条对账：

- **「缓存键包含角色维度，避免越权读取」**——`_build_cache_key` 确实会拼 `role=`，但唯一的使用点把 `include_role` 传成了 `False`。`/knowledge/status` 返回的是四组计数加模型名与维度，不含用户数据，关掉不构成越权；只是这条原则在实际链路上生效次数为 0。
- **「缓存命中时标记 X-Cache: HIT」**——实现是 `request.state.cache_hit = True`，注释写「通过 request.state 传递，路由层可读取」。全仓 grep `cache_hit`：写入 2 处（`rate_limit.py:233`、`:238`），读取 0 处，也没有任何中间件把它变成响应头。缓存命中没有，对外、对日志、对指标都不可见。

## 根因：降级开关是单向的

`RedisClient` 的设计原则是「所有方法捕获异常，返回 None/False，不向上抛」。调用侧确实省事，但它漏掉了一个状态：`_available`。

`get()` 的异常分支是这么写的：

```python
except Exception as e:
    logger.warning(f"Redis GET失败 key={key}: {e}")
    self._available = False  # 标记不可用，触发后续降级
    return None
```

`set` / `delete` / `exists` / `expire` 的 `except` 里也是同一句赋值。而把 `_available` 置回 `True` 的只有两个地方：`connect()`（:55）与 `ping()`（:80）。

运行期还有谁会调这两个？`grep -rn "redis_client\."` 之后，`connect`/`ping` 只有三处：`main.py:58` 的 lifespan 启动、`main.py:271` 的 `/health`、`main.py:307` 的 `/readyz`。

关键在部署侧。`docker-compose.yml` 给 backend 配的健康检查打的是 `/livez`：

```yaml
healthcheck:
  test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/livez', timeout=5)"]
```

而 `/livez` 的 docstring 第一句就是「仅检查进程是否运行，不检查依赖组件」，函数体只有 `return {"status": "alive", "version": settings.APP_VERSION}`。它不碰 Redis，自然不会调 `ping()`。

于是这条链是闭合的：客户端 `socket_timeout=3`，一次卡到 3 秒的读超时 → `_available = False` → 之后每个 `get()` 在入口处 `if not self.is_available: return None` 直接返回 → 缓存既不命中也不写入 → 而这个状态在进程剩下的生命周期里没有任何东西会复位，除非有人手工访问一次 `/health` 或 `/readyz`。

服务表面上完全正常：请求照常 200，只是每次都真去跑四次数库计数、再真去 `RAGService().get_vector_count()` 取一遍向量库。这正是「数字停住」的来源——缓存已经不在了，而它还挡在前面假装在挡。

## 修法

1. **降级要带恢复时钟。** 给 `RedisClient` 加一个 `_retry_after` 时间戳：进入不可用态后，`get()` 先看时间，超过 N 秒就试一次 `ping()`，成功即复位。少了这一步，任何 `_available = False` 都是单向闸刀。
2. **探活要打带依赖的端点。** `/livez` 是给 K8s 判断「是否重启容器」用的，拿它当 compose 的 healthcheck，等于主动放弃依赖探测；要么改成 `/readyz`，要么清楚自己在放弃什么。
3. **命中率必须能看见。** `cache_hit` 只写不读等于没有。接进中间件的响应头和指标卡，这次的「缓存悄悄关掉」第一小时就会露头。
4. **失效放进 `finally`。** 上传端点已经这么写了，重建端点还差这一步。

## 可搬走的一条

韧性组件的风险不在「降级」本身，而在降级状态没有出口。每一行 `_available = False` 的下面，都该回答「什么时候再试」；如果答案是「等下次重启」，那它就不叫降级，叫关停。
