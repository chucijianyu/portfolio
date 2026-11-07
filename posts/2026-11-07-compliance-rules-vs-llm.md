# 规则拿否决权，模型给分数：研报合规审核里谁在查什么

研报流水线的 Reviewer 节点，注释写的是「结合规则引擎和 LLM 双重审核」。双重是真的：先请模型审一遍，再把规则命中的问题拼进同一个 issues 数组。但拼进去之后两边的话语权完全不同。

```python
# app/services/agent_service.py:1379-1392
rule_issues = self._rule_based_compliance_check(state.get("draft_report", ""))
review_data["issues"].extend(rule_issues)

score = review_data.get("score", 70)
if rule_issues:
    score = max(score - len(rule_issues) * 5, 0)

if score >= 80 and len(review_data["issues"]) == 0:
    review_result = "approved"
else:
    review_result = "revision_needed"
```

`extend` 在判定之前。规则命中一条，issues 就不为空；而 approved 要求 issues 为空。所以规则不是「扣分项」，是「一票否决」——模型给 95 分也没用，那 5 分只是白送。分数只影响一行日志和落库的 `score` 字段。

反过来，模型是唯一能给分的：`score` 直接取模型返回的 JSON 字段，没有类型转换也没有区间裁剪，模型返回 85 就是 85，返回字符串 "85" 就交给数据库的 Float 列去接。

## 怎么发现的：把三份清单抄出来对齐

要判断「谁在查什么」，光读一个函数不够，得把模型清单、规则函数、法条实现三份并排放。

- `REVIEWER_PROMPT`（`llm_service.py:610`）：7 条
- `_rule_based_compliance_check`（`agent_service.py:1674`）：4 条，返回中文字符串，不是结构化对象
- `ComplianceService.check_compliance`（`services/compliance_service.py`）：10 条，R001–R010，带 severity 权重（critical 20 / major 10 / minor 5），从 100 分往下扣，输出 `rule_id`/`severity`/`location`/`suggestion`

对齐之后缺口露出来：

| 检查项 | 模型清单 | 规则函数 | ComplianceService |
|---|---|---|---|
| 收益承诺 R001 | 有 | 有 | 有 |
| 绝对化表述 R002 | 有 | 有 | 有 |
| 风险提示 R003 | 有 | 有 | 有 |
| 数据来源 R004 | 有 | 无 | 有 |
| 评级术语 R005 | 有 | 无 | 有 |
| 未验证数据 R006 | 有 | 无 | 有 |
| 分析师信息 R007 | 无 | 无 | 有 |
| 免责声明 R008 | 有 | 有 | 有 |
| 买卖建议未提示风险 R009 | 无 | 无 | 有 |
| 内幕信息 R010 | 无 | 无 | 有 |

第三列全是「有」，但它没在跑。`ComplianceService` 在整个 backend 的调用点是 0：`app/services/__init__.py` 导出了它，`tests/test_compliance.py` 里 6 个用例调它，业务代码一处没有。同一个命运的还有 `COMPLIANCE_REVIEW` 那份 10 项提示词（`llm_service.py:424`），调用点 0。

于是第 7、9、10 项——分析师姓名与执业编号、个股买卖建议未提示风险、内幕信息——在真正执行的链路上，模型清单没有，规则函数没有，唯一实现它们的那套服务没接线。其中 R010 是 critical 级别。

## 分工该按什么切

修法不是「让模型多查三项」。模型查不查得到，取决于提示词那一句话，你没法保证；能保证的是把清单做成一份数据，按能力分工：

**能穷举的交给规则。** 词表命中、章节是否存在、字段是否齐全、正则能不能捞出无来源数值——这类判定结果是确定的，命中即否决，不看模型心情。

**只能判语气的交给模型。** 风险提示是否「充分」、结论和数据是否对得上、措辞有没有暗示收益。这类只能产出「问题描述 + 理由」，不该给否决权——因为一旦它说不，你得能解释为什么。

**清单只维护一份。** 规则函数的 `rule_id` 和模型清单的检查项共用同一组 id，启动时对一遍，缺哪项当场暴露，而不是等到评审会上被人问「第 10 项谁查的」。

具体到免责声明，现在的写法是 `if "免责声明" not in content`。而写研报用的 `WRITER_PROMPT`（`llm_service.py:587`）7 个小节里没有免责声明这一节；带「6. 末尾包含免责声明」的是 `REPORT_WRITING`（同文件 394），它调用点也是 0。所以第一轮必然被规则打回，第二轮回填 feedback 后靠模型自觉补上——能不能过，取决于模型听不听话。

风险提示那条更松：`if "风险提示" not in content and "风险" not in content`。用的是 `and`，全文任何一处出现「风险」两个字就通过，写「无风险」也算。

## 数完之后的几个数字

规则函数 4 条，模型清单 7 项，法条实现 10 项；实际覆盖 7 项，无人负责 3 项；`ComplianceService` 调用点 0，`COMPLIANCE_REVIEW` 调用点 0，`REPORT_WRITING` 调用点 0；`test_compliance.py` 六个用例全部测的是那个没人调用的服务——所以测试全绿，缺口全在。

还有一个耦合必须知道：`_review_router` 在 `revision_needed` 且 `revision_count` 达到上限时直接 `forced_end`，而 `forced_end` 在大屏上是被算进违规数的。规则太严 → 反复打回 → 强制结束 → 看板记一笔违规。规则的正确性会直接变成指标的正确性，这也是我不敢把「幻觉率」当独立结论说的原因之一。

可搬走的一句：规则和模型的分工不是「谁更准」，是「谁有权说不」。给模型否决权，你很难解释；给规则否决权，你必须保证每一条都能被正确满足——包括写作模板里得先有那一节。

明天能做的动作只有一个：把 10 项的 `rule_id` 抄成一张表，逐项标「规则 / 模型 / 无人负责」。只要第三列不是空的，先补接线，再谈调优。
