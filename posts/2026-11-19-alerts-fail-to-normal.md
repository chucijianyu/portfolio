# 三条告警规则，取不到数就显示「正常」

监控页上有一块告警面板，三张卡：RAG 延迟过高、LLM 错误率过高、合规违规检测。它们的「最近触发」那一栏写着同一句话——「从未触发」。

我一直以为这是好消息。直到我把三条规则的 `current_value` 逐条往源头追了一遍：三个值，三个来源，没有一个是标签上写的那个东西，而且取不到数据时它们会一起变成绿色。

## 一、同一个字段下面挂着三个不同的量

接口是 `GET /monitor/alerts`，规则定义在 `ALERT_RULES`（`api/monitor.py:65`），三条规则的 `current_value` 一开始都是 `None`，由端点实时算出来再回填。问题是回填的三个量根本不同源：

- `rag_latency_high`，条件是「P95 > 5s」，值来自 `metrics_collector.get_p95_latency_ms()`——这是中间件对**所有 HTTP 请求**的耗时统计，不分端点，不光 RAG。
- `llm_error_rate_high`，条件是「error rate > 5%」，值来自 `AgentExecution` 表的 `failed / total`。这是 Agent 执行失败率，不是 LLM 调用失败率；同一个算式在 `/metrics` 里叫 `agent_error_rate`，那张卡的目标值是 `0.02`。同一个数，两块屏，两把尺子：5% 以内这边算正常，2% 以上那边算不达标。
- `compliance_violation_detected`，条件是「violations > 0」，值是全表 `ComplianceReview` 里 `review_result == "forced_end"` 的条数，**没有时间窗口**。

第三条尤其值得看一眼。没有窗口意味着这是一个单调不减的计数：只要历史上出现过一条 `forced_end`，这块卡就永久变红，没有任何它能自己恢复的路径。而它看起来像实时告警，实际是「历史上有没有出过事」的墓碑。

第一条的取值还有一条三层回退链，`monitor.py:718-737`：中间件没有请求记录时，去 `MonitoringMetric` 表捞最近 5 分钟的 `latency` 行取分位；再没有，就拿 `AgentExecution` 的平均 `execution_time` **乘 0.8** 当作 P95。均值乘系数还是均值，它不是分位数，但它填在同一个 `current_value` 字段里，页面上的标签还是「P95」。

## 二、取不到值时，一律算「正常」

这是我最想改的地方。三条规则的判定分支长这样：

```python
for rule in ALERT_RULES:
    alert = dict(rule)
    if rule["name"] == "rag_latency_high":
        alert["current_value"] = round(rag_p95, 2) if rag_p95 is not None else None
        alert["status"] = "alerting" if (rag_p95 is not None and rag_p95 > rule["threshold"]) else "normal"
    elif rule["name"] == "llm_error_rate_high":
        alert["current_value"] = round(llm_error_rate, 3) if llm_error_rate is not None else None
        alert["status"] = "alerting" if (llm_error_rate is not None and llm_error_rate > rule["threshold"]) else "normal"
    elif rule["name"] == "compliance_violation_detected":
        alert["current_value"] = violation_count
        alert["status"] = "alerting" if violation_count > rule["threshold"] else "normal"
    if alert["last_triggered"] is None:
        alert["last_triggered"] = "从未触发"
    alerts.append(alert)
```

注意每一行的 `else "normal"`。`rag_p95` 取不到（`None`）、`llm_error_rate` 取不到（`None`），判定都落到 `normal`。更隐蔽的是第三条：它的采集块是 `try/except`，异常分支里写的是 `violation_count = 0`，于是数据库挂了也会算出「0 条违规」——绿灯。

也就是说，「系统健康」和「我读不到数据」在这块面板上是同一个颜色。这不是容错，这是把传感器断路接成了「正常」。前端也在同一条路上：`useState(alertRules)` 的初始值就是三条 `status: 'normal'` 的规则，`Promise.allSettled` 里只有在 `/alerts` 请求成功时才覆盖状态，失败就保持初始值——首屏渲染的那一下，和接口挂掉一整晚，画面完全一致。

## 三、「从未触发」是初始值，不是事实

全仓 grep `last_triggered`，只有两类命中：`ALERT_RULES` 里三个 `None`，和上面那两行把 `None` 改写成字符串「从未触发」。没有任何一行代码写入真实的触发时间。

所以这栏文案的真实含义是「这个字段一直是空的」，不是「从未触发过」。它和 `status` 的关系是：`status` 可能已经变红，`last_triggered` 还是那句「从未触发」，两行字自相矛盾地印在同一张卡上。

## 四、还有一个小样本陷阱

中间件那层的 P95 是这么算的（`core/middleware.py:112-123`）：

```python
durations = sorted(d for _, d in self._recent_requests)
p95_index = min(int(len(durations) * 0.95), len(durations) - 1)
return durations[p95_index]
```

样本 20 条时 `int(20 * 0.95) = 19`，取的是第 20 个，也就是**最大值**；10 条时 `int(9.5) = 9`，还是最大值。小样本下这个「P95」几乎恒等于最慢那一次，配上 5 秒阈值，一次慢请求就够把卡打红，下一次窗口里的普通请求又把它打回绿——阈值没动，颜色的含义变了。

而且这个函数自己不做时间裁剪，`popleft()` 只发生在 `get_performance_trend()` 里。也就是说告警用多大的窗口，取决于有没有人打开趋势图：开了，窗口是 30 分钟；没人开，窗口是 deque 里全部记录（`maxlen=50000`），进程活多久就是多久。

## 五、我打算怎么改

1. 取值函数返回 `(value, window, source)`，页面上把窗口和来源一起显示出来。看不到窗口的分位数不算分位数。
2. 状态加第三态：取不到值就是 `unknown`，灰色，不计入「全部正常」，也不计入「告警中」。默认值的方向决定你是被叫醒还是睡过头，所以默认必须是不出声的灰，而不是绿。
3. 违规数改成带时间窗口的查询，再配一个已处理标记；否则这块卡只能红一次，之后所有新事故都被淹没在同一个红灯里。
4. 第四条规则如果哪天加进 `ALERT_RULES` 而忘了加分支，`dict(rule)` 里的 `status` 会保持 `normal`、`current_value` 是 `None`——一个安静的、永远正常的告警。这条得在加规则的地方写死注释。

## 结果

现在数得清的东西：3 条规则、3 个不同源的取值、3 处 `else "normal"`、1 处 `except` 里把 `violation_count` 置 0、`last_triggered` 真实写入点 0 处、`backend/tests/` 里关于 `/alerts` 的 2 个用例都只断 `status_code`（200 和 403），对 `current_value` 的断言 0 条。

一句话带走：告警的健康状态必须是算出来的，不能是默认的；一个告警系统最危险的输出不是误报，是它读不到数据时还告诉你一切正常。
