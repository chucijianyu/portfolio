# 上面七天，下面全历史：一张大屏上两个同名的数

## 现象

大屏左上是「使用趋势」，七根柱、六个指标，横轴是 `11-14` 到 `11-20`；右下是「任务分布」饼图，图例写着「研报解读」「研报生成」「财务分析」「知识图谱」。两块图挨着放，一共 3 秒刷新一次，谁看都以为它们在描述同一批数据。

我先注意到的是对不上：趋势图七天的「研报解读」加起来是个小几十，饼图上「研报解读」的占比却接近一半，切片面积换算回去是三位数。一个系统不可能七天解读二十次、累计解读三百次——除非它跑了好几年。

## 一、同名，不同窗口

两个接口都在 `app/api/monitor.py`：`/monitor/trend`（:473）和 `/monitor/task-distribution`（:570）。只差一百行，取数却是两套。

```python
# /trend :501 —— 每个 day 循环里执行一次
interpret_count = db.query(sql_func.count(ChatHistory.id)).filter(
    ChatHistory.role == 'user',
    ChatHistory.created_at >= day_start,
    ChatHistory.created_at <= day_end
).scalar() or 0

# /task-distribution :585 —— 全表，没有任何时间条件
interpret_count = db.query(sql_func.count(ChatHistory.id)).filter(
    ChatHistory.role == 'user'
).scalar() or 0
```

同一张表、同一个条件、同一个字段名 `interpret`，差别只在 `created_at` 的三个过滤条件。趋势图问「这七天各干了多少」，饼图问「自建库以来一共干了多少」。两张图并排，一个叫日增、一个叫总量，标签都写「研报解读」。

## 二、同名，不同表：Writer 行不是报告篇数

窗口还能靠眼睛分辨，换表就完全看不出来了。趋势图的「研报生成」数 `analysis_tasks` 里当天的 `generation` 行；饼图先把 `generation` 跳过，再用 `agent_executions` 里 `agent_name == "Writer"` 的行数顶上去：

```python
# /task-distribution :608
if atype in ("interpretation", "generation"):
    continue          # 跳过，避免重复

# :614 —— 换成另一张表
report_count = db.query(sql_func.count(AgentExecution.id)).filter(
    AgentExecution.agent_name == "Writer"
).scalar() or 0
```

问题是 Writer 的行数和报告篇数不是一个东西。`_writer_node`（`agent_service.py:1247`）一进节点就无条件插一条执行记录：

```python
execution = AgentExecution(
    task_id=state.get("task_id"),
    agent_name="Writer",
    status="running",
    input_data={"sec_code": state["sec_code"], "revision": revision_count}
)
db.add(execution)
db.commit()
```

而 Writer 会进不止一次。审核不过回退要重进（`_review_router` 在 `revision_count >= max_revisions` 前一直回退），人工审核点「要求修改」也重进（:963 分支）。一篇报告改两轮就是三条 Writer 行、饼图上 +3，而 `generation` 任务只有一行、趋势图 +1。更松的是 `status` 没过滤——LLM 报错时 `execution.status` 被置成 `failed`，那一行照样留在表里，照样被算成一次「研报生成」。

所以饼图那个切片量的是**撰写轮次**，趋势图那根柱量的是**报告篇数**。名字一样，量纲不同。

## 三、八个图例，两个能出来

`type_map`（:594）列了八种任务类型，中文名、颜色都配齐了。但 `AnalysisTask.task_type` 是 `Enum` 列，`models/database_models.py:77` 只允许四个值：`interpretation`、`generation`、`financial`、`knowledge_graph`。`trend_prediction`、`risk_assessment`、`clustering`、`comprehensive` 这四种在数据库层面根本写不进去，插入会被 ENUM 拒绝。剩下四个里 `interpretation` 和 `generation` 又被上面的 `continue` 跳过。

结论是：`for atype, acount in analysis_types` 这个循环，实际只会吐出 `financial` 和 `knowledge_graph` 两片。

那四种分析类型去哪了？在 `analysis_service.py:105`：

```python
task = AnalysisTask(
    task_type="financial",
    parameters={"analysis_type": analysis_type, "periods": periods},
    ...
)
```

趋势预测、风险预警、聚类、综合，全都以 `task_type="financial"` 落库，真实类型塞在 JSON 的 `parameters.analysis_type` 里。饼图按枚举列分组，于是四种分析在图上合并成一片「财务分析」——`type_map` 里那四个颜色永远不会出现，不是因为没数据，是因为它们不在被分组的那一列上。

## 结果

三处修正的方向，按代价从小到大：

1. **指标名带窗口。** `interpret` 拆成 `interpret_7d` 和 `interpret_all`，前端标签跟着改。同名不同源是最难发现的一类错误，因为它不报错。
2. **饼图切片必须同源。** 要么全部来自 `AnalysisTask.task_type`，要么给 `sub_type` 加一列，不能一半来自 `chat_history`、一半来自 `agent_executions`。
3. **计篇数和计轮次分开。** 「研报生成」用 `AnalysisTask` 数篇，「撰写轮次」用 `AgentExecution` 数轮，两个数都有用，但不能共用一个标签。

另外记一笔成本：`/trend` 一次请求是 7 天 × 6 指标 = 42 条 `COUNT`，每个都套了 `try/except` 兜 0；前端 3 秒一轮（`Dashboard.jsx:200`），一分钟 20 次，就是 840 条 `COUNT`。这条查询打在没有按 `created_at` 建索引的 `chat_history` 上，越往后越慢——而它每 3 秒重算一次七天的历史。

## 一句能搬走的

大屏上并排放的两个数，只要标签一样，观众就会默认它们是同一个数。所以指标的口径不该写在文档里，该写进字段名：窗口多大、来自哪张表、数的是篇还是轮，三件事缺一个，就是在邀请下一个人重新踩一遍。
