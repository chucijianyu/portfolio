# 删完主记录才去清向量，失败了就没有第二次机会

复核知识库的「删除文档」时，我在函数注释里看到一行很自信的话：V4.7 修复，删除文档时同步清理 Chroma 向量库中对应的 embeddings，避免残留导致检索指向已删除的文档。

代码确实写了。但写完不等于能兜住——这条链路把三个删除动作排成了「文件 → 主记录 → 向量」，而最后一步是所有补偿动作里唯一可能失败的，也是唯一在失败后失去重试资格的。顺序错了，注释再对也没用。

## 三段顺序，问题在中间那段

端点挂在 `/reports/{document_id}` 上（`api/reports.py:552`），不在 `/knowledge/*` 里，权限是 `require_role("admin")`。去掉入参与权限声明，主体只有三件事：

```python
if doc.file_path and os.path.exists(doc.file_path):
    os.remove(doc.file_path)          # 569: 先删文件

doc_id = doc.id
db.delete(doc)
db.commit()                            # 574: 主记录落库删除

try:
    rag_service = RAGService()         # 579: 为删一次向量现造一个实例
    if rag_service._collection is not None:
        rag_service._collection.delete(where={"document_id": doc_id})
except Exception as e:
    logger.warning(f"文档 {doc_id} 清理Chroma向量失败(不影响删除流程): {e}")
```

第 574 行 commit 之后，第 581 行才动向量库。而函数开头第一件事是：

```python
doc = db.query(Document).filter(Document.id == document_id).first()
if not doc:
    raise HTTPException(status_code=404, detail="文档不存在")   # 565
```

把这两段拼起来看：向量清理一旦失败，主记录已经没了，再调一次端点，第 565 行先返回 404。**孤儿向量不是「下次再清」，是没有下次。** 唯一还能清掉它的路径是全库重建，而重建本身的代价我在另一篇里写过。

## 第二个漏口：清理失败可能一行日志都没有

第 579 行为了拿一个 `_collection` 而新造 `RAGService()`。构造函数里 `_init_chroma()` 的收尾是这样：

```python
except Exception as e:
    logger.warning(f"Chroma初始化失败: {e}")
```

它把异常咽了，`self._collection` 留在 `__init__` 里赋的 `None`。于是删除端点的 `if rag_service._collection is not None` 不成立，整段直接跳过——不进 `try`，不触发 `except`，`logger.warning` 也不打。**「向量没清掉」和「向量本来就空」在日志上完全同形**，排查时看不出区别。

## 第三个漏口：删完了，状态卡还是旧数

同一文件里，上传端点在 `finally` 里做了失效，注释写得很清楚：

```python
finally:
    # V5.3: 无论处理成功或失败，都失效知识库状态缓存
    # 确保前端总文档数/已完成/处理中/向量数等指标为实时真实数据
    invalidate_cache('cache:kb:status')
```

删除端点在 552–585 这段里，`invalidate_cache` 出现 **0 次**。而 `/knowledge/status` 挂着 `@cached(prefix='cache:kb:status', ttl=CACHE_KB_STATUS_TTL)`，默认 `60` 秒。删完立刻回列表页，文档数、`completed_documents`、`vector_count` 最多 60 秒内还是旧值。用户看到的是「删了没生效」，第二次再点，这次是真 404。

## 我原本以为的 bug，不是 bug

先记一笔反证：`document_chunks` 表里没有任何显式 `delete` 语句，我第一反应是分块会残留。查下去发现不会：`Document.chunks` 声明了 `cascade="all, delete-orphan"`，`DocumentChunk.document_id` 的 FK 也写了 `ondelete="CASCADE"`，两条路都通，`db.delete(doc)` 会带走分块。

这条重要，因为它说明：**「表里没写 delete」不等于「没删」**。判断级联要看关系声明和外键约束，不能靠 grep 关键字。

反过来，真正要单独清的是另外三处派生状态，它们都不在任何一张表的级联路径上：Chroma 向量、Neo4j 图谱节点、BM25 语料。`knowledge_graph_service.py` 里连一个 `delete` 方法都没有，删完文档，那条 `Company` 节点和它的 `RELATES_TO` 边原地不动；BM25 靠 `(分块总数, sec_code)` 做缓存键，删除后总数变了下次才重建，删除动作本身不通知它。

## 顺序应该怎么排

改法（尚未上线，只写在这里）：把「不可重建的派生数据」放最前，「决定重试资格的主记录」放最后。

```python
doc_id = doc.id
file_path = doc.file_path

# 1. 先清派生数据，失败就整单失败，主记录还在，可以重试
try:
    rag_service = RAGService()
    if rag_service._collection is None:
        raise RuntimeError("Chroma 未初始化，向量未清理")
    rag_service._collection.delete(where={"document_id": doc_id})
except Exception:
    db.rollback()
    raise HTTPException(status_code=503, detail="向量清理失败，请重试")

# 2. 再删主记录与文件（文件删失败只记日志，不阻断）
db.delete(doc)
db.commit()
invalidate_cache('cache:kb:status')
```

两个要点：向量清理失败时把 `_collection is None` 也当失败抛出来，别让「没初始化」混进「本来就没有」；删除成功后补一次缓存失效，跟上传端点对齐。

顺序这件事，在我这里最后落成一句可搬走的话：**删除操作里，谁决定了「还能不能重试」，谁就必须最后一个删。** 反过来排，等于给自己留了一条只能走一次的补偿路径。
