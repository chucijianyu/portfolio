# 点一次「重新解析」，整个知识库被重跑了一遍

知识库页面的文档操作菜单里有个「重新解析」。我一直以为它只处理这一篇。直到读代码才发现，`handleReparse` 收了一个 `doc` 参数，从头到尾没用过：

```jsx
const handleReparse = async (doc) => {
  setActionMenuId(null)
  addToast(`文档"${doc.title}"重新解析已触发`, 'info')
  try {
    await axios.post('/api/v1/knowledge/rebuild')
    loadDocuments()
    loadStatus()
  } catch { addToast('重新解析失败', 'error') }
}
```

（`frontend/src/pages/KnowledgeBase.jsx:192` 起）

提示语里写着这篇文档的名字，请求体是空的。再往上翻，页面顶部那个「重建索引」按钮走的也是同一个地址。

## 后端根本没有「一篇」这个入参

```python
@router.post("/rebuild", summary="重建知识库索引")
async def rebuild_index(
    current_user: User = Depends(require_role("admin")),
    db: Session = Depends(get_db)
):
    rag_service = RAGService()
    result = await rag_service.rebuild_index(db)
    invalidate_cache('cache:kb:status')
    return result
```

（`backend/app/api/knowledge.py:60`）

没有 document_id，也没有 doc_type。到 `rag_service.py:603` 就清楚了：重建 = 先 `delete_collection(settings.CHROMA_COLLECTION)`，再按 `Document.status == "completed"` 把全表文档捞出来，逐篇 `process_document`。返回的两个键是 `total` 和 `success`，没有「哪些失败了」。

## 代价不在时间，在分块表

`rebuild_index` 只删了向量集合，没碰 MySQL 里的分块。而 `process_document` 每处理一篇都新插一整套 `DocumentChunk`，`document_chunks` 表（`database_models.py:56` 起）上没有 `(document_id, chunk_index)` 唯一约束，整个 backend 也搜不到一处 `delete(DocumentChunk)`。

两边的主键生成方式决定了后果不一样：向量侧是 `chunk_id = f"doc_{doc.id}_chunk_{chunk.chunk_index}"`，同一篇重跑写的还是那批 id；MySQL 侧是自增主键，每跑一次就是新的一批行。所以「向量数没变、分块数翻倍」不是异常，是这两条链路各走各的必然结果。

可见的副作用有三个：知识库列表的 `chunk_count` 直接查 `DocumentChunk`（`knowledge.py:121`），数字会随重建次数线性上涨；`/documents/{id}/chunks` 会把同一段文字返回多份；BM25 是 `_build_bm25_from_db` 全量从 MySQL 加载语料建的，同一段文字在 BM25 里出现几次，分数和排名就不是原来那个了。顺带一提，那个缓存的键是 `(分块总数, sec_code)`——总分块数一变就必然失效，正好被上一步的翻倍推着重算。

单篇的代价也不止一次解析：`process_document` 对每个分块调一次 `embed_query`，整篇跑完再用 `_generate_doc_summary` 调一次大模型生成摘要。全量重建就是「文档数 ×（分块数次嵌入 + 1 次 LLM）」，而这个函数是同步 await 的，请求会一直挂着。

## 重建期间，状态字段在发抖

`process_document` 做的第一件事是把 `doc.status` 置成 `processing` 并 commit，跑完再置 `completed`。于是重建期间 `/knowledge/status` 的 `completed_documents` 会逐篇减少、`processing_documents` 逐篇增加。这个接口挂着 `@cached(ttl=settings.CACHE_KB_STATUS_TTL)`，而 `invalidate_cache` 是在整个重建结束之后才调的——中途刷新页面，很可能把一张「正在重建中」的快照缓存住，然后对着它判断系统是不是挂了。

## 失败的那一篇会永久出列

`process_document` 的异常分支把 `doc.status` 改成 `failed` 再 commit，而重建的候选集是 `status == "completed"`。所以只要某篇在重建中失败一次（解析器读不了某个 pdf、嵌入服务超时），它就再也不会进入下一次重建。不是「这次没成功」，是「以后都不试了」，唯一痕迹是一行 log 和 `success < total`。

而前端 `handleRebuild` 根本没读 `res.data`，只弹了一句「索引重建已启动」——这个接口是同步 await 的，弹窗出现的时刻整个库已经跑完了。

还有两处换手也是静默的。`delete_collection` 外面包着 `except Exception: pass`，删不掉照样往下走，于是「清空后重建」可能变成「没清空又叠一层」。`self.retriever.vector_store` 只在 `_init_chroma`（`rag_service.py:64`）绑过一次，重建把 `self._collection` 换成新集合，retriever 手里还是旧的那个——现在因为端点每次 new 一个 `RAGService()`，这个错位没暴露；但 `agent_service.py:1547` 已经把实例缓存在 `self._rag_service` 上，哪天重建也从那里走，检索就会静默返回空。

## 我会怎么改

三件事，按顺序：

1. 重建前先按 document_id 删分块行，再删向量；两个存储的删除要么都在，要么都不在。
2. 给 `document_chunks` 加 `UniqueConstraint(document_id, chunk_index)`，让重复插入直接报错，而不是默默多一行。
3. rebuild 的返回值带上 `failed: [{document_id, error}]`，前端读它并展示出来。

至于「单篇重新解析」，它该有自己的端点：收 document_id，只删这一篇的分块与向量，只跑这一篇。

一句可搬走的经验：**重建的成本不在跑多久，在于它偷偷改了哪些表**。一个只有 total 和 success 两个键的返回值，等于没有返回值——「失败率 0」和「失败被吞掉」在接口上看是同一件事。
