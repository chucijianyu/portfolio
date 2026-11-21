# 三个字段都在渲染，接口一个都没返回

知识库页面每一条文档的右下角，代码里写了三个带守卫的 `span`：分块数、上传人、处理耗时。跑起来之后这三处一个都没出现过。列表看着挺干净，也没人报 bug——少显示一个字段不像坏掉，像一个还没用上的功能。

## 现象：三个分支都在，三个都不亮

守卫的写法本身没错，`{doc.chunk_count != null && …}`、`{doc.upload_user && …}`、`{doc.processing_time && …}`，字段是 `undefined` 就跳过。问题在于这三个字段从来没有被返回过。

前端取文档列表走的是这一行（`KnowledgeBase.jsx:74`）：

```js
const res = await axios.get('/api/v1/reports/list?limit=100')
setDocuments(res.data)
```

而 `/reports/list` 上挂了响应模型（`api/reports.py:90`）：

```python
@router.get("/list", response_model=list[DocumentResponse], summary="获取文档列表")
```

`DocumentResponse`（`models/schemas.py`）一共 13 个字段：id、title、source、doc_type、file_path、file_size、sec_code、publish_date、author、rating、summary、status、created_at。没有 `chunk_count`，没有 `upload_user`，没有 `processing_time`。Pydantic 按声明输出，多出来的东西一律不进响应体。

所以这三个渲染分支从上线起就是死的：不是 React 的问题，不是 `!= null` 的问题，是数据源里根本没有这三个键。

## 另一套列表接口，字段都在，但没人调

`api/knowledge.py:104` 还有一个 `GET /knowledge/documents`，手写 dict，不挂响应模型：

```python
@router.get("/documents", summary="获取文档列表")
async def list_documents(...):
    documents = db.query(Document).order_by(Document.created_at.desc()).offset(skip).limit(limit).all()
    return [
        {
            "id": doc.id,
            "title": doc.title,
            ...
            "chunk_count": db.query(DocumentChunk).filter(DocumentChunk.document_id == doc.id).count() if hasattr(doc, 'id') else 0,
            "created_at": doc.created_at.isoformat() if doc.created_at else None,
            "upload_user": getattr(doc, 'upload_user', 'admin'),
            "embedding_model": settings.EMBEDDING_MODEL,
            "processing_time": getattr(doc, 'processing_time', None),
        }
        for doc in documents
    ]
```

11 个键，前端要的三个刚好都在这里。但全前端搜 `knowledge/documents` 只有一处命中，是取某个文档分块的 `/documents/{id}/chunks`——列表接口本身没有任何调用点。两套接口各自演进，字段各长各的，谁也没跟谁对齐。

## 那三个字段，其实也撑不住

就算把前端改调 `/knowledge/documents`，这三行也显示不出东西。

`upload_user`：`getattr(doc, 'upload_user', 'admin')` 的默认值 `'admin'` 永远轮不到。`upload_user` 在模型里是一个 relationship（`models/database_models.py:53`，`relationship("User", back_populates="documents")`）——属性存在，所以 `getattr` 不会走默认值；有上传人时返回的是 `User` 实例，没上传人时返回 `None`。返回 `None` 是小事，返回 `User` 实例是另一回事：这个端点没挂 `response_model`，FastAPI 直接用 `jsonable_encoder` 兜。它的 `sqlalchemy_safe` 只排除 `_sa` 开头的内部键，剩下的列全吐。

我拿一个只有 `id`、`username`、`password_hash` 三个属性的普通对象做过对照：

```python
from fastapi.encoders import jsonable_encoder

class U: pass
obj = U(); obj.id = 1; obj.username = 'zhang'; obj.password_hash = '$2b$12$...'
jsonable_encoder({'upload_user': obj})
# {'upload_user': {'id': 1, 'username': 'zhang', 'password_hash': '$2b$12$...'}}
```

一个「显示上传人名字」的字段，实际会把整行用户记录送出去。这不是 `/reports/list` 那条路的问题——那条路有 `DocumentResponse` 白名单挡着；出问题的正是没人调的那条。

`processing_time`：`Document` 上根本没有这一列，最近的只有 `created_at` / `updated_at`。`getattr(..., None)` 每次返回 `None`，前端那句 `doc.processing_time.toFixed(1)` 也就永远执行不到。

`chunk_count`：能算出来，但是 `limit` 条文档就是 `limit` 次额外 COUNT。`/reports/list` 默认 `limit=20`，前端传 100，就是 100 条 `SELECT count(*)` 排在一串列表请求后面。

## 为什么三个死分支没人发现

翻测试就明白了。名字最像的那个用例叫 `test_document_list`，它打的是 `/api/v1/reports/list`，断 `status_code == 200`：

```python
def test_document_list(self, client, auth_headers):
    response = client.get("/api/v1/reports/list", headers=auth_headers)
    assert response.status_code == 200
```

`/knowledge/documents` 在测试里只出现一次，在 `test_auth.py:273`，断言的是分析师拿到 403——只验权限，不验响应体。也就是说：一条路有响应模型、有 200 断言，但没有字段断言；另一条路只被验过「不该进来的人进不来」。

更靠前一点的用例里还留着这种写法：

```python
assert response.status_code in [200, 500]  # 500可能因数据库连接问题
```

把 500 也算通过。这三个渲染分支之所以能活这么久，不是因为藏得深，是因为「不显示」和「显示对了」在测试里是同一个绿灯。

## 修法：把契约写在 schema 上，别写在 getattr 里

三件事，按顺序：

1. 删掉 `getattr(doc, 'upload_user', 'admin')`。要名字就显式取 `doc.upload_user.username if doc.upload_user else None`，要 id 就取 `upload_user_id`。`getattr` 带默认值只能兜「属性不存在」，兜不住「属性存在但类型不是你以为的那个」。
2. 给 `/knowledge/documents` 补 `response_model`，或者干脆删掉它，只留 `/reports/list` 一条路。两套列表接口并存，字段一定继续分叉。
3. 前端缺的字段，走 `/reports/list` 的 schema 加。`chunk_count` 用一次 `GROUP BY document_id` 聚合出来再拼，别在列表推导里查。

## 结果数字

- 前端渲染的三个字段：`chunk_count`、`upload_user`、`processing_time`，命中 0 个
- `DocumentResponse` 13 个字段，手写 dict 11 个键，三个字段只存在于后者
- `upload_user` 的实际取值：`User` 实例或 `None`，字符串 `'admin'` 出现 0 次
- `processing_time` 在 `Document` 上列数 0

## 一句可搬走的经验

接口返回什么，应该由 schema 声明，不能由 `getattr` 的默认值兜。默认值兜的只是「这个属性有没有」，而字段显示不出来的真实原因，往往是「属性在，但类型不是你要的那个」——它不报错，只是安静地变成 `undefined`。
