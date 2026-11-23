# 号称向量加 BM25 的混合检索，研报生成时 BM25 那一路一直是空的

检索函数的注释写得很满：向量 ANN 粗排、BM25 关键词粗排、RRF 融合、Cross-Encoder 精排，四个阶段一个不少。但研报生成时去查历史研报，那条 BM25 支路返回的结果恒为 0，而且不报错、不告警、日志级别还是 info。

先看清楚调用链。研报流水线的 DataCollector 节点在 `agent_service.py:1071` 里并行拉五路数据，其中第四路是历史研报：

```python
results = await asyncio.gather(
    self.tool_service.stock_price_query(sec_code, "1y"),
    ...
    self._retrieve_historical_reports(sec_code, 5, company_name),
    return_exceptions=True
)
```

`_retrieve_historical_reports` 本身很简洁，拿到 RAG 实例后只做一件事：

```python
rag = self._get_rag_service()
query = f"{company_name or ''} {sec_code} 研报分析 投资建议 财务".strip()
results = await rag.search(query=query, top_k=top_k)
```

问题出在 `search()` 的签名上。它的 `db` 参数默认 `None`，而构建 BM25 索引需要数据库会话：

```python
async def search(self, query: str, top_k: int = 5,
                 mode: str = "hybrid", use_rerank: bool = True,
                 db=None) -> List[Dict]:
    if not self.retriever:
        return []
    # bm25/hybrid模式需要从数据库构建BM25索引
    if mode in ("bm25", "hybrid") and db is not None:
        self._build_bm25_from_db(db)
    if mode == "hybrid":
        results = self.retriever.retrieve(query=query, top_k=top_k)
```

调用处没传 `db`，这个 `and db is not None` 就成了短路点。构建被跳过，而下面 `retrieve()` 照样执行，它内部会调 `_bm25_search`，那里还有第二道静默返回：

```python
def _bm25_search(self, query: str, top_k: int = 20) -> List[Dict]:
    if not self._bm25_docs:
        return []
```

`_bm25_docs` 从没被赋过值，因为 `build_bm25_index()` 一次都没跑过。于是混合检索退化成纯向量检索，RRF 融合的一半是空列表。`retrieve()` 结尾那行日志其实一直在说这件事：`向量N条, BM25 0条, 合并后M条`——答案写在日志里，只是没人去看。

同一段代码里还有第二个漏口。`search()` 委托给 `retriever.retrieve(query=query, top_k=top_k)`，没有传 `sec_code`。而 `_vector_search` 的过滤条件是现成的：

```python
where_filter = None
if sec_code:
    where_filter = {"sec_code": sec_code}
```

传 None，`where` 就是 None，全库检索。V5.8 的注释写「未显式指定证券代码时，从问题中识别公司主体，唯一命中时将检索范围过滤至该公司文档」，这条修复只覆盖了 `ask()` 和 SSE 那条链路（`rag_service.py:399`、`reports.py:177` 都显式传了 `sec_code`），走 `search()` 的这条没有。研报侧最后只剩 `_content_matches_target()` 事后过滤：已经召回的片段里不含目标公司名、却含别家公司名的，直接丢掉。这是丢，不是筛。

把三个入口摆在一起，差别一目了然：

| 入口 | 传 db | 传 sec_code | BM25 实际生效 |
|---|---|---|---|
| `rag_service.ask()` :399 | 是 | 是 | 是 |
| `reports.py:177` 问答 SSE | 是 | 是 | 是 |
| `agent_service` → `search()` | 否 | 否 | 否 |

真正的功能入口（研报生成）落在第三行，测试性质的检索接口反而走的是完整路径——`api/knowledge.py:93` 的搜索端点是把 `db=db` 老老实实传下去的。这类错位有个共同特征：能跑通的那条路是手动点的，跑不通的那条是自动触发的。

代价有多大？我用仓库自带的 15.9 万字需求文档，按项目里的 `FinanceChunker(512, 64)` 切块，再用真实的 `HybridRetriever.build_bm25_index` 跑构建，本机实测（三次取中位）：

| 块数 | 总字符 | 构建耗时 | 每块 |
|---|---|---|---|
| 346 | 159,267 | 167 ms | 0.48 ms |
| 692 | 318,534 | 335 ms | 0.48 ms |
| 1,384 | 637,068 | 1,269 ms | 0.92 ms |
| 2,768 | 1,274,136 | 2,637 ms | 0.95 ms |

按每块 0.5–0.9 ms 线性外推，一个近千块的知识库重建一次大约 0.5–0.9 秒。这个量级本身不算灾难，但 `ask()` 是 `async def`，`_build_bm25_from_db` 是同步调用、没有丢进 threadpool，jieba 分词是纯 CPU 计算——这 0.6 秒会占住事件循环，期间其它请求（包括正在流式吐字的研报 SSE）一起等。

顺带说清那层听起来很聪明的缓存。`_build_bm25_from_db` 用 `cache_key = (total_chunks, sec_code)` 做单槽缓存，注释写「仅当语料变化时重建，避免每次查询全量构建」。但 `RAGService()` 在 backend 里共 9 处实例化，8 处是按请求 new 的，只有 `agent_service.py:1547` 把实例存在 `self._rag_service` 上——而 AgentService 自身也是按请求 new 的（`api/reports.py` 三处）。所以这个缓存的生命周期等于一次请求，跨请求命中率接近 0；每次请求仍要先跑一条 `COUNT(*)` 才能确认没命中。缓存键用行数而非内容指纹，行数不变而内容变了的时候，它会安静地继续用旧索引。

修法不复杂，三件事：`search()` 增加 `sec_code` 参数并透传给 `retrieve()`；调用处把 `db` 显式传进去——如果它其实是必填的，就别给默认值，让缺参在启动时就炸；构建动作挪进 `run_in_threadpool`，或者干脆在文档入库时增量更新，而不是在查询路径上重建。

这件事的可搬走经验只有一句：把必填依赖写成可选参数，缺参不会报错，只会让一条路径静默消失。所以「可选参数」在调用链上必须被当成断言，而不是默认值。
