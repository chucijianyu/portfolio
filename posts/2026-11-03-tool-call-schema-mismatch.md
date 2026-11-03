# 工具调用全部成功，因为失败从不抛异常

看板上有个指标叫 `tool_call_success_rate`，target 0.98。它一直很漂亮，漂亮到我不敢拿它当成绩讲。查完之后发现两件事：这个数跟工具调用没什么关系，以及它漂亮的原因是——失败从来不抛异常。

## 一、分子分母都不是工具

`backend/app/api/monitor.py` 里这一行就是全部实现：

```python
total_agents = db.query(sql_func.count(AgentExecution.id)).scalar() or 0
success_agents = db.query(sql_func.count(AgentExecution.id)).filter(
    AgentExecution.status == "completed"
).scalar() or 0
tool_success_rate = (success_agents / total_agents) if total_agents > 0 else None
```

分子是「状态 completed 的 AgentExecution 条数」，分母是「AgentExecution 总数」。一次研报生成里调了 6 个工具、其中 3 个返回错数据，只要这个 Agent 最后跑完，6 次调用就全部算成功。指标名里的 tool，只活在变量名里。

同一个字典里还有两行：`first_pass_rate = tool_success_rate * 0.95`，`avg_revisions = 1.5 - tool_success_rate * 0.2`。乘系数出来的数不是统计出来的数，它只会跟着上面那个比值一起动。

## 二、schema 写了，但没人读

`ToolService.TOOL_SCHEMAS` 规规矩矩定义了 6 个工具的 JSON Schema。我在整个 backend 目录里搜 `get_tool_schemas`，只有定义和 return 两行，没有任何调用点。模型看到的参数说明，和真正接参数的函数，是两套东西，中间没有校验。

于是有了这组签名：

```python
# tool_service.py：schema 里叫 dimensions，类型是 array
"risk_assessment": {
    "parameters": {"type": "object", "properties": {
        "sec_code": {"type": "string"},
        "dimensions": {"type": "array", "items": {"type": "string"}},
    }, "required": ["sec_code"]}
}

# mcp/server.py：收到 list，然后位置传参
async def risk_assessment(sec_code: str, dimensions: Optional[List[str]] = None) -> str:
    result = await _tool_service.risk_assessment(
        sec_code,
        dimensions or ["market", "financial", "operational", "liquidity"]
    )

# tool_service.py：第二个位置参数是 str，不是 list
async def risk_assessment(self, sec_code: str, dimension: str = "comprehensive") -> Dict:
    ...
    result["dimension"] = dimension
```

一个 list 顺着位置参数落进 `dimension: str`，Python 不做运行时类型检查，所以不报错。后果是维度选择完全失效，而返回体里的 `dimension` 字段会把整个列表原样回显——下游只看「字段有没有值」，看到的是一个非空列表，比 `"comprehensive"` 看起来还详细。

`industry_comparison` 是同一个坑的另一半：

```python
async def industry_comparison(self, sec_code: str, dimension: str = "valuation") -> Dict:
    dimension_data = {"valuation": {...}, "profitability": {...}, "growth": {...}}
    return {"comparison": dimension_data.get(dimension, dimension_data["valuation"])}
```

MCP 层的默认值写的是 `metric: str = "ROE"`。`dimension_data` 的键只有 valuation / profitability / growth，`"ROE"` 查不到，`.get` 的兜底把它悄悄换成 valuation。所以「按 ROE 做行业对比」这个调用，永远返回 PE 分位。不抛异常，不打日志，返回 200。

## 三、不抛异常就算成功

MCP 包装器只在 except 分支里记失败：

```python
try:
    result = await func(*args, **kwargs)
    _metrics.record(tool_name, True, elapsed_ms)
    return result
except Exception as e:
    _metrics.record(tool_name, False, elapsed_ms)
```

上面两处都是「参数不对但语法合法」，走的是第一个分支。成功率的分子因此永远包含这些静默错值。这不是指标不够准，是记账口径把「没崩」等同于「算对了」。

## 四、failover 那句 raise 是死的

`stock_price_query` 声明了三源冗余，循环里 `break` 在第一次成功，后面有 `if data is None: raise RuntimeError("所有数据源均不可用")`。但 `_fetch_price_from_tushare` 内部自己 `except Exception` 之后 `return self._generate_mock_price(...)`，另外两个源直接 `return self._generate_mock_price(...)`。三个函数没有一个会抛，循环永远在第一个源 break，那句 raise 一次也执行不到。「多源冗余」实际是「单源 + 两层兜底」，兜底回来的是模拟数据。

## 五、我改的四件事

1. **把 schema 变成唯一真源。** 起服务时用 `inspect.signature` 对照 `TOOL_SCHEMAS` 的键名跑一遍，`dimensions` 对不上 `dimension` 就直接启动失败。这类错该在进程起来那一秒炸，不该在报表被质疑的那一天炸。
2. **工具返回统一信封。** `{"ok": False, "error_type": "bad_args", "detail": ...}`，包装器按 `ok` 记账，不再按「有没有抛异常」记账。
3. **单测按 schema 的键名调用。** 现在 `test_risk_assessment` 写的是 `risk_assessment("600519.SH", "comprehensive")`，位置参数绕过了参数名，所以两个测试都是绿的。改成 `dimensions=["market"]` 会立刻红，红得对。
4. **降级必须留痕。** `stock_price_query` 已经在返回体里带 `data_source` 和 `_is_real`，`news_retrieval` 和 `industry_comparison` 没有。模拟数据不带标记，下游就分不清哪句能引用。

可搬走的一句：**给工具调用做账，别拿「进程有没有崩」当口径。** 参数名对不上、兜底把参数吃了、mock 冒充真实数据，这三种错在 Python 里都不抛异常，也都长得像成功。
