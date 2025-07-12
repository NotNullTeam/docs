# 模型应用与调优团队任务指南

> **团队规模**: 2人
> **核心职责**: 构建并优化RAG流程、设计提示词工程、构建向量数据库、管理模型接口
> **技术栈**: 阿里云文档智能、langgraph、Weaviate、Qwen3系列模型、OLLAMA

## 📋 任务概览

### 主要任务清单
- [ ] **任务1**: 数据收集与IDP集成 (预计3-4天)
- [ ] **任务2**: 向量数据库构建与配置 (预计2-3天)
- [ ] **任务3**: 混合检索算法实现 (预计4-5天)
- [ ] **任务4**: 提示词工程与优化 (预计3-4天)
- [ ] **任务5**: langgraph Agent流程构建 (预计5-6天)
- [ ] **任务6**: 模型接口集成与测试 (预计2-3天)
- [ ] **任务7**: (进阶)小模型微调 (预计5-7天，可选)

---

## 🎯 任务1: 数据收集与IDP集成

### 1.1 数据收集准备 (第1天)

**目标**: 收集网络运维相关的知识文档，为知识库构建做准备。

**具体步骤**:

1. **创建数据目录结构**
   ```bash
   mkdir -p assets/datasets/raw/{rfc,manuals,cases,others}
   mkdir -p assets/datasets/processed
   mkdir -p assets/datasets/embeddings
   ```

2. **收集RFC文档** (重点收集以下RFC)
   - RFC 2328 (OSPFv2)
   - RFC 4271 (BGP-4)
   - RFC 3031 (MPLS架构)
   - RFC 4364 (BGP/MPLS IP VPN)
   - RFC 2547 (BGP/MPLS VPN)

   **下载方式**: 访问 https://tools.ietf.org/rfc/ 下载PDF版本

3. **收集设备手册**
   - 华为: VRP系统配置指南、故障处理手册
   - 思科: IOS配置指南、故障排除手册
   - 华三: Comware系统手册

   **获取方式**: 从各厂商官网技术文档区下载

4. **收集案例文档**
   - 网络故障案例分析
   - 运维最佳实践文档
   - 技术博客和论坛精华帖

**验收标准**:
- 收集到至少50个高质量文档
- 文档按厂商和技术分类存放
- 建立文档清单Excel表格，记录文件名、来源、分类、厂商标签

### 1.2 阿里云文档智能服务配置 (第2天)

**目标**: 配置并测试阿里云文档智能服务，确保能够高质量解析各类文档。

**具体步骤**:

1. **开通服务**
   - 登录阿里云控制台
   - 搜索"文档智能"服务并开通
   - 获取AccessKey ID和AccessKey Secret
   - 记录服务endpoint地址

2. **安装SDK**
   ```bash
   pip install alibabacloud_tea_openapi alibabacloud_docmind_api20220711
   ```

3. **编写测试脚本** (`test_idp.py`)
   ```python
   from alibabacloud_docmind_api20220711.client import Client
   from alibabacloud_tea_openapi import models as open_api_models
   import base64
   import os

   def test_document_parsing():
       # 配置客户端
       config = open_api_models.Config(
           access_key_id='your_access_key_id',
           access_key_secret='your_access_key_secret',
           endpoint='docmind-api.cn-hangzhou.aliyuncs.com'
       )
       client = Client(config)

       # 测试文档解析
       try:
           from alibabacloud_docmind_api20220711 import models as docmind_models

           # 读取测试文件
           test_file = "test_document.pdf"
           with open(test_file, 'rb') as f:
               file_content = base64.b64encode(f.read()).decode('utf-8')

           # 提交解析任务
           request = docmind_models.SubmitDocumentExtractJobRequest(
               file_name=test_file,
               file_content=file_content
           )

           response = client.submit_document_extract_job(request)
           print(f"任务提交成功，Job ID: {response.body.data.id}")

       except Exception as e:
           print(f"测试失败: {e}")
   ```

