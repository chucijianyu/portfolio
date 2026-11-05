# 92 个 pytest 用例全绿，但页码回填那 34 行没有一条断言碰它

## 现象

`backend/tests/` 下六个文件，`def test_` 一共 92 个：`test_api` 11、`test_auth` 38、`test_compliance` 6、`test_ml` 15、`test_rag` 15、`test_tools` 7。同级还有一个空的 `tests/` 目录，里面什么都没有，别看错地方。

跑起来全绿。但那段时间真正在修的事——文本块的 `page_number` 一直落成 0，只有表格块能从解析结果拿到页——这 92 条一条都没管。

## 定位：判据只有一条

回归测试的判据不是「跑得快」「覆盖模块多」，就一句话：**把修复撤掉，它会不会红。**

我把 `page_number` 这个词在 `backend/tests/` 下搜了一遍，命中 0 次。带 `page` 的只有两处，都不相干：`test_auth.py:81` 注释里的「V4.6 分页结构」，以及 `test_rag.py:74` —— `test_table_chunking` 给表格传了 `{"content": ..., "page": 1}`。

第二处最容易骗人。它看着像在测页码，实际断言只有一句：

```python
table_chunks = [c for c in chunks if c.chunk_type == "table"]
assert len(table_chunks) == 1
```

表格块的页码是 `table.get("page", table.get("index", 0))` 直接从解析结果搬过来的，本来就不会错。被修的是文本块那条路径，而这条用例一行都没断言。

旁边的 `test_chunk_metadata` 更典型：断 `metadata["document_id"] == 1`、`chunk_index >= 0`，就是不碰 `page_number`。于是这 15 条 RAG 用例的断言集合，改前改后一模一样——它们证明的是「分块器还在跑」，不是「页码修好了」。

还有个容易忽略的地方：`TestFinanceChunker` 的 `setup_method` 里写的是 `FinanceChunker(chunk_size=200, chunk_overlap=30)`，而生产的默认值是 512 / 64。这批用例从头到尾跑在一个比生产小一号的配置上。分块是阈值敏感的，配置换了，走的边界就不是同一条。

## 按这条判据过一遍

- **38 条 auth**：能红。把 `require_roles` 那类依赖摘掉，`test_analyst_cannot_list_users` 立刻从 403 变 200。大部分是权限/角色断言，是这批里最硬的。
- **6 条 compliance**：能红。断的是具体违规项字符串，规则一改就掉。
- **15 条 ml**：能红。`r_squared > 0.8`、`trend_direction == "上升"` 是真数值断言。
- **11 条 api**：其中 9 条只断 `status_code == 200`，是通路冒烟。字段错值它不管——`industry_comparison` 永远返回 PE 分位那事，就活在这 9 条底下。
- **7 条 tools**：断「有这个字段」，而三个数据源都不抛异常、失败回落 mock，所以这 7 条永远绿。
- **15 条 rag**：断块数、`chunk_type`、元数据是否存在，断不到页码。

92 条里，能替你守住页码回填的是 0 条。

## 该补的断言

不要断块数（`chunk_size` / `overlap` 一动块数就变，用例跟着碎），断**属性**：

```python
def test_text_chunk_page_backfill():
    chunker = FinanceChunker(chunk_size=200, chunk_overlap=30)
    text = "[第1页]" + "第一页内容。" * 20 + "\n\n[第2页]" + "第二页内容。" * 20
    chunks = chunker.chunk(text)
    text_chunks = [c for c in chunks if c.chunk_type == "text"]

    assert text_chunks
    assert text_chunks[0].page_number == 1
    assert all(c.page_number > 0 for c in text_chunks)   # 修复前这里全是 0
    assert 2 in {c.page_number for c in text_chunks}     # 第二页必须出现过
```

后两行才是关键：`all(... > 0)` 在原实现上必红（文本块恒 0），在现实现上必绿；`2 in {...}` 防的是「只回填了首页」这种半修。

跨页块得单独一条，因为 `_resolve_pages` 对「标记落在块中部」是另一套处理：块主体仍属上一页，只把尾段记进 `end_page`，`start_page` 继承上一块的末页。要断的是 `metadata["start_page"]` 与 `metadata["end_page"]` 这一对，不是 `page_number`。

还有个坑：`head_tolerance = self.chunk_overlap + 8`。写用例时图顺手用 `chunk_overlap=30`，容差就是 38；生产默认 `overlap=64`，容差 72。同一个标记位置，两边会走进不同分支。想测生产行为，就把参数对齐生产默认值，否则你测的是另一条边界。

## 结果

`_resolve_pages` 正文从第 99 行到第 132 行，34 行。这条修复对外的数字是分块页码字段填充率：本机 981 块，填上的从 226 涨到 874，23.0% → 89.1%。注意这是**字段填充率**，不是溯源准确率；另有一批 357/1430（全部是表格块）的 24.97%，属不同数据集，两个数不能混算也不能相加。

而能守住这个 89.1% 不往下掉的用例，在补之前是 0 条。

## 一句可搬走的

**「测试全绿」和「修复被守住」是两个命题，中间隔着一个动作：把修复反向撤掉，再跑一遍。** 92 条用例、六个模块，真能回答这个问题的只有断言落在「修复前长什么样」上的那些——其余的只是在证明代码还能 import，顺带回你一句绿灯。
