# 说明书第 8 行写着分层索引，代码里它从没被实例化

整理面试材料时翻到 `智融析系统使用说明书.md` 里的「核心技术」表。第 8 行是：分层索引 / `HierarchicalFinanceIndex` / 报告级摘要 → 段落级精排。这张表一共 11 行，是我对外讲系统时最容易被逐行追问的地方，所以我打算把每一行都指到代码上。前 7 行几分钟就指完了。到第 8 行，我搜了半小时没搜到入口。

## 我怎么定位

先搜类名。全仓 3 处命中：`retriever.py:575` 是定义，`项目需求文档.md:3232` 把源码原样贴进了文档，`使用说明书.md:472` 就是这行表格。backend 里 `HierarchicalFinanceIndex(...)` 这种写法 **0 处**。

再搜方法名。`hierarchical_retrieve` 只有 `retriever.py:723` 一处定义，另外两个命中是 `__pycache__` 里的 pyc。

从生产入口反向走才对得上：`rag_service._init_chroma()` 只建一个集合，名字取自配置 `settings.CHROMA_COLLECTION`（`config.py:72`，默认 `finance_documents`），然后 `HybridRetriever(vector_store=self._collection)`。真实检索方法 `retrieve()`（`retriever.py:91`）只做四件事：向量检索取 `top_k*2` → BM25 取 `top_k*2` → RRF 融合（k=60）→ CrossEncoder 精排 → `merged[:top_k]`。整条链路里没有第二次检索，也没有集合切换。

而分层索引要的是两个集合：`finance_doc_level`（`:614`）和 `finance_chunk_level`（`:619`）。这两个名字全仓只在这两行出现，再没有任何一处读写它们——因为 `__init__` 从没被执行过，`_init_collections()` 里那两行也就从没跑过。

## 它不缺实现，缺一个调用方

这个类写了 349 行（575 到 923），单看代码是完整的：`build_index` 真做了两层写入，第一层把摘要 `embed_query` 后进 doc 集合，第二层把 `embed_documents(chunks)` 进 chunk 集合并带 `parent_doc_id`；`_generate_doc_summary` 先试 LLM（`REPORT_SUMMARY` 模板，`content[:3000]` 进、超过 50 字才取 `summary[:1000]`），失败降级成 `content[:800]`；`hierarchical_retrieve` 真写了「先取 5 篇、再在这 5 篇里取 8 块」；`_apply_time_decay` 真按 `exp(-age_days/90)` 算完再重排。

问题在于它的失败方式是安静的：

```python
def _init_collections(self):
    if not self.chroma_client:
        logger.warning("未提供Chroma客户端，分层索引将无法持久化")
        return          # 不抛异常，两个 collection 保持 None

def build_index(self, documents):
    if not self._doc_collection or not self._chunk_collection:
        logger.error("Chroma集合未初始化，无法构建索引")
        return {"status": "failed", "reason": "集合未初始化"}
```

构造函数前三个参数（`chroma_client` / `embedding_engine` / `llm_service`）默认都是 `None`，所以 `HierarchicalFinanceIndex()` 完全合法；不传 client 只打一条 warning，接着 `build_index` 返回 `{"status": "failed"}`，`hierarchical_retrieve` 返回 `[]`。**「接上了但没生效」和「根本没接」在日志里长得一模一样**，这大概就是它能在文档里活这么久的原因。

## 同一张表的另外两行

顺着表往下查，第 11 行「时间衰减 exp(-age_days/90)」也点不进去。这个函数在同一文件里有两份实现：`HybridRetriever._apply_time_decay`（`:476`）和 `HierarchicalFinanceIndex._apply_time_decay`（`:861`）。两个调用点分别是 `multi_query_retrieve`（`:178`）和 `hierarchical_retrieve`（`:808`），而这两个方法本身全仓 0 个外部调用者。真实入口 `retrieve()` 不带衰减。

作为对照，第 9 行「查询重写」是真接上的：`rag_service.py:39` 有 `QueryRewriter()` 实例化，`:497` 把 `query_rewritten` 回填进返回值。所以这不是「文档整体吹水」，是 11 行里 9 行能指到代码、2 行指不到。

还有个更小的：同一张表写「bge-large-zh-v1.5，768 维」，而 `config.py:73` 的注释写「1024 维」。两个文件对同一个模型给了两个维度。

## 我的处理

我没有去把分层索引接上。理由是生产库只有 `finance_documents` 一个集合、检索侧是 `top_k*2` 召回加精排，在这个规模下「先粗后精」多出来的那一层没有可预期的收益，而补它意味着把同一批分块再往第二个集合写一遍。

我实际做的是一张三列对账表：文档里的名字 / 类或函数 / 从生产入口第几跳能走到。判据只有一条——**从 API 入口顺着调用链能不能走到它**，而不是「文件里有没有这个类」。11 行过完，9 行有入口、2 行没有、维度描述冲突 1 处。下一步是把说明书那两行标成「未启用」，或者干脆从表里删掉。

## 可搬走的一句

「代码里写了」和「系统里有」之间差一次实例化。查这件事不用读代码：grep 类名看有没有调用点，再反查生产入口的初始化函数看它建了什么。两次搜索就能定位。真正花时间的是接受结果——那 349 行看起来完全能用的代码，其实一次都没跑过。