4. **测试不同文档类型**
   - 测试PDF文档解析
   - 测试扫描件OCR
   - 测试表格提取
   - 测试图文混排文档

**验收标准**:
- IDP服务配置成功，能正常调用API
- 测试脚本能成功解析至少3种不同类型的文档
- 解析结果包含文本内容、表格数据、版面信息

### 1.3 语义切分算法实现 (第3-4天)

**目标**: 基于IDP返回的结构化数据，实现语义感知的文本切分。

**具体步骤**:

1. **分析IDP返回数据结构**
   - 研究文档层级树结构
   - 理解版面分析结果
   - 识别不同元素类型(标题、段落、表格、列表等)

2. **实现语义切分器** (`semantic_splitter.py`)
   ```python
   class SemanticSplitter:
       def __init__(self, max_chunk_size=1000, overlap=100):
           self.max_chunk_size = max_chunk_size
           self.overlap = overlap

       def split_by_structure(self, idp_result):
           """基于文档结构进行语义切分"""
           chunks = []
           # 实现基于层级树的切分逻辑
           return chunks
   ```

3. **实现元数据提取**
   - 提取章节标题作为上下文
   - 标记元素类型(正文/表格/代码)
   - 记录页码和位置信息
   - 自动识别厂商标签

**验收标准**:
- 切分后的文本块语义完整
- 每个文本块包含丰富的元数据
- 表格和代码块不被切断
- 生成的Markdown格式规范

---

## 🎯 任务2: 向量数据库构建与配置

### 2.1 Weaviate本地部署 (第1天)

**目标**: 使用Docker在本地部署Weaviate向量数据库。

**具体步骤**:

1. **创建Docker Compose配置** (`docker-compose.local.yml`)
   ```yaml
   version: '3.8'
   services:
     weaviate:
       image: semitechnologies/weaviate:1.21.2
       ports:
         - "8080:8080"
         - "50051:50051"
       environment:
         QUERY_DEFAULTS_LIMIT: 25
         AUTHENTICATION_ANONYMOUS_ACCESS_ENABLED: 'true'
         PERSISTENCE_DATA_PATH: '/var/lib/weaviate'
         DEFAULT_VECTORIZER_MODULE: 'none'
         ENABLE_MODULES: 'text2vec-openai,generative-openai'
         CLUSTER_HOSTNAME: 'node1'
       volumes:
         - weaviate_data:/var/lib/weaviate

   volumes:
     weaviate_data:
   ```

2. **启动服务**
   ```bash
   docker-compose -f docker-compose.local.yml up -d
   ```

3. **验证部署**
   ```bash
   curl http://localhost:8080/v1/meta
   ```

**验收标准**:
- Weaviate服务正常启动
- 能通过REST API访问
- 数据持久化配置正确

### 2.2 Schema设计与创建 (第1天)

**目标**: 设计适合网络运维知识的数据Schema。

**具体步骤**:

1. **设计Schema结构** (`schema_design.py`)
   ```python
   KNOWLEDGE_SCHEMA = {
       "class": "KnowledgeChunk",
       "description": "网络运维知识片段",
       "properties": [
           {
               "name": "content",
               "dataType": ["text"],
               "description": "文本内容"
           },
           {
               "name": "title",
               "dataType": ["string"],
               "description": "标题或摘要"
           },
           {
               "name": "vendor",
               "dataType": ["string"],
               "description": "设备厂商"
           },
           {
               "name": "category",
               "dataType": ["string"],
               "description": "技术分类"
           },
           {
               "name": "source_document",
               "dataType": ["string"],
               "description": "源文档名称"
           },
           {
               "name": "page_number",
               "dataType": ["int"],
               "description": "页码"
           }
       ],
       "vectorizer": "none"
   }
   ```

2. **创建Schema**
   ```python
   import weaviate

   client = weaviate.Client("http://localhost:8080")
   client.schema.create_class(KNOWLEDGE_SCHEMA)
   ```

**验收标准**:
- Schema创建成功
- 属性定义完整合理
- 支持元数据过滤查询

### 2.3 向量化模型集成 (第2天)

