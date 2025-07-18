# 技术白皮书

> **文档状态**：草稿
>
> **摘要**：本文档详细阐述了“IP智慧解答专家系统”的设计理念、核心技术架构与创新实践。系统巧妙地将智能文档处理（IDP）、混合检索与大模型Agent能力相结合，构建了一个能深度理解网络运维知识、支持多轮对话式诊断，并具备知识自进化能力的专家级问答系统。

---

## 1. 项目概述

### 1.1 问题背景
- 传统网络运维的痛点：故障定位难、知识检索效率低、专家经验缺失。
- RAG技术的机遇与挑战：如何构建高质量的知识库是RAG成功的关键。

### 1.2 系统目标
- 构建一个“提问即解决”的、面向网络运维领域的智能问答专家系统。
- 核心能力：精准的问题诊断、可执行的方案生成、持续的知识自学习。

### 1.3 创新亮点
- **知识处理**：引入企业级IDP流程，实现从文档到LLM友好知识的自动化、高质量转化。
- **智能交互**：基于`langgraph`构建多轮对话Agent，模拟专家进行主动追问。
- **答案质量**：通过混合检索、引用溯源和跨厂商方案生成，确保答案的精准、可信和实用。

## 2. 系统总体架构

### 2.1 架构图
- （此处嵌入系统总体架构图，体现混合式（本地+云端）数据流转）

### 2.2 核心模块
- **前端交互层**：负责用户交互、诊断流程可视化、数据看板展示。
- **后端服务层**：基于 `Flask` 在本地运行，负责业务逻辑处理、API网关、用户与案例管理，并通过后台任务队列处理异步流程。
- **基础设施层 (本地)**: 基于 `Docker` 运行 `MySQL` 和 `Weaviate`，并使用本地文件系统进行存储，为项目提供稳定可靠的本地数据支持。
- **AI能力层 (云端)**: 封装所有与AI相关的核心能力，包括阿里云的文档智能、PAI-EAS模型服务和百炼大模型。

## 3. 核心技术详解

### 3.1 文档解析与语义切分 (IDP)
- **挑战**：扫描件解析难、表格提取易出错、常规切块丢失语义。
- **我们的方案**：
    - **细粒度混合解析**：利用阿里云文档智能，融合电子解析与OCR/NLP，提升解析准确率。
    - **基于版面分析的语义切分**：采用GeoLayoutLM等模型，构建文档层级树，实现真正意义上的语义切分。
    - **LLM友好的结构化输出**：生成包含丰富元数据（层级、类型、页码）的Markdown，提升大模型理解效率。
- **效果**：从源头保证了知识库的高质量和高可用性。

### 3.2 混合检索与重排序
- **策略**：采用“召回+精排”的两阶段范式。首先，并行执行BM25关键词检索与向量语义检索，保证召回的全面性；然后，通过Langchain统一集成的`Qwen3-Reranker`精排模型对召回的候选文档进行二次排序，确保最终提交给大模型的上下文质量最高。
- **技术实现**：所有AI服务（云端向量模型、大模型和本地重排序模型）均通过Langchain框架统一集成，提供一致的API接口和错误处理机制。
- **可解释性**：支持用户动态调整权重，并高亮显示检索命中的原文片段。

### 3.3 基于`langgraph`的多轮对话Agent
- **设计理念**：将复杂的诊断流程建模为状态机。
- **核心节点**：分析节点、追问节点、检索节点、生成节点。
- **实现效果**：赋予系统主动提问、澄清意图、引导用户解决问题的能力。

## 4. 关键功能实现

### 4.1 知识自进化闭环
- **流程**：用户反馈 -> 人工审核 -> 知识入库 -> 自动处理 -> 服务新查询。
- **价值**：使系统能够从与用户的真实交互中持续学习和成长。

### 4.2 跨厂商解决方案生成
- **实现**：通过元数据过滤和Prompt指令注入，为不同厂商（华为、思科等）生成对应的命令语法。

### 4.3 可视化与可观测性
- **诊断路径图**：实时可视化后端Agent的工作流。
- **数据看板**：从多维度统计分析系统的运行状态和知识覆盖情况。
- **LangSmith集成**：提供对RAG流程的端到端可观测性，便于调试与优化。

## 5. 总结与展望

### 5.1 项目总结
- （总结项目完成度、达成的核心目标及取得的关键成果）

### 5.2 未来展望
- **模型微调**：微调小模型用于问题意图分类或答案校验。
- **Agent能力增强**：探索模型自主调用工具（如执行ping、traceroute命令）的能力。
- **知识图谱构建**：从文档中提取实体和关系，构建网络运维知识图谱。 