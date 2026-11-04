# Neo4j 在这套系统里是只写的：9 处实例化，没有一处进检索

知识图谱页面是这套系统里看起来最像模像样的模块：公司节点、行业节点、概念节点，画布上有圆点也有连线，点一下还能展开三级关联。直到我想在研报里引一句「同行业还有谁」，才发现没地方引。

于是我做了一件很笨的事：把 `KnowledgeGraphService` 在全 backend 的引用点一条条列出来，看谁会**读**它。

| 位置 | 处数 | 用途 |
|---|---|---|
| `api/knowledge_graph.py` | 4 | 两个查询端点、一个写入端点、一个健康检查 |
| `main.py` | 3 | 启动检查、`/health`、`/readyz` |
| `services/rag_service.py` | 1 | 文档入库后同步写节点 |
| `scripts/seed_knowledge_graph.py` | 1 | 30 家种子公司 |

读侧只有 api 里那两个端点，服务的对象是图谱页面本身。检索器、LangGraph 那条四节点流水线、`resolve_sec_code_from_question`——**0 处**。它是一段只写的代码。

## 一、写进去的边，查不出来也不会报错

写入端和读取端对「关系叫什么」没有共同真源：

```python
# 写入：关系类型由参数决定
def create_company_concept_relation(self, sec_code, concept_name,
                                    relation_type: str = "RELATES_TO") -> bool:
    cypher = (
        "MATCH (c:Company {sec_code: $sec_code}) "
        "MATCH (k:Concept {name: $concept_name}) "
        f"MERGE (c)-[:{relation_type}]->(k)"
    )

# 读取：关系类型写死在语句里
concept_result = session.run(
    "MATCH (c:Company {sec_code: $sec_code})-[:RELATES_TO]->(k:Concept) "
    "RETURN k.name as name, k.description as description",
    sec_code=sec_code
)
```

只要有人传一个不是 `RELATES_TO` 的类型，边会被正常 MERGE 进去，`health_check` 里那句 `MATCH ()-[r]->() RETURN count(r)` 照样把它算上，页面上的边数是涨的；但 `query_company_info` 返回的 `concepts` 是空数组。写入成功、统计正确、查询为空，三件事同时成立。

顺带一提，这段写入代码上面的注释写的是「使用APOC风格的关系类型拼接，避免Cypher注入风险」，而它做的事恰好是把外部字符串用 f-string 拼进 Cypher——注释和代码说的是反的。

## 二、主行业由关键词表的排列顺序决定

`rag_service._sync_to_knowledge_graph` 里的行业判定，全部是包含判断，只看正文前 3000 字：

```python
text_lower = text[:3000]
detected_industries = []
for industry, keywords in industry_keywords.items():
    if any(kw in text_lower for kw in keywords):
        detected_industries.append(industry)
...
kg_service.add_company(
    sec_code=sec_code, name=company_name,
    industry=detected_industries[0] if detected_industries else None,
    market=market
)
```

15 类行业、8 类概念，键分别是「银行、白酒、保险、证券、新能源、人工智能、房地产、汽车、医药、通信、电子、化工、钢铁、煤炭、电力」和「蓝筹股、国企改革、数字经济、ESG、高分红、科技创新、消费升级、碳中和」。

一份汽车研报里出现「银行贷款」「车险」「芯片」是常态，15 类里命中四五类不奇怪。而主行业取的是 `detected_industries[0]`——也就是这张 dict 里**最先被命中的那个**，而「银行」排在第一。决定一家公司归属哪个行业的，不是它的财报，是这张关键词表的插入顺序。

这个标签一旦写进图，后面所有「同行业」查询都建立在它之上，而且错得毫无声音。

## 三、降级和「成功」长得一模一样

`docker-compose.yml` 里有两处密码：neo4j 容器的 `NEO4J_AUTH: neo4j/Zhirongxi@2026`，backend 容器的 `NEO4J_PASSWORD=CHANGE_ME`。仓库里没有 `.env` 去覆盖后者。按 compose 原样起，`verify_connectivity()` 会认证失败，except 里打一行 warning，`self._driver = None`，然后一切归零。

问题在于归零之后对外还是这个表情：

```python
result = {
    "sec_code": sec_code,
    "company": info,
    "graph": triple,
    "status": "ok"
}
```

`"status": "ok"` 是硬编码的，跟 service 有没有降级没关系。Neo4j 完全没连上时，这个接口依然返回 200、`status: ok`、三个空数组。

最能说明问题的是种子脚本自己那句 docstring——它存在的理由是「解决『查询图谱只显示一个圆点』的问题」。一个圆点，就是公司节点在、关系一条没有。这句注释比任何监控指标都诚实。

## 四、那它到底该解决什么

图数据库在这里是有位置的。「谁和谁有关」这个问题向量检索答不了：向量只能告诉你哪段话像，答不出「同行业还有谁」「这条概念还挂在哪些公司上」。这种查询在 MySQL 里要三张表 join 再 distinct，写一遍就不想写第二遍。

有位置不等于用上了。它现在的位置是「一个能演示的页面」。要真留下它，三件事得按顺序做：

1. **行业标签换真源。** 从证券代码对应的行业分类字段取，不从正文关键词猜。猜出来的标签进了图，比不写更麻烦——它会以一个结构化事实的样子被下游信任。
2. **给读侧一个调用点。** 研报的同行对比节点现在拿 `sec_code` 直接查工具，把 `related_companies` 接进去，「同行业」这一栏才算有来源。接不上，就在文档里老实写「图谱展示页」，别写成「知识推理能力」。
3. **关系类型收敛成一个常量。** 读写两端引用同一个，别再一边能传参、一边写死。

## 五、一个可以自查的动作

`backend/tests/` 下 7 个测试文件，没有一个碰过 knowledge_graph。这不是疏忽，是因为「只写」的模块天生不会被测试逼出问题——写进去就算成功，没有任何断言需要读回来。

所以下次评审一个「已接入」的模块，别问它有没有接口、有没有页面、健康检查是不是绿的。只问一句：**除了它自己的展示页，还有谁在读它？** 答不上来的那个模块，无论画布上画得多好看，都还停在 demo 状态。
