# LangGraph 四角色研报流水线：State 里放什么，谁来定下一步

## 现象：四个角色，五个地方在决定"下一步"

智融析的研报生成链路，最早是四个函数顺序调用：采数据、做分析、写正文、审合规。四个 prompt，一个 `for` 循环，看起来就是流水线。

接上 LangGraph 之后我才发现，真正难的不是把四个函数变成四个节点，而是**"下一步去哪儿"这件事，当时散在五个地方**：

- 数据采集失败要不要继续？判断写在一个 `if not market_data: return` 里；
- 分析结论为空要不要退回重采？写在 prompt 末尾一句"若数据不足请说明"里，靠模型自觉；
- 审核不通过退回去改，改几次算完？写在循环变量里；
- 审核节点自己抛异常了怎么办？没有处理，整个 `ainvoke` 直接冒泡；
- 人工审核要暂停，暂停点在哪？没有，全靠调用方在中间插一段代码。

五处逻辑，五种写法，一处改动就要顺着五个地方对一遍。这就是我要解决的问题：**不是 prompt 不够好，是流程控制没有归宿。**

## 我怎么定位：State 混了三类东西

翻自己写的 `ReportState`，25 个字段挤在一个 `TypedDict` 里，看上去整齐，实际上混了三类完全不同寿命的东西：

1. **能重放的**——`collected_data`、`analysis_conclusions`、`draft_report`，纯文本，重新算一遍就有；
2. **不能重放的**——`db` 会话、事件发射器，绑在进程和连接上，写进状态里就是给序列化埋雷；
3. **决定流程的**——`error`、`review_result`、`revision_count`，这几个字段的值直接等价于"下一条边走哪根线"。

第一类该进 checkpointer，第二类一个都不能进，第三类才是这整套编排真正的主体。旧代码把三类平等对待，所以每次加功能都是"再塞一个字段"，路由判断也就跟着散开。

## 修法：先定字段所有权，再画边

第一步不是写图，是给 25 个字段标"谁写、谁读"。`ReportState` 按节点分区注释，每个节点只返回自己那一段，其余字段一个字都不碰：

```python
class ReportState(TypedDict, total=False):
    # ===== 输入参数 =====
    sec_code: str
    company_name: str
    analysis_framework: str
    task_id: int
    max_revisions: int          # 来自 settings.MAX_REVISION_COUNT，默认 3

    # ===== DataCollector 输出 =====
    collected_data: str
    data_status: Dict
    research_data: str          # RAG 检索的历史研报上下文

    # ===== Analyst 输出 =====
    analysis_conclusions: str
    key_findings: List[str]
    risk_points: List[str]

    # ===== Writer 输出 =====
    draft_report: str
    revision_count: int

    # ===== Reviewer 输出 =====
    review_result: str          # approved / revision_needed / forced_end
    review_score: float
    revision_feedback: List[str]

    # ===== 流程控制 =====
    error: str
```

`total=False` 是关键：节点返回 `{"draft_report": text}` 就是一次部分更新，不用把 25 个字段抄一遍，也不会因为漏抄某个字段把上游结果抹掉。字段所有权一旦写清，"某个字段被谁改过"就不再需要靠日志猜。

第二步，把"下一步"从 prompt 搬到边。四条业务线，加上一个错误兜底，全部走条件边：

```python
workflow = StateGraph(ReportState)
workflow.add_node("data_collector", self._wrap_node(self._data_collector_node, db))
workflow.add_node("analyst", self._wrap_node(self._analyst_node, db))
workflow.add_node("writer", self._wrap_node(self._writer_node, db))
workflow.add_node("error_handler", self._wrap_node(self._error_handler_node, db))
workflow.set_entry_point("data_collector")

workflow.add_conditional_edges(
    "data_collector", self._route_after_data,
    {"analyst": "analyst", "error_handler": "error_handler"},
)
workflow.add_conditional_edges(
    "reviewer", self._review_router,
    {"approved": END, "revision": "writer",
     "forced_end": END, "error_handler": "error_handler"},
)
workflow.add_edge("error_handler", END)
return workflow.compile(checkpointer=self._checkpointer)
```

第三步，路由函数写成**只读 state、不发请求**的纯函数。回退几次、什么时候强制收尾，全收到 `_review_router` 一处：

```python
def _review_router(self, state: ReportState) -> str:
    if state.get("error"):
        return "error_handler"
    result = state.get("review_result", "")
    if result in ("approved", "forced_end"):
        return result
    if state.get("revision_count", 0) >= state.get("max_revisions", settings.MAX_REVISION_COUNT):
        logger.info("达到最大修改次数，强制结束流程")
        return "forced_end"
    return "revision"
```

这带来一个很实际的好处：**路由可以单测**。造一个 dict，断言返回值，不需要起 LLM、不需要连库。以前"审核不过会退回几次"这个问题只能靠跑一遍真任务看日志，现在是一条断言。

## 代价：checkpointer 用了，但没真用上

这套东西跑通之后，我得如实说一个没解决的账。

`self._checkpointer = MemorySaver()`，注释写的是"状态持久化与中断恢复"。但实际代码里，每跑一次任务 `thread_id = f"report-{task.id}-{uuid.uuid4().hex[:8]}"`——**每次都是新线程**。也就是说内存里的检查点从来没有被 resume 读取过，只是一直在堆着，进程一重启全部消失。

真正让"人工审核后继续"能工作的是另一条路：阶段一把完整状态序列化进 `task.result["saved_state"]`，阶段二从数据库读回来，`state = dict(saved_state)`，然后直接手调 `_writer_node` 和 `_reviewer_node`。`resume_report_with_human_decision` 的 docstring 里我写得很直白：`thread_id: 仅用于日志，不再用于检查点恢复`。

根因是版本：LangGraph 0.0.20 不支持 `interrupt_before`，所以"暂停"不是靠 checkpointer 的 interrupt，而是靠**建两张拓扑不同的图**——`_build_graph(db, interrupt=True)` 时干脆不注册 `reviewer` 节点，Writer 之后直接 `END`。中断是拓扑差异模拟出来的，不是检查点能力。

想清楚这件事以后，判断标准反而清楚了：**checkpointer 只该装"能重放且需要跨进程"的状态**。这一版里这类状态其实只有一份，就是 `saved_state`，而它已经在数据库里了——所以 MemorySaver 在这个架构里是多余的一层，正确的做法是换成 SqliteSaver 或直接把 thread_id 固定成 `task.id`，让检查点真的可续。这是下一版要还的债。

## 一句可搬走的经验

动笔画图之前，先给 State 的每个字段标上"谁写、谁读、能不能重放"；标不出来，就别急着 `add_node`——因为 `add_edge` 之后，所有没写清的东西都会变成运行时才暴露的 if/else。
