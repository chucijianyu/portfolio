# 作品集展示 Portfolio

个人项目与实习期间参与项目的展示仓库：**源码均存放在私有仓库**（含公司内部代码，不公开），此处只展示项目介绍与脱敏后的效果图/设计产出。

---

## 一、智融析 · 金融智能研报分析与投资决策辅助系统

> 源码仓库（私有）：`zhirongxi`

面向金融研报场景的 AI 全栈应用系统：研报上传解析、智能问答、一键研报生成与合规审核，覆盖数据采集到报告产出完整闭环。技术栈：FastAPI + React 18 + MySQL + Redis + Neo4j + ChromaDB + LangGraph + Ollama + Docker Compose。

### 系统截图

![01_登录页](assets/screenshots/zhirongxi/01_%E7%99%BB%E5%BD%95%E9%A1%B5.png)

**01_登录页**：登录页：RAG 智能解读 / ML 量化分析 / 合规自动审核三大能力 + 三角色 RBAC 差异化菜单

![02_文档解析分块](assets/screenshots/zhirongxi/02_%E6%96%87%E6%A1%A3%E8%A7%A3%E6%9E%90%E5%88%86%E5%9D%97.png)

**02_文档解析分块**：财报 PDF 解析为语义分块（bge-large-zh 嵌入，含 tokens 与原文溯源）

![03_关联图谱](assets/screenshots/zhirongxi/03_%E5%85%B3%E8%81%94%E5%9B%BE%E8%B0%B1.png)

**03_关联图谱**：Neo4j 知识图谱：公司-行业-概念关系可视化（归属/关联/同行业）

![04_监控告警与链路追踪](assets/screenshots/zhirongxi/04_%E7%9B%91%E6%8E%A7%E5%91%8A%E8%AD%A6%E4%B8%8E%E9%93%BE%E8%B7%AF%E8%BF%BD%E8%B8%AA.png)

**04_监控告警与链路追踪**：告警面板（RAG 延迟/LLM 错误率/合规违规）+ LangSmith 全链路 Trace

![05_系统首页仪表盘](assets/screenshots/zhirongxi/05_%E7%B3%BB%E7%BB%9F%E9%A6%96%E9%A1%B5%E4%BB%AA%E8%A1%A8%E7%9B%98.png)

**05_系统首页仪表盘**：首页：知识库文档/解读查询/分析任务/风险预警 + 使用趋势 + 任务分布

![06_系统监控与架构](assets/screenshots/zhirongxi/06_%E7%B3%BB%E7%BB%9F%E7%9B%91%E6%8E%A7%E4%B8%8E%E6%9E%B6%E6%9E%84.png)

**06_系统监控与架构**：四引擎架构：RAG(LlamaIndex+Chroma)/ML(sklearn+NumPy)/LLM(Qwen2.5+Ollama)/Agent(LangGraph)

![07_评估指标看板](assets/screenshots/zhirongxi/07_%E8%AF%84%E4%BC%B0%E6%8C%87%E6%A0%87%E7%9C%8B%E6%9D%BF.png)

**07_评估指标看板**：16 项质量指标：问答准确率 82.3%、幻觉率 0%、来源覆盖 100%、工具调用成功率 85.3%

![08_研报解读问答](assets/screenshots/zhirongxi/08_%E7%A0%94%E6%8A%A5%E8%A7%A3%E8%AF%BB%E9%97%AE%E7%AD%94.png)

**08_研报解读问答**：智能问答：核心回答+详细分析+风险提示，置信度自评估并触发补充检索

![09_综合分析ML量化](assets/screenshots/zhirongxi/09_%E7%BB%BC%E5%90%88%E5%88%86%E6%9E%90ML%E9%87%8F%E5%8C%96.png)

**09_综合分析ML量化**：ML 量化分析：趋势预测 R²、风险分级、聚类分群 + AI 五维专业解读

