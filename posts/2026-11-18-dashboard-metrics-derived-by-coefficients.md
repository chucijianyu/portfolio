# 16 张指标卡，7 张是量出来的，6 张是乘出来的

监控看板有 4 组 16 张指标卡。每次刷新都有数，除了「Rerank 提升率」始终显示 `--`，其余全部有值，看起来检索质量、生成质量、Agent 效能、系统性能四个维度都在被度量。

真正让我停下来的是「审核一次通过率」和「工具调用成功率」这两张卡：它们的曲线形状完全一样，只是纵向压扁了一点。我开始逐张反查每个数字的来源。

## 接口返回 25 条，前端渲染 16 张

`GET /api/v1/monitor/metrics` 的返回其实有 25 条：rag 6 条、generation 6 条、agent 6 条、system 7 条。而 `Dashboard.jsx` 的 `defaultEvaluationMetrics` 里只有 16 张卡，四个分组各 4 张。

多出来的 9 条是：`ndcg@10`、`hybrid_improvement`、`confidence`、`satisfaction`、`error_rate`、`e2e_duration`、`ttft`、`throughput`、`concurrent_users`。后端每天都算，前端从来不显示。

真正落在卡上的 16 个数字，我按「能不能指到一条 SQL 或一个采集点」分了类。

## 一张卡背后可能是同一次乘法

`monitor.py` 里 25 条指标是同一个字典字面量里写死的，逐个看表达式就能看出血缘：

```python
{"name": "recall@10",   "value": _r(avg_confidence)},
{"name": "precision@5", "value": _r(avg_confidence * 0.94)},
{"name": "mrr",         "value": _r(avg_confidence * 0.89)},
{"name": "ndcg@10",     "value": _r(avg_confidence * 0.93)},
{"name": "rerank_improvement", "value": None},

{"name": "accuracy",        "value": _r(avg_confidence)},
{"name": "confidence",      "value": _r(avg_confidence)},
{"name": "source_coverage", "value": _r(retrieval_coverage)},
{"name": "hybrid_improvement", "value": _r(retrieval_coverage)},

{"name": "first_pass_rate", "value": _r(tool_success_rate * 0.95)},
{"name": "avg_revisions",   "value": _r(1.5 - tool_success_rate * 0.2, 2)},

{"name": "p99_search_latency", "value": _r(p95_latency * 40, 0)},
{"name": "gpu_utilization",    "value": _r(min(0.95, max(0.3, throughput / 60)), 2)},
{"name": "ttft",               "value": _r(p95_latency * 0.2, 2)},
```

`avg_confidence` 是 `ChatHistory.confidence` 的均值，而这个字段来自 `rag_service._self_evaluate()`——模型对自己回答的评分。也就是说「Recall@10」和「回答准确率」这两张卡，量的是同一个东西：模型自评置信度。它们的达标线却不同（85% 和 90%），于是同一个 0.72 会让一张卡红、一张卡绿。

`p99_search_latency = p95_latency * 40`，P95 的单位是秒，乘 40 之后当毫秒显示。P99 的目标是 200ms，200 / 40 = 5s，正好等于 P95 的目标值。所以这两张卡的达标线是同一根：P95 一旦超过 5 秒，两张一起红；没超过，两张一起绿。中间不存在 P99 单独告警的可能。

`gpu_utilization` 更直接：`min(0.95, max(0.3, throughput / 60))`。全 backend 搜 `gpu`，只有这一行计算和上面那行目标值定义，没有任何 nvml、nvidia-smi 或显存采集。它是被夹在 30% 到 95% 之间的吞吐量，无论机器上有没有 GPU。

还有一个更隐蔽的：`concurrent_users` 取的是 `max(1, total_agents)`，而 `total_agents` 是 `AgentExecution` 表的累计行数。它不是并发用户，是历史上跑过的 Agent 执行次数，只增不减。目标 200——跑满 200 条执行记录之后，这张卡就永远达标，且数字只会越来越大。

## 名字是唯一的身份证明

后端给 25 条、前端要 16 张卡，中间靠一张 `METRIC_NAME_MAP` 按名字对齐：`'系统可用性': 'availability'`、`'回答准确率': 'accuracy'`。`Dashboard.jsx` 里留着一条注释，记录这个映射是怎么来的——「V4.9 修复：增加中英文 name 映射，按 name 优先匹配，避免索引错位导致『系统可用性』显示 4200%」。

4200% 这个数不可能来自任何一次比例计算。渲染函数是 `if (m.unit === '%') return (val * 100).toFixed(...) + '%'`，能打出 4200% 的原始值只能是 42——那是别的指标串到了可用性这张卡上。指标的身份靠一串中文字符串维持，加减一条指标就会让整套卡片错位。

## 数出来的结果

16 张卡拆开是：7 张对应一次真实查询（幻觉率、来源覆盖率、合规率、单 Agent 平均耗时、工具调用成功率、P95 延迟、系统可用性），2 张共用 `avg_confidence`，6 张由系数派生（Precision@5、MRR、审核一次通过率、平均修改次数、P99 延迟、GPU 利用率），1 张恒为 `None`。

这 16 个数字架在 10 个原始量上：置信度均值、检索覆盖率、`forced_end / total_reviews`、`approved / total_reviews`、`execution_time` 均值、`completed / total_agents`、`failed / total_agents`、P95、吞吐量、`1 - 5xx 率`。前四个里有三个的分母是同一张 `ComplianceReview` 表。

还有两处算术决定了卡片永远不会报警。`avg_revisions = 1.5 - tool_success_rate * 0.2`，取值范围是 [1.3, 1.5]，而它的目标是「≤1.5、越低越好」——这张卡在数学上不可能不达标。`satisfaction = 4.0 + avg_confidence * 0.5`，取值范围 [4.0, 4.5]，目标是 ≥4.0，同样不可能不达标（这张卡前端没渲染，所以危害停留在接口层）。两处的系数都把「零点」放在了目标值上：置信度取 0，满意度正好等于 4.0。

## 一条判断规则

把指标按来源分三档写进返回结构，比改公式更要紧：`measured` 能指到一条 SQL 或一个采集点，`derived` 声明它由哪个字段乘系数得来，`placeholder` 就是恒 `None`。

派生不是罪——有些量本来就没法直接测。真正的问题是派生值和测量值长得一样：同样的卡片、同样的字号、同样的绿色对勾，看的人无法区分「查出来的」和「推出来的」。我已经开始在每个指标对象里加 `source` 字段，`derived` 的卡片在右下角标一个「推算」，`placeholder` 直接不渲染卡片，而不是画一个永远的 `--`。
