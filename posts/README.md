# 技术笔记

记录做项目时真正卡住过的位置——现象、定位、修法、以及怎么证明没改坏。每天一篇，不定期回补。

共 **27** 篇。

| 日期 | 标题 | 标签 |
|---|---|---|
| 2026-11-26 | [五个限流配置，只有一个端点用上了](2026-11-26-login-rate-limit-one-endpoint.md) | `限流` `缺陷排查` `FastAPI` |
| 2026-11-25 | [一次 3 秒读超时之后，缓存对整个进程关闭了](2026-11-25-cache-disabled-forever-after-one-timeout.md) | `缓存` `韧性设计` `缺陷排查` |
| 2026-11-24 | [分页组件把页码夹紧了，可它只管显示，取数的是外面那份](2026-11-24-pagination-clamp-only-for-render.md) | `前端` `缺陷排查` `React` |
| 2026-11-23 | [号称向量加 BM25 的混合检索，研报生成时 BM25 那一路一直是空的](2026-11-23-hybrid-search-bm25-leg-always-empty.md) | `RAG` `缺陷排查` `检索` |
| 2026-11-22 | [大屏上「最近活动」的最大视野是三篇研报](2026-11-22-recent-activities-only-agent-steps.md) | `Agent` `可观测性` `前端` |
| 2026-11-21 | [三个字段都在渲染，接口一个都没返回](2026-11-21-knowledge-list-fields-never-returned.md) | `FastAPI` `接口契约` `缺陷排查` |
| 2026-11-20 | [上面七天，下面全历史：一张大屏上两个同名的数](2026-11-20-trend-vs-task-distribution-window-mismatch.md) | `指标口径` `缺陷排查` |
| 2026-11-19 | [三条告警规则，取不到数就显示「正常」](2026-11-19-alerts-fail-to-normal.md) | `可观测性` `缺陷排查` |
| 2026-11-18 | [16 张指标卡，7 张是量出来的，6 张是乘出来的](2026-11-18-dashboard-metrics-derived-by-coefficients.md) | `可观测性` `指标设计` `缺陷排查` |
| 2026-11-17 | [删完主记录才去清向量，失败了就没有第二次机会](2026-11-17-delete-document-before-vector-cleanup.md) | `RAG` `缺陷排查` |
| 2026-11-16 | [点一次「重新解析」，整个知识库被重跑了一遍](2026-11-16-rebuild-index-full-library.md) | `RAG` `缺陷排查` |
| 2026-11-15 | [说明书第 8 行写着分层索引，代码里它从没被实例化](2026-11-15-hierarchical-index-never-instantiated.md) | `RAG` `缺陷排查` `工程方法` |
| 2026-11-14 | [精排只看到每块前 200 字：一次被超时逼出来的截断](2026-11-14-rerank-input-truncated-to-200.md) | `RAG` `重排` `缺陷排查` |
| 2026-11-12 | [三次重试一次都没轮到 5xx：LLM 调用容错的三处漏口](2026-11-12-llm-retry-connection-layer-only.md) | `LLM` `容错` `缺陷排查` |
| 2026-11-11 | [本地账本说没发过，远端说已经在了：幂等判断该信谁](2026-11-11-idempotency-check-remote-not-ledger.md) | `工程方法` `定时任务` `幂等` |
| 2026-11-10 | [三个文件一次提交：把正文、索引和入口绑在同一次 ref 更新上](2026-11-10-atomic-commit-posts-and-index.md) | `Git` `工程方法` `定时任务` |
| 2026-11-09 | [昨天 0/0，今天 25 条 error：设计规则不落盘就等于没通过](2026-11-09-kicad-dru-track-width-rule.md) | `PCB` `KiCad` `可复现性` |
| 2026-11-08 | [21 段轮廓拼不出一个外形：把 AI 生成的板子做到可下单](2026-11-08-ai-board-to-orderable-gerber.md) | `PCB` `KiCad` `缺陷排查` |
| 2026-11-07 | [规则拿否决权，模型给分数：研报合规审核里谁在查什么](2026-11-07-compliance-rules-vs-llm.md) | `Agent` `合规审核` `缺陷排查` |
| 2026-11-06 | [每个块都以前一块的尾巴开头：64 字符 overlap 怎么骗过探针](2026-11-06-chunk-overlap-64-probe.md) | `RAG` `缺陷排查` `评测` |
| 2026-11-05 | [92 个 pytest 用例全绿，但页码回填那 34 行没有一条断言碰它](2026-11-05-pytest-92-which-are-real-regression.md) | `测试` `RAG` `缺陷排查` |
| 2026-11-04 | [Neo4j 在这套系统里是只写的：9 处实例化，没有一处进检索](2026-11-04-neo4j-write-only-graph.md) | `Neo4j` `知识图谱` `架构权衡` |
| 2026-11-03 | [工具调用全部成功，因为失败从不抛异常](2026-11-03-tool-call-schema-mismatch.md) | `Function Calling` `可观测性` `缺陷排查` |
| 2026-11-02 | [LangGraph 四角色研报流水线：State 里放什么，谁来定下一步](2026-11-02-langgraph-report-pipeline-state.md) | `LangGraph` `Agent` `状态机` |
| 2026-11-01 | [大屏上那个幻觉率 0%：分子和分母各是什么](2026-11-01-hallucination-rate-zero.md) | `RAG` `指标口径` `缺陷排查` |
| 2026-10-31 | [命中@5 90% 是怎么算出来的：20 道题和一张判定表](2026-10-31-retrieval-hit-at-5-eval.md) | `RAG` `评测` `工程实践` |
| 2026-10-27 | [引用点不回原文：一次 RAG 页码管道的排查记录](2026-10-27-rag-citation-pages.md) | `RAG` `缺陷排查` `工程实践` |