![10_研报生成](assets/screenshots/zhirongxi/10_%E7%A0%94%E6%8A%A5%E7%94%9F%E6%88%90.png)

**10_研报生成**：LangGraph 四角色流水线自动生成结构化研报（投资要点/公司概况/行业分析/财务分析）

![11_财务分析](assets/screenshots/zhirongxi/11_%E8%B4%A2%E5%8A%A1%E5%88%86%E6%9E%90.png)

**11_财务分析**：财务与估值：ROE/PE/PB/EPS 解读 + 三表解读 + 估值分析 + 风险提示


---

## 二、硅流 · AI 辅助硬件设计多智能体平台

> 源码仓库（私有）：`silicon-flow`
> 实习期间参与的公司内部项目，应保密要求不公开源码；以下截图为脱敏后的产品界面，账号信息已隐去。

一句话需求 → 6 个 Agent 串行协作（规格解析 → 器件选型 → 原理图生成 → PCB 布局 → 多模态校准 → 产物生成），产出可直接使用的 EDA 工程包：KiCad 工程、BOM、钻孔与 Gerber 打样包、方案文档与主控源码。

技术栈：FastAPI + Vue 3 + SQLite（字段兼容 MySQL 便于迁移）+ Milvus / 本地 JSON 双通道向量库 + bge-large-zh 向量召回 + BM25 + bge-reranker-v2-m3 精排 + Ollama（qwen2.5 / minicpm-v4.6）+ Docker。

### 平台界面

![01 多智能体流水线](assets/screenshots/silicon-flow/01-task-pipeline.png)

**01 多智能体流水线**：输入一句自然语言需求（四旋翼无人机飞控板：STM32F405 + MPU6050 + BMP280，4 路 PWM 电调，2S–6S 供电），6 个 Agent 依次执行并实时回显日志，单任务全程 47s。

![02 设计产物](assets/screenshots/silicon-flow/02-task-outputs.png)

**02 设计产物落盘**：单次任务输出 17 个文件——KiCad 工程（`board.kicad_pcb`）、Gerber 打包（`siliconflow_pcb_gerber.zip`）、钻孔文件、BOM（csv + md）、主控源码与方案文档，前端可直接下载。

![03 BOM 物料表](assets/screenshots/silicon-flow/03-bom.png)

**03 BOM 物料表**：器件型号全部来自库内真实元件（STM32F405 / MPU6050 / BMP280），带封装与位号，可直接核对采购清单——这也是用 RAG 检索替代模型「记忆」型号的原因。

![04 PCB 布局参考](assets/screenshots/silicon-flow/04-pcb-layout.png)

**04 PCB 布局参考**：按板框功能区自动分区布线（图为蓝牙耳机形态：34 元件、19 条信号走线、电源网络铺铜连接）。

![05 形态库](assets/screenshots/silicon-flow/05-form-library.png)

**05 形态库 · 异形板**：把真实产品外形建模成板框几何约束——四旋翼 X 形四臂 160×160（11 个功能区）、智能温控风扇等。通用布局只会产出矩形板，形态库解决的是「生成的板能不能装进产品」。

![06 知识库](assets/screenshots/silicon-flow/06-knowledge-base.png)

**06 纠错型知识库**：历史失败案例（复位遗漏、位宽不匹配、阻塞赋值混用等）入库，召回后经 bge-reranker 精排再回灌给 Agent——这套 RAG 用于纠错，不是问答。

### 设计产出（可编辑 KiCad 工程）

![07 KiCad 编辑器](assets/screenshots/silicon-flow/07-kicad-editor.png)

**07 KiCad PCB 编辑器**：产物是原生可编辑的 KiCad 工程，而不是一张渲染图，可继续人工修改与打样。

![08 PCB 3D 渲染](assets/screenshots/silicon-flow/08-kicad-3d.png)

**08 PCB 3D 渲染**：STM32 最小系统板（STM32F103C8T6）3D 视图。