**目标**: 集成阿里云百炼平台的text-embedding-v4模型。

**具体步骤**:

1. **配置百炼API**
   ```python
   from dashscope import TextEmbedding
   import os

   class QwenEmbedding:
       def __init__(self, api_key=None):
           self.api_key = api_key or os.environ.get('DASHSCOPE_API_KEY')

       def embed_text(self, text):
           response = TextEmbedding.call(
               model='text-embedding-v4',
               input=text,
               api_key=self.api_key
           )
           return response.output['embeddings'][0]['embedding']
   ```

2. **批量向量化工具**
   ```python
   def batch_embed_documents(documents, batch_size=10):
       embeddings = []
       for i in range(0, len(documents), batch_size):
           batch = documents[i:i+batch_size]
           # 批量调用embedding API
           batch_embeddings = embed_batch(batch)
           embeddings.extend(batch_embeddings)
       return embeddings
   ```

**验收标准**:
- 能成功调用text-embedding-v4 API
- 支持批量向量化处理
- 向量维度正确(通常1536维)

---

## 🎯 任务3: 混合检索算法实现

### 3.1 BM25关键词检索 (第1天)

**目标**: 实现基于BM25算法的关键词检索。

**具体步骤**:

1. **配置Weaviate BM25**
   ```python
   def bm25_search(client, query, limit=50):
       result = client.query.get("KnowledgeChunk", [
           "content", "title", "vendor", "category"
       ]).with_bm25(
           query=query
       ).with_limit(limit).do()

       return result
   ```

2. **关键词预处理**
   ```python
   def preprocess_query(query):
       # 网络术语标准化
       # 同义词扩展
       # 停用词过滤
       return processed_query
   ```

**验收标准**:
- BM25检索功能正常
- 能检索到相关文档
- 支持中英文混合查询

### 3.2 向量语义检索 (第2天)

**目标**: 实现基于向量相似度的语义检索。

**具体步骤**:

1. **向量检索实现**
   ```python
   def vector_search(client, query_vector, limit=50):
       result = client.query.get("KnowledgeChunk", [
           "content", "title", "vendor", "category"
       ]).with_near_vector({
           "vector": query_vector
       }).with_limit(limit).do()

       return result
   ```

2. **相似度计算优化**
   - 实现余弦相似度计算
   - 设置相似度阈值
   - 处理向量归一化

**验收标准**:
- 向量检索功能正常
- 相似度计算准确
- 检索速度满足要求(< 500ms)

### 3.3 混合检索与重排序 (第3-4天)

**目标**: 实现两阶段检索：召回+精排。

**具体步骤**:

1. **混合召回策略**
   ```python
   def hybrid_recall(query, bm25_weight=0.3, vector_weight=0.7):
       # 关键词召回
       bm25_results = bm25_search(query, limit=50)

       # 向量召回
       query_vector = embed_text(query)
       vector_results = vector_search(query_vector, limit=50)

       # 结果合并去重
       combined_results = merge_and_deduplicate(
           bm25_results, vector_results,
           bm25_weight, vector_weight
       )

       return combined_results
   ```

2. **重排序模型集成**
   ```python
   # 使用OLLAMA部署Qwen3-Reranker
   def setup_reranker():
       # 下载并启动OLLAMA
       # 拉取Qwen3-Reranker模型
       pass

   def rerank_documents(query, documents, top_k=5):
       # 调用重排序模型
       scores = []
       for doc in documents:
           score = call_reranker_api(query, doc['content'])
           scores.append(score)

       # 按分数排序并返回top_k
       ranked_docs = sort_by_score(documents, scores)[:top_k]
       return ranked_docs
   ```

**验收标准**:
- 混合检索召回率高(>80%)
- 重排序模型部署成功
- 最终检索精度明显提升

---

## 🎯 任务4: 提示词工程与优化

### 4.1 基础Prompt设计 (第1天)

**目标**: 设计用于不同场景的基础提示词模板。

**具体步骤**:

