# 精排只看到每块前 200 字：一次被超时逼出来的截断

指标页上「重排提升」那一格永远是空的：`monitor.py:420` 里 `rerank_improvement` 的 `value` 被写死成 `None`，`target` 倒是认真写了 `0.15`。想知道重排到底提升了多少，只能去读实现。读到的第一个数字是 200。

## 现象：200 这个数不是从相关性里来的

`backend/app/rag/retriever.py:353` 有两行注释，写得很诚实：

```python
# CPU推理下精排耗时与文本长度近似线性：40条×500字符约需45秒，会超出超时降级
# 截断至200字符(相关性信号通常集中在段首)，实测40条约13秒，稳定落在超时窗口内
documents = [item.get("content", "")[:200] for item in results]
```

200 是从 30 秒超时里倒推出来的，不是从「哪一段最像答案」里挑出来的。问题不在于选了这个数，在于这个预算只活在注释里——配置项里没有它，日志里也不打它。

## 定位：四个数字对不上

顺着这条路把链路读了一遍：

- 客户端 `httpx.Client(timeout=30.0)`（`retriever.py:359`）；
- 服务端 `top_n: int = Field(5, ge=1, le=100)`（`rerank_service/main.py:68`），默认 5；
- 主项目实际传的是 `"top_n": len(documents)`（`retriever.py:362`），等于永远让精排给全部候选排序，回到主项目再用 `merged[:top_k]` 截；
- `settings.RERANK_TOP_N = 8` 与 `RAG_RERANK_TOP_N = 8`（`core/config.py:84-85`）在整个 backend 只有一处被读——`config.py:228` 判断两者是否相等然后互相赋值。检索侧一次都没读过它。

也就是说，配置里那个 8 是死的，真正生效的是「候选有多少就排多少」。top_k=10 时向量召回 20 条、BM25 召回 20 条，RRF 合并去重后可能接近 40 条，全部塞给 Cross-Encoder。

服务端还有一道同样关不上的闸：`document=doc[:500]`（`main.py:128`）。输入已经 ≤200，这个 500 永远切不到东西。

有一个点我本来怀疑是 bug，查完确认是对的：服务端 `index=i` 取的是**原始列表下标**（`main.py:124-129`），排序只动列表顺序、不动 index，主项目 `item = results[idx]` 的映射正确。这一处不用改。

真正咬人的是 200 和 64 的叠加。分块时 `RAG_CHUNK_OVERLAP = 64`，每块都前置了前一块末尾的 64 个字符，所以精排读到的 200 字里，前 64 字是上一块的尾巴，占 32%。表格块最难受：一个表格块的开头常常是上一张表的最后一行，与本题无关，却占掉了三分之一的输入预算。

## 修法：把预算写进配置，把截断挪开块头

第一处改动，让 `top_n` 真正起作用——精排只负责选出前 N，不再给全量排序：

```python
# 改前：让精排对全部候选排序，截断留给外面的 merged[:top_k]
payload = {"query": query, "documents": documents, "top_n": len(documents)}

# 改后：N 由配置决定，精排的选择就是最终选择
top_n = min(len(documents), settings.RERANK_TOP_N)
payload = {"query": query, "documents": documents, "top_n": top_n}
```

第二处，截断起点避开 overlap：取 `content[64:264]`，或者在 chunker 侧把去掉 overlap 的正文存进 metadata，精排只读那个字段。后者更干净，但要求入库时多存一份文本，我先用了前者。

第三处是量纲。降级分支把两种分数写进了同一个字段名：

```python
except Exception as e:
    logger.warning(f"Rerank服务不可用，降级为关键词重叠度: {e}")
...
result["rerank_score"] = result.get("rrf_score", 0) * 0.7 + (overlap / max(len(query_terms), 1)) * 0.3
```

正常分支写的是 Cross-Encoder 分，降级分支写的是 RRF 与关键词重叠度的加权和，两者量纲完全不同，却共用 `rerank_score` 这一个名字。下游排序只看字段名，降级了没人知道。至少该同时写一个 `rerank_mode`。

## 顺带核到：时间衰减活在死代码里

`_apply_time_decay`（`retriever.py:476`）只被 `multi_query_retrieve`（`:133`）调用，而 `multi_query_retrieve` 全仓 0 个调用点。四个真实入口——`rag_service.py:400`、`:427`、`:585`、`api/reports.py:178`——走的全是 `retrieve()`。所以「上下文扩展 + 时间衰减」这套二次排序，在生产路径上一次都没跑过。

如果它跑起来，公式也值得看一眼：`decayed_score = base_score * exp(-age_days / 90)`。`exp(-90/90) = 0.368`，`exp(-365/90) = 0.017`。一条一年前的文档精排 0.99，衰减后是 0.0172；今天发布的文档精排 0.50，衰减后仍是 0.50。乘性衰减在分数挤在 0.9 附近时几乎不改次序，在跨度大时又直接压死。要么对名次做衰减，要么先分档再衰减。这段先记下不动——它在死代码里，改它没有任何可观测收益。

## 结果与一句经验

三处改完，能观测到的变化是：精排的请求体小了（`top_n` 从 40 降到 8），日志里能记下每次的候选数与耗时，200 这个数终于可以从注释搬进配置。至于「重排提升了多少」，还是给不出来——`rerank_improvement` 仍是 `None`。要证明它，得自己攒一版对照：同一批 query，`use_rerank=False` 与 `True` 各跑一次，比 top-5 命中，而不是等看板自己长出数字。

可搬走的一句：精排的输入长度不是超参数，是预算；预算既然是被超时推出来的，就必须写进配置和日志，否则下一个读代码的人只会看到一行注释，然后以为 200 是调参调出来的。
