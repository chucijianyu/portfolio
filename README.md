# 作品集展示 Portfolio

个人项目作品展示仓库：**源码均存放在私有仓库**（防止被克隆抄用），此处公开展示项目介绍与效果图/设计图。

---

---

## 一、智融析 · 金融智能研报分析与投资决策辅助系统智融析 · 金融智能研报分析与投资决策辅助系统

> 源码仓库（私有）：`zhirongxi`

面向金融研报场景的 AI 全栈应用系统：研报上传解析、智能问答、一键研报生成与合规审核，覆盖数据采集到报告产出完整闭环。技术栈：FastAPI + React 18 + MySQL + Redis + Neo4j + ChromaDB + LangGraph + Ollama + Docker Compose。

### 系统截图

![01_登录页](assets/screenshots/zhirongxi/01_%E7%99%BB%E5%BD%95%E9%A1%B5.png)

**01_登录页**：登录页：RAG 智能解读 / ML 量化分析 / 合规自动审核三大能力 + 三角色 RBAC 差异化菜单

![02_知识库管理](assets/screenshots/zhirongxi/02_%E7%9F%A5%E8%AF%86%E5%BA%93%E7%AE%A1%E7%90%86.png)

**02_知识库管理**：金融文档知识库：批量上传（PDF/Word/HTML/TXT）、自动解析嵌入、重建索引、检索测试

![03_文档解析分块](assets/screenshots/zhirongxi/03_%E6%96%87%E6%A1%A3%E8%A7%A3%E6%9E%90%E5%88%86%E5%9D%97.png)

**03_文档解析分块**：财报 PDF 解析为语义分块（bge-large-zh 嵌入，含 tokens 与原文溯源）

![04_关联图谱](assets/screenshots/zhirongxi/04_%E5%85%B3%E8%81%94%E5%9B%BE%E8%B0%B1.png)

**04_关联图谱**：Neo4j 知识图谱：公司-行业-概念关系可视化（归属/关联/同行业）

![05_监控告警与链路追踪](assets/screenshots/zhirongxi/05_%E7%9B%91%E6%8E%A7%E5%91%8A%E8%AD%A6%E4%B8%8E%E9%93%BE%E8%B7%AF%E8%BF%BD%E8%B8%AA.png)

**05_监控告警与链路追踪**：告警面板（RAG 延迟/LLM 错误率/合规违规）+ LangSmith 全链路 Trace

![06_系统首页仪表盘](assets/screenshots/zhirongxi/06_%E7%B3%BB%E7%BB%9F%E9%A6%96%E9%A1%B5%E4%BB%AA%E8%A1%A8%E7%9B%98.png)

**06_系统首页仪表盘**：首页：知识库文档/解读查询/分析任务/风险预警 + 使用趋势 + 任务分布

![07_系统监控与架构](assets/screenshots/zhirongxi/07_%E7%B3%BB%E7%BB%9F%E7%9B%91%E6%8E%A7%E4%B8%8E%E6%9E%B6%E6%9E%84.png)

**07_系统监控与架构**：四引擎架构：RAG(LlamaIndex+Chroma)/ML(sklearn+NumPy)/LLM(Qwen2.5+Ollama)/Agent(LangGraph)

![08_评估指标看板](assets/screenshots/zhirongxi/08_%E8%AF%84%E4%BC%B0%E6%8C%87%E6%A0%87%E7%9C%8B%E6%9D%BF.png)

**08_评估指标看板**：16 项质量指标：问答准确率 82.3%、幻觉率 0%、来源覆盖 100%、工具调用成功率 85.3%

![09_研报解读问答](assets/screenshots/zhirongxi/09_%E7%A0%94%E6%8A%A5%E8%A7%A3%E8%AF%BB%E9%97%AE%E7%AD%94.png)

**09_研报解读问答**：智能问答：核心回答+详细分析+风险提示，置信度自评估并触发补充检索

![10_综合分析ML量化](assets/screenshots/zhirongxi/10_%E7%BB%BC%E5%90%88%E5%88%86%E6%9E%90ML%E9%87%8F%E5%8C%96.png)

**10_综合分析ML量化**：ML 量化分析：趋势预测 R²、风险分级、聚类分群 + AI 五维专业解读

![11_研报生成](assets/screenshots/zhirongxi/11_%E7%A0%94%E6%8A%A5%E7%94%9F%E6%88%90.png)

**11_研报生成**：LangGraph 四角色流水线自动生成结构化研报（投资要点/公司概况/行业分析/财务分析）

![12_财务分析](assets/screenshots/zhirongxi/12_%E8%B4%A2%E5%8A%A1%E5%88%86%E6%9E%90.png)

**12_财务分析**：财务与估值：ROE/PE/PB/EPS 解读 + 三表解读 + 估值分析 + 风险提示

![13_风险提示与投资建议](assets/screenshots/zhirongxi/13_%E9%A3%8E%E9%99%A9%E6%8F%90%E7%A4%BA%E4%B8%8E%E6%8A%95%E8%B5%84%E5%BB%BA%E8%AE%AE.png)

**13_风险提示与投资建议**：风险维度拆解（财务/市场）+ 自动评级与目标价（强烈推荐）


---

---

## 二、硅流 · AI 辅助硬件设计多智能体平台硅流 · AI 辅助硬件设计多智能体平台

> 源码仓库（私有）：`silicon-flow`

AI 辅助硬件设计多智能体平台，覆盖从原理图生成到打板的全流程（多 Agent 流水线编排 + 三级降级保障），图为 STM32-MinSys 最小系统板（KiCad）设计产出。

### 产品图

![STM32-MinSys PCB 版图](assets/screenshots/silicon-flow/01-stm32-pcb-layout.png)
![PCB 3D 渲染](assets/screenshots/silicon-flow/02-stm32-pcb-3d.png)
![四旋翼赛道地图](assets/screenshots/silicon-flow/03-drone-track.png)

---

---

## 三、智法通 · 企业级智能法律顾问平台智法通 · 企业级智能法律顾问平台

> 源码仓库（私有）：`zhifatong`

面向企业的智能法律顾问平台：基于大模型 + RAG 检索增强，提供法律问答、合同智能审查（风险条款识别与分级）、知识库管理、数据分析看板等能力，全流程 FastAPI + Vue3 + MySQL + 向量检索部署。

### 系统截图

![1_登录页_实机](assets/screenshots/zhifatong/1_登录页_实机.png)
![2_仪表盘总览](assets/screenshots/zhifatong/2_仪表盘总览.png)
![3_仪表盘_AI助手流式回答](assets/screenshots/zhifatong/3_仪表盘_AI助手流式回答.png)
![4_合同审查过程](assets/screenshots/zhifatong/4_合同审查过程.png)
![5_合同审查完成提示](assets/screenshots/zhifatong/5_合同审查完成提示.png)
![6_合同审查报告](assets/screenshots/zhifatong/6_合同审查报告.png)
![7_知识库文档详情](assets/screenshots/zhifatong/7_知识库文档详情.png)
![8_数据分析看板](assets/screenshots/zhifatong/8_数据分析看板.png)
![9_用户管理](assets/screenshots/zhifatong/9_用户管理.png)
![10_系统设置](assets/screenshots/zhifatong/10_系统设置.png)
![11_系统参数配置](assets/screenshots/zhifatong/11_系统参数配置.png)

---
