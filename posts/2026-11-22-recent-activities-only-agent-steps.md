# 大屏上「最近活动」的最大视野是三篇研报

演示前我先自己点了一遍。上传一份 PDF，等索引进度跑完；去问答页问一句；删掉一份旧文档。回到大屏，右下角那张「最近活动」里还是那几条「研报撰写 · 执行完成」，一条新记录都没有。它每 3 秒刷新一次，刷得挺勤，只是刷的东西和我的操作无关。

## 它到底在查什么

先看查询本身。

```python
records = db.query(AgentExecution).order_by(
    AgentExecution.created_at.desc()
).limit(limit).all()
```

`limit` 参数是 `Query(8, ge=1, le=50)`，默认 8、上限 50；前端固定传 30。整段没有 join、没有 union、没有别的表，只有这一个 `AgentExecution`。

再数写入点。全仓 `AgentExecution(` 一共 5 处，全部在 `agent_service.py`，分别对应 data_collector / analyst / writer / reviewer / error_handler 五个节点。上传文档不写它、问答不写它、删除文档不写它、财务分析也不写它。也就是说，这张卡显示的不是「系统里发生了什么」，而是「研报流水线跑到第几步」。

按一篇研报数行数：DataCollector 一条、Analyst 一条、Writer 一条起、Reviewer 一条起。走完整条链路且首次审核通过，是 4 条。`MAX_REVISION_COUNT = 3`，每退一次，Writer 与 Reviewer 各多一条，最坏 1+1+4+4 = 10 条。

于是 30 这个上限的含义变了：30 ÷ 10 = 3。这张卡的最大视野是三篇研报。哪天有几百人同时在上传、问答、看研报，它依然只看得到最近三个研报任务。

前端每页显示 3 条，一共 10 页。3 条正好是一篇研报的三分之一——所以你翻到的每一页都不是「一段时间内发生了什么」，而是「某一次任务的前三步」。

还有一个放大项：`_writer_node` 里那次插入是无条件的，回退重写与人工改稿都会再插一行，`status` 也不做过筛。修改次数在这个列表里直接变成行数的乘数。这和饼图量的是「撰写轮次」而非「报告篇数」是同一个根因，只是那条链路走的是 `agent_name='Writer'` 的计数，这边走的是最近 N 行明细。

## 我怀疑错的一条

一开始我以为是时间算错了。`created_at` 是 `server_default=func.now()`，由库端生成；相对时间是应用端用 `datetime.now()` 减出来的，中间还 `.replace(tzinfo=None)` 把时区抹掉。库端时钟对应用端时钟，中间还主动丢掉了时区信息，这是最容易整段偏移的写法。

去 compose 查了一遍：mysql 和 backend 两处都写了 `TZ: Asia/Shanghai`。今天是对齐的，这条假设不成立，我没写进缺陷清单。但 `.replace(tzinfo=None)` 说明这种对齐靠的是两份环境变量恰好一致，不是代码里的约束——哪天只改一处，列表里所有时间会一起偏 8 小时，而且不会有任何报错。这类「靠部署配置维持的正确」值得记一笔，但不算 bug。

## 我会怎么改

改法还没落地，取决于这张卡要回答哪个问题。两条路：

A. 它要回答「流水线跑到哪了」——就按 `task_id` 折叠，一次任务一行，右侧显示 `3/6 步`。查询从「取最近 30 行明细」变成「取最近 N 个任务的聚合」：

```python
# 改法示意（未合并）
rows = (
    db.query(AgentExecution.task_id, func.count(AgentExecution.id))
    .group_by(AgentExecution.task_id)
    .order_by(func.max(AgentExecution.created_at).desc())
    .limit(limit)
    .all()
)
```

B. 它要回答「系统里发生了什么」——就得有一张业务事件表，由上传、删除、问答各自的端点各写一行。这些数据其实都在：`documents.created_at`、`documents.updated_at`、`chat_history.created_at`（还有 `user_id`）。但它们分散在三张表，没有任何一处汇进这张列表，也没有统一的 event 视图。

两条都不是前端能改的，这也是我把它留成待办、没有当天修掉的原因。

## 顺手记下的一处

后端每条活动的 dict 是 5 个键：`icon`/`text`/`time`/`color`/`route`，没有 `id`。所以前端渲染只能写 `key={i}`。列表每 3 秒整表替换一次——位置键配上会位移的数据，等于你正看着的那一行，下一次刷新后就成了另一条活动，而点一下会 `navigate` 到对应页面。补一个 `id` 字段、把 key 换成它，是这次改动里最便宜的一行。

另外前端 `iconMap` 里的 `FileText` 永远用不上：后端只会返回 `CheckCircle`/`Activity`/`AlertTriangle` 三个名字，未知名字反而落到 `Activity` 兜底。不影响功能，只是说明这个「字符串图标名」的契约两边没对齐过。

## 可数的几个数

1 张表、5 个写入点、前端 30 条 / 每页 3 条 / 10 页、一篇研报 4–10 条。所以它在演示里「一直有内容」，其实是那三篇研报在反复占位。

落点动作很具体：每加一张列表卡，先把它的「数据源表 + 写入点个数」写在代码注释里。写不出写入点的卡，展示的不是这个系统的行为，而是某个子模块的日志。这张卡的写入点全是 Agent 步骤，那它的标题就该叫「研报流水线」，而不是「最近活动」。