---

---

## 三、智法通 · 企业级智能法律顾问平台

> 源码仓库（私有）：`zhifatong`

面向企业的智能法律顾问平台：基于大模型 + RAG 检索增强，提供法律问答、合同智能审查（风险条款识别与分级）、知识库管理、数据分析看板等能力，全流程 FastAPI + Vue3 + MySQL + 向量检索部署。

### 系统截图

![01_AI助手_法律问答](assets/screenshots/zhifatong/01_AI%E5%8A%A9%E6%89%8B_%E6%B3%95%E5%BE%8B%E9%97%AE%E7%AD%94.png)

**01_AI助手_法律问答**：AI 助手：法律问答流式回答 + 法条引用 + 免责声明，结论可溯源到具体法条

![02_仪表盘总览](assets/screenshots/zhifatong/02_%E4%BB%AA%E8%A1%A8%E7%9B%98%E6%80%BB%E8%A7%88.png)

**02_仪表盘总览**：仪表盘：今日咨询 / 合同审查 / 知识库文档三项指标 + 30 天咨询趋势 + 右侧 AI 助手侧栏

![03_仪表盘_AI助手流式回答](assets/screenshots/zhifatong/03_%E4%BB%AA%E8%A1%A8%E7%9B%98_AI%E5%8A%A9%E6%89%8B%E6%B5%81%E5%BC%8F%E5%9B%9E%E7%AD%94.png)

**03_仪表盘_AI助手流式回答**：AI 助手流式回答：多轮追问，法条与案例混排引用

![04_合同审查过程](assets/screenshots/zhifatong/04_%E5%90%88%E5%90%8C%E5%AE%A1%E6%9F%A5%E8%BF%87%E7%A8%8B.png)

**04_合同审查过程**：合同审查中：解析 → 完整性 → 风险 → 合规 → 报告 五步流水线实时进度

![05_合同审查完成提示](assets/screenshots/zhifatong/05_%E5%90%88%E5%90%8C%E5%AE%A1%E6%9F%A5%E5%AE%8C%E6%88%90%E6%8F%90%E7%A4%BA.png)

**05_合同审查完成提示**：审查完成：五步全部命中并给出完成提示

![06_合同审查报告](assets/screenshots/zhifatong/06_%E5%90%88%E5%90%8C%E5%AE%A1%E6%9F%A5%E6%8A%A5%E5%91%8A.png)

**06_合同审查报告**：审查报告：综合评分 62 / 风险等级 medium / 3 条风险条款（critical·high·medium 分级 + 逐条修改建议），单份耗时 8.6s；底部为多 Agent 对抗式审查

![07_知识库文档详情](assets/screenshots/zhifatong/07_%E7%9F%A5%E8%AF%86%E5%BA%93%E6%96%87%E6%A1%A3%E8%AF%A6%E6%83%85.png)

**07_知识库文档详情**：知识库文档详情：文档类型、分块数、入库状态与标签（劳动法 / 合同 / 经济补偿）

![08_数据分析看板](assets/screenshots/zhifatong/08_%E6%95%B0%E6%8D%AE%E5%88%86%E6%9E%90%E7%9C%8B%E6%9D%BF.png)

**08_数据分析看板**：数据分析看板：审查量统计与法规覆盖

![09_用户管理](assets/screenshots/zhifatong/09_%E7%94%A8%E6%88%B7%E7%AE%A1%E7%90%86.png)

**09_用户管理**：用户管理：超级管理员 / 法务专员 / 普通用户 三角色 RBAC（账号信息已打码）

![10_系统参数配置](assets/screenshots/zhifatong/10_%E7%B3%BB%E7%BB%9F%E5%8F%82%E6%95%B0%E9%85%8D%E7%BD%AE.png)

**10_系统参数配置**：系统参数：AI 温度、最大 Token、检索文档数、分块大小可调

---