1. **分析节点Prompt** (`prompts/analysis_prompt.py`)
   ```python
   ANALYSIS_PROMPT = """
   你是一位资深的网络运维专家。请分析用户提出的网络问题。

   用户问题: {user_query}

   上下文信息:
   {context}

   请按以下格式分析:
   1. 问题分类: [OSPF/BGP/MPLS/VPN/其他]
   2. 可能原因: 列出3-5个可能的原因
   3. 需要补充的信息: 如果信息不足，列出需要用户提供的具体信息
   4. 初步建议: 给出初步的排查方向

   如果信息不足以给出完整分析，请在回答末尾添加: [NEED_MORE_INFO]
   """
   ```

2. **追问节点Prompt**
   ```python
   CLARIFICATION_PROMPT = """
   基于当前的分析，我需要更多信息来帮助您解决问题。

   当前分析: {current_analysis}

   请回答以下问题:
   {questions}

   您也可以上传相关的配置文件、日志文件或网络拓扑图。
   """
   ```

3. **解决方案生成Prompt**
   ```python
   SOLUTION_PROMPT = """
   你是一位{vendor}网络设备专家。基于以下信息，提供详细的解决方案。

   问题描述: {problem}
   相关文档: {retrieved_docs}
   用户补充信息: {user_context}

   请提供:
   1. 问题根因分析
   2. 详细解决步骤 (每步都要具体可执行)
   3. 验证方法
   4. 预防措施

   在引用文档内容时，请使用 [doc1], [doc2] 等标记。
   所有命令都要使用{vendor}的语法格式。
   """
   ```

**验收标准**:
- Prompt模板覆盖所有节点类型
- 模板支持变量替换
- 输出格式规范统一

### 4.2 厂商适配Prompt (第2天)

**目标**: 为不同厂商设备定制专门的提示词。

**具体步骤**:

1. **华为VRP系统Prompt**
   ```python
   HUAWEI_PROMPT = """
   你是华为VRP系统专家。请使用华为设备的命令语法回答问题。

   常用命令格式:
   - 查看接口: display interface
   - 查看路由: display ip routing-table
   - 查看OSPF: display ospf peer
   - 配置接口: interface GigabitEthernet0/0/1

   {base_prompt}

   注意: 所有命令必须符合VRP语法规范。
   """
   ```

2. **思科IOS系统Prompt**
   ```python
   CISCO_PROMPT = """
   你是思科IOS系统专家。请使用思科设备的命令语法回答问题。

   常用命令格式:
   - 查看接口: show interfaces
   - 查看路由: show ip route
   - 查看OSPF: show ip ospf neighbor
   - 配置接口: interface GigabitEthernet0/0

   {base_prompt}

   注意: 所有命令必须符合IOS语法规范。
   """
   ```

**验收标准**:
- 支持主流厂商(华为、思科、华三)
- 命令语法准确无误
- 能根据上下文自动选择厂商

### 4.3 引用标记与溯源 (第3天)

**目标**: 实现答案的引用标记和知识溯源功能。

**具体步骤**:

1. **引用标记Prompt**
   ```python
   CITATION_PROMPT = """
   在生成答案时，必须为每个引用的信息添加标记。

   引用格式: [doc{序号}]

   示例:
   "根据RFC 2328规定，OSPF邻居关系建立需要经过Down、Init、2-Way、ExStart、Exchange、Loading、Full七个状态 [doc1]。
   如果卡在ExStart状态，通常是MTU不匹配导致的 [doc2]。"

   引用的文档信息:
   {source_documents}

   请严格按照此格式添加引用标记。
   """
   ```

2. **溯源信息提取**
   ```python
   def extract_citations(answer_text, source_docs):
       citations = []
       # 提取[doc1], [doc2]等标记
       # 匹配对应的源文档信息
       return citations
   ```

**验收标准**:
- 答案中正确添加引用标记
- 引用信息准确对应源文档
- 支持点击查看原文

### 4.4 Prompt效果测试与优化 (第4天)

**目标**: 测试和优化提示词效果。

