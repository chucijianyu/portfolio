# 三次重试一次都没轮到 5xx：LLM 调用容错的三处漏口

研报解读接口偶尔会在几百毫秒内返回「⚠️ AI推理服务暂时不可用」。但 `llm_service.py` 顶部的版本注释写得很明确：引入指数退避重试机制，应对 Ollama 瞬时故障；仅对网络瞬时故障重试。按配置算，三次尝试加超时最坏要等六分钟。秒回说明它根本没进重试分支。

## 一、白名单少一个类型，等于没有重试

`LLMService` 把可重试异常列成一个类属性：

```python
_RETRYABLE_EXCEPTIONS = (httpx.ConnectError, httpx.ReadTimeout, httpx.ConnectTimeout)

try:
    response = await client.post(f"{self.base_url}/api/chat", json=payload)
    response.raise_for_status()
    ...
except self._RETRYABLE_EXCEPTIONS as e:
    # 退避后重试
except Exception as e:
    logger.error(f"LLM 推理异常（不重试）: {e}")
    return self._fallback_response(messages[-1].get("content", ""))
```

`raise_for_status()` 抛的是 `httpx.HTTPStatusError`，MRO 是 `HTTPStatusError → HTTPError → Exception`，和上面那三类没有任何继承关系。httpx 0.28.1 下实测：

```python
>>> issubclass(httpx.HTTPStatusError,
...           (httpx.ConnectError, httpx.ReadTimeout, httpx.ConnectTimeout))
False
```

于是只要服务端选择用状态码表达失败（而不是把连接断掉），第一轮就落到 `except Exception`，直接降级，一次都不重试。而「服务端活着、但暂时给不出结果」恰恰是最该重试的情形——连接层反而是最不容易出问题的那一层。

## 二、重试只写在了一个入口上

统共三个对外方法，重试只在 `_invoke_with_retry` 里。`invoke()` 走它，`stream_invoke()` 和 `invoke_sync()` 各自另抄了一份 httpx 调用，都没有重试循环，也没有退避。

调用点统计：`invoke` 7 处（`agent_service.py` 3 处、`rag_service.py` 4 处）；`stream_invoke` 1 处，就是 `api/reports.py:250` 给前端逐字推流的 SSE 接口；`invoke_sync` 2 处（`rag/retriever.py:827`、`analysis_service.py:82`）。10 个调用点，7 个有保护，3 个裸奔，其中一个还是用户唯一能实时看见的那个。

## 三、`LLM_RETRY_MAX_BACKOFF` 是死配置

退避公式是 `min(initial_backoff * 2 ** (attempt - 1), max_backoff)`。配置默认 `LLM_RETRY_MAX_ATTEMPTS = 3`、`INITIAL_BACKOFF = 0.5`、`MAX_BACKOFF = 5.0`，能算出来的只有 attempt=1 的 0.5 秒和 attempt=2 的 1.0 秒，`5.0` 永远比不过。真正卡住耗时的是 `LLM_TIMEOUT = 120`：最坏 `3 × 120 + 1.5 = 361.5` 秒，六分钟。退避在总量里只占 1.5 秒，调它是无效劳动。

另外三处小漏口。流式分支没调 `raise_for_status()`，只 `json.loads` 每一行并取 `message.content`，状态码不对时会静默产出空串。流的 `except` 分支也是 `yield self._fallback_response(prompt)`，所以一旦在吐了半段之后断流，前端会先收到半截回答、再收到一条「AI推理服务暂时不可用」——拼接在一起就是一段既不像回答也不像报错的东西。`_last_token_count` 只在 `_invoke_with_retry` 里赋值，`invoke_sync` 走完全程不更新它，从那条路进来的调用在统计里是零 token。还有 `check_health()`：它只 `GET /api/tags` 判 200，服务进程在就等于健康，不验证能不能生成——拿它当重试开关同样是错判。

## 四、降级文案在说谎

`_fallback_response` 的第一行是「以下为检索到的相关信息摘要」，但函数体只有一句 `prompt[:200]`——把用户的原始问题原样回显，没有任何检索内容。前端看到的是「服务不可用 + 你刚才问的那句话」，而不是摘要。文案写了实现没做的事，比不写更糟：它让人以为降级链路是通的。

`grep -rn "retry\|backoff\|max_attempts" backend/tests/` 命中 **0**。这套重试逻辑一次都没被测过，所以白名单少一个异常类型两年没人发现。

## 五、修法

核心只有一句：让「可重试」由状态码决定，而不是由异常类型碰运气。

```python
class _TransientHTTP(Exception):
    def __init__(self, status: int):
        self.status = status

_RETRYABLE_STATUS = {429, 500, 502, 503, 504}
_RETRYABLE_EXCEPTIONS = (httpx.ConnectError, httpx.ReadTimeout,
                         httpx.ConnectTimeout, _TransientHTTP)

response = await client.post(url, json=payload)
if response.status_code in _RETRYABLE_STATUS:
    raise _TransientHTTP(response.status_code)
response.raise_for_status()
```

配套四件事：把重试循环抽出来，让三个入口共用；`LLM_TIMEOUT` 按「总预算」而不是「单次」来配，否则重试等于把用户挂在六分钟里；`_fallback_response` 要么真的拼检索片段，要么改成「服务不可用，请稍后重试」；补一条 `monkeypatch` 掉 `client.post` 让它返回 503 的单测，断言调用次数是 3 而不是 1。

可搬走的一句：重试策略不是写在注释里的承诺，是写在 `except` 类型里的白名单；白名单错一个类型，重试次数配多少都是 1。
