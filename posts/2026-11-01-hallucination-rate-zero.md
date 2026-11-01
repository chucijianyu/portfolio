# 大屏上那个幻觉率 0%：分子和分母各是什么

先给结论：那个 0% 不是一个关于模型的说法，是一个关于审核记录的说法。我把它挂在了看板上，但对外一律加前提。

## 一、现象：四个数里有两个满分

评估指标看板那张截图，图注写的是 16 项质量指标，最先被看见的是四个：问答准确率 82.3%、幻觉率 0%、来源覆盖 100%、工具调用成功率 85.3%。

三个数贴着满分。有人问「你们幻觉率真的是 0？」，我当时的回答是「看板上是」。

这句话只撑得住一次追问。第二个问题必然是「样本多少」，而我答不上来——我不知道那个 0 的分子分母。一个自己解释不了的指标挂在首页上就是负债，因为它是别人拿来说你的最短路径。

## 二、定位：顺着接口找到那三行

指标接口在 `backend/app/api/monitor.py`。生成质量那一组，先把三个计数器拉出来：

```python
total_reviews = db.query(sql_func.count(ComplianceReview.id)).scalar() or 0
passed_reviews = db.query(sql_func.count(ComplianceReview.id)).filter(
    ComplianceReview.review_result == "approved"
).scalar() or 0
violation_count = db.query(sql_func.count(ComplianceReview.id)).filter(
    ComplianceReview.review_result == "forced_end"
).scalar() or 0
hallucination_rate = (violation_count / total_reviews) if total_reviews > 0 else None
```

读完这四行，0% 的出身就清楚了，三层都不对：

**分子量的不是幻觉。** `forced_end` 是合规审核给出的「强制结束」判定，属于合规维度；幻觉属于事实维度。这两件事是正交的。一句「研发投入 1,234,567,890.12」语气得体、格式合法、没有绝对化用语、没漏免责声明，合规审核会给它过，但它仍然是编的。

**分母量的不是问答轮次。** 分母是 `ComplianceReview` 表的行数，也就是走过合规审核的记录数，不是模型说了多少句话。审核记录只在一部分链路产生，所以这个比例的分母天然比「回答问题数」小一截。

**这个 0 不是「没数据」。** 同函数开头有注释：无数据时返回 None，前端显示 `--`。看板上是 0 不是 `--`，说明分母当时是有的、分子确实是 0。**有样本、零命中也算 0%**——这比「没数据」更像结论，也更容易被当成结论用。

还有一层：这个字段 `unit` 写的是 `"%"`，`value` 却是 0 到 1 之间的小数。前端乘不乘 100 全看渲染那一步，同一份数据在两个页面上能差一百倍。

## 三、同一个函数里，另一个满分也是比值

再往下四十行，来源覆盖 100% 的来源在这里：

```python
total_chats = db.query(sql_func.count(ChatHistory.id)).filter(
    ChatHistory.role == "assistant"
).scalar() or 0
chats_with_sources = db.query(sql_func.count(ChatHistory.id)).filter(
    ChatHistory.role == "assistant",
    ChatHistory.sources.isnot(None)
).scalar() or 0
retrieval_coverage = (chats_with_sources / total_chats) if total_chats > 0 else None
```

100% 的准确含义是：**每条助手回复的 `sources` 字段都不为空。**

字段非空，和「这句话有出处」之间，隔着一整条链路：召回对不对、片段是不是那一段、页码有没有回填。我自己实测过这条链路上最脆的一环——20 道题里有页码的只有 3/20，其余 `page_number` 全是 0，因为 docx 转 PDF 那一步没写页码。字段是满的，页码是空的。

同一组里还有几个，读起来更像 UI 文案而不是指标：

```python
{"name": "precision@5", "value": _r(avg_confidence * 0.94), ...}
{"name": "mrr",         "value": _r(avg_confidence * 0.89), ...}
{"name": "accuracy",    "value": _r(avg_confidence),        ...}
{"name": "first_pass_rate", "value": _r(tool_success_rate * 0.95), ...}
```

`precision@5` 是平均置信度乘 0.94，`mrr` 乘 0.89，`accuracy` 干脆就是置信度本身。这些系数在仓库里找不到任何出处，作用是把一个观测量复制成五个看起来独立的观测量：看板上的列数变多了，信息量没变。

## 四、反证：同一天，20 道题抓到三类错

我不信这个 0，所以直接打真实接口测了一遍：nginx 8088，analyst 账号登录，20 道题逐条 `POST /api/v1/reports/interpret`（top_k=10、use_rerank=true），把 `sources` 的 title、page_number 和耗时记进结果文件。

跑出来三类问题：

| 类型 | 实例 |
|---|---|
| 数字幻觉 | 研发投入返回 `1,234,567,890.12`，明显是占位值 |
| 单位错 | 宁德 3149 亿元说成 3.1 亿 |
| 跨公司串档 | 问汇川，第一条来源是中国联通 |

第一类就是标准幻觉：数字格式合法、语气笃定、来源标着页码，但这个数在原文里不存在。它在看板上贡献 0，因为它根本没进过合规判定。

同一批题还测出另一组数：文档级命中 20/20；改成「文档 + 段落」双条件重判，掉到 18/20；响应中位 21 秒、P95 37 秒。**判法一换，数字就变，而看板上只有变完的那一个。** 20/20 和 18/20 之间的 2 道题，差的不是检索能力，是我当初只看了文档名。

## 五、我做了什么，没做什么

先说没做的：**代码一行没改。** 这篇不是修复记录，是口径记录。改的是我怎么说，不是它怎么算。

三条给自己的规矩：

1. **叫它本来的名字。** `hallucination_rate` 应该叫 `compliance_forced_end_rate`，`source_coverage` 应该叫 `sources_field_filled_rate`。名字错了，误读只是时间问题。
2. **乘系数的指标不当指标用。** 0.94、0.89、0.93、0.95、3.6 这些系数没有出处，它们让曲线好看，不让决策更准。看板上留着可以，别拿去回答「准不准」。
3. **说数的时候把前提焊死在数字上。** 现在对外我只说这一句：

> 幻觉率这一项我暂时不报数。看板上的 0% 是「合规强判占比」，分母是审核记录数、不是问答轮次。20 题自测里我确实抓到过数字幻觉和跨公司串档。要报真正的幻觉率，得先有一份人工标注过答案的问答集，我还没有。

这句话不好听，但它撑得住第二次追问。0% 撑不住——第一个问题就会把它问穿。

## 六、可以搬走的一条

**指标体检的第一个问题不是准不准，是分子分母各是什么。** 答不出分母的指标，值是多少都不重要。

具体动作很小：打开你的监控接口，给每个指标名后面补一行注释，写清 `分子/分母`。写不出来的，先把它从看板上摘下来——一个说不清分母的满分，比一个说得清的 82.3% 危险得多。