**具体步骤**:

1. **构建测试用例**
   ```python
   TEST_CASES = [
       {
           "query": "OSPF邻居状态卡在ExStart怎么办？",
           "expected_keywords": ["MTU", "接口", "配置"],
           "expected_vendor": "通用"
       },
       {
           "query": "华为设备BGP邻居建立失败",
           "expected_keywords": ["peer", "AS号", "路由器ID"],
           "expected_vendor": "华为"
       }
   ]
   ```

2. **自动化测试脚本**
   ```python
   def test_prompt_quality(test_cases):
       results = []
       for case in test_cases:
           response = generate_answer(case['query'])
           score = evaluate_response(response, case)
           results.append(score)
       return results
   ```

**验收标准**:
- 测试用例覆盖主要场景
- 答案质量评分>80%
- Prompt迭代优化完成

---

## 🎯 任务5: langgraph Agent流程构建

### 5.1 状态机设计 (第1天)

**目标**: 设计多轮对话的状态机架构。

**具体步骤**:

1. **定义状态结构**
   ```python
   from typing import TypedDict, List

   class AgentState(TypedDict):
       messages: List[dict]  # 对话历史
       context: List[dict]   # 检索到的知识
       user_query: str       # 用户问题
       vendor: str          # 设备厂商
       category: str        # 问题分类
       need_more_info: bool # 是否需要更多信息
       solution_ready: bool # 是否可以给出解决方案
   ```

2. **设计节点类型**
   ```python
   # 分析节点
   def analyze_query(state: AgentState) -> AgentState:
       # 分析用户问题
       # 分类问题类型
       # 识别厂商信息
       pass

   # 检索节点
   def retrieve_knowledge(state: AgentState) -> AgentState:
       # 执行混合检索
       # 更新context
       pass

   # 追问节点
   def ask_clarification(state: AgentState) -> AgentState:
       # 生成追问问题
       # 等待用户回答
       pass

   # 解决方案节点
   def generate_solution(state: AgentState) -> AgentState:
       # 生成最终解决方案
       # 添加引用标记
       pass
   ```

**验收标准**:
- 状态结构设计合理
- 节点功能划分清晰
- 支持状态传递和更新

### 5.2 工作流编排 (第2-3天)

**目标**: 使用langgraph构建完整的对话流程。

**具体步骤**:

1. **创建状态图**
   ```python
   from langgraph.graph import StateGraph, END

   def create_agent_graph():
       workflow = StateGraph(AgentState)

       # 添加节点
       workflow.add_node("analyze", analyze_query)
       workflow.add_node("retrieve", retrieve_knowledge)
       workflow.add_node("clarify", ask_clarification)
       workflow.add_node("solve", generate_solution)

       # 定义边和条件
       workflow.set_entry_point("analyze")

       workflow.add_conditional_edges(
           "analyze",
           should_retrieve,
           {
               "retrieve": "retrieve",
               "clarify": "clarify"
           }
       )

       workflow.add_conditional_edges(
           "retrieve",
           should_solve_or_clarify,
           {
               "solve": "solve",
               "clarify": "clarify"
           }
       )

       workflow.add_edge("clarify", "analyze")
       workflow.add_edge("solve", END)

       return workflow.compile()
   ```

2. **实现条件判断函数**
   ```python
   def should_retrieve(state: AgentState) -> str:
       if state.get("category") and not state.get("need_more_info"):
           return "retrieve"
       return "clarify"

   def should_solve_or_clarify(state: AgentState) -> str:
       if len(state["context"]) > 0 and not state.get("need_more_info"):
           return "solve"
       return "clarify"
   ```

**验收标准**:
- 工作流逻辑正确
- 条件判断准确
- 支持多轮对话

### 5.3 Agent测试与调试 (第4-5天)

**目标**: 测试Agent的完整对话流程。

**具体步骤**:

1. **单元测试**
   ```python
   def test_analyze_node():
       state = {"user_query": "OSPF邻居建立失败"}
       result = analyze_query(state)
       assert result["category"] == "OSPF"

   def test_retrieve_node():
       state = {"user_query": "BGP配置", "category": "BGP"}
       result = retrieve_knowledge(state)
       assert len(result["context"]) > 0
   ```

2. **集成测试**
   ```python
   def test_full_conversation():
       agent = create_agent_graph()

       # 测试完整对话流程
       initial_state = {
           "user_query": "华为设备OSPF邻居状态异常",
           "messages": []
       }

       final_state = agent.invoke(initial_state)
       assert final_state["solution_ready"] == True
   ```

**验收标准**:
- 所有节点单元测试通过
- 完整对话流程测试通过
- Agent能正确处理各种场景

---

## 🎯 任务6: 模型接口集成与测试

### 6.1 百炼大模型API集成 (第1天)

**目标**: 集成阿里云百炼平台的qwen-plus模型。

**具体步骤**:

1. **配置API客户端**
   ```python
   from dashscope import Generation
   from langchain_openai import ChatOpenAI
   import os

   class QwenLLM:
       def __init__(self, api_key=None, model="qwen-plus"):
           self.api_key = api_key or os.environ.get('DASHSCOPE_API_KEY')
           self.model = model

       def generate(self, prompt, max_tokens=2000):
           response = Generation.call(
               model=self.model,
               prompt=prompt,
               api_key=self.api_key,
               max_tokens=max_tokens,
               temperature=0.1
           )
           return response.output.text

   # 使用langchain_openai的方式（推荐）
   class QwenLangChain:
       def __init__(self, api_key=None, model="qwen-plus"):
           self.llm = ChatOpenAI(
               model=model,
               openai_api_key=api_key or os.environ.get('DASHSCOPE_API_KEY'),
               openai_api_base="https://dashscope.aliyuncs.com/compatible-mode/v1",
               temperature=0.1
           )

       def generate(self, messages):
           response = self.llm.invoke(messages)
           return response.content
   ```

2. **实现流式输出**
   ```python
   def stream_generate(self, prompt):
       responses = Generation.call(
           model=self.model,
           prompt=prompt,
           api_key=self.api_key,
           stream=True
       )

       for response in responses:
           yield response.output.text
   ```

**验收标准**:
- API调用成功
- 支持流式和非流式输出
- 错误处理完善

### 6.2 性能优化与监控 (第2天)

**目标**: 优化模型调用性能，添加监控。

**具体步骤**:

1. **添加缓存机制**
   ```python
   import redis
   import hashlib

   class CachedLLM:
       def __init__(self, llm, redis_client):
           self.llm = llm
           self.redis = redis_client

       def generate_with_cache(self, prompt):
           # 计算prompt hash
           prompt_hash = hashlib.md5(prompt.encode()).hexdigest()

           # 检查缓存
           cached = self.redis.get(prompt_hash)
           if cached:
               return cached.decode()

           # 调用模型
           result = self.llm.generate(prompt)

           # 存入缓存
           self.redis.setex(prompt_hash, 3600, result)
           return result
   ```

2. **添加性能监控**
   ```python
   import time
   import logging

   def monitor_llm_call(func):
       def wrapper(*args, **kwargs):
           start_time = time.time()
           try:
               result = func(*args, **kwargs)
               duration = time.time() - start_time
               logging.info(f"LLM call success: {duration:.2f}s")
               return result
           except Exception as e:
               duration = time.time() - start_time
               logging.error(f"LLM call failed: {duration:.2f}s, {e}")
               raise
       return wrapper
   ```

**验收标准**:
- 缓存机制工作正常
- 性能监控数据准确
- 平均响应时间<3秒

---

## 🎯 任务7: (进阶)小模型微调

### 7.1 数据准备 (第1-2天)

**目标**: 准备用于微调的训练数据集。

**具体步骤**:

1. **收集问题分类数据**
   ```python
   # 构建问题分类数据集
   classification_data = [
       {"text": "OSPF邻居建立失败", "label": "OSPF"},
       {"text": "BGP路由不通", "label": "BGP"},
       {"text": "MPLS VPN配置问题", "label": "MPLS"},
       # ... 更多数据
   ]
   ```

2. **数据清洗和标注**
   - 去重和格式统一
   - 人工标注和校验
   - 数据集划分(训练/验证/测试)

**验收标准**:
- 收集到1000+标注样本
- 数据质量高，标注准确
- 数据集划分合理

### 7.2 模型微调 (第3-5天)

**目标**: 微调BERT模型用于问题意图分类。

**具体步骤**:

1. **选择基础模型**
   ```python
   from transformers import AutoTokenizer, AutoModelForSequenceClassification

   model_name = "bert-base-chinese"
   tokenizer = AutoTokenizer.from_pretrained(model_name)
   model = AutoModelForSequenceClassification.from_pretrained(
       model_name,
       num_labels=len(categories)
   )
   ```

2. **训练脚本**
   ```python
   from transformers import Trainer, TrainingArguments

   training_args = TrainingArguments(
       output_dir="./results",
       num_train_epochs=3,
       per_device_train_batch_size=16,
       per_device_eval_batch_size=64,
       warmup_steps=500,
       weight_decay=0.01,
       logging_dir="./logs",
   )

   trainer = Trainer(
       model=model,
       args=training_args,
       train_dataset=train_dataset,
       eval_dataset=eval_dataset,
   )

   trainer.train()
   ```

**验收标准**:
- 模型训练完成
- 验证集准确率>85%
- 模型可以正常推理

---

## 📊 进度跟踪与质量控制

### 每日站会
- **时间**: 每天上午9:30
- **内容**: 汇报昨日完成情况、今日计划、遇到的问题
- **参与人**: 模型团队2人 + 项目负责人

### 质量检查点
1. **第一周末**: 数据收集和IDP集成完成度检查
2. **第二周末**: 向量数据库和检索算法验收
3. **第三周末**: Agent流程和模型集成测试

### 交付物清单
- [ ] 处理后的知识库数据集
- [ ] Weaviate向量数据库(包含索引数据)
- [ ] 混合检索算法代码
- [ ] langgraph Agent工作流
- [ ] 提示词模板库
- [ ] 模型接口封装代码
- [ ] 技术文档和使用说明

### 风险预案
1. **IDP服务调用失败**: 准备本地OCR备选方案
2. **向量化API限流**: 实现请求队列和重试机制
3. **模型效果不佳**: 准备多个Prompt版本进行A/B测试

---

## 🔧 开发环境配置

### 必需软件
- Python 3.9+
- Docker & Docker Compose
- Git
- OLLAMA (用于本地模型部署)

### Python依赖
```bash
pip install dashscope weaviate-client langgraph langchain langchain_openai redis transformers torch alibabacloud_tea_openapi alibabacloud_docmind_api20220711
```

### 环境变量配置
```bash
export DASHSCOPE_API_KEY="your_dashscope_api_key"
export WEAVIATE_URL="http://localhost:8080"
export REDIS_URL="redis://localhost:6379"

# 用于langchain_openai访问Qwen-Plus
export OPENAI_API_KEY="your_dashscope_api_key"
export OPENAI_API_BASE="https://dashscope.aliyuncs.com/compatible-mode/v1"

# 阿里云文档智能服务
export ALIBABA_ACCESS_KEY_ID="your_access_key_id"
export ALIBABA_ACCESS_KEY_SECRET="your_access_key_secret"
```

---

## 📚 参考资料

### 技术文档
- [阿里云文档智能API文档](https://help.aliyun.com/document_detail/442515.html)
- [Weaviate官方文档](https://weaviate.io/developers/weaviate)
- [LangGraph教程](https://langchain-ai.github.io/langgraph/)
- [百炼大模型API文档](https://help.aliyun.com/zh/dashscope/)

### 学习资源
- RAG技术原理与实践
- 向量数据库使用指南
- 提示词工程最佳实践
- 多轮对话系统设计

记住：遇到问题及时沟通，保持代码质量，注重文档记录！🚀