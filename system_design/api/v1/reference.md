# API 文档 (v1)

## 基本信息

- **基础路径**: `/api/v1`
- **数据交换格式**: JSON
- **字符编码**: UTF-8

## 认证方式

所有需要认证的API请求，必须在HTTP头部包含 `Authorization` 字段。

`Authorization: Bearer {access_token}`

## 通用响应结构

### 成功响应
```json
{
  "code": 200,
  "status": "success",
  "data": { /* 响应数据 */ }
}
```

### 失败响应
```json
{
  "code": 400,
  "status": "error",
  "message": "错误详细信息"
}
```

#### 错误码说明

| HTTP 状态码 | 业务错误码(`error.type`) | 错误类型 | 说明 | 示例场景 |
| --- | --- | --- | --- | --- |
| 400 | `INVALID_REQUEST` | 参数校验失败 | 请求体缺失必需字段或字段格式不合法 | 创建案例时缺少 `query` 字段 |
| 401 | `UNAUTHORIZED` | 未认证 | 请求未携带或携带无效 Token | Token 过期或无效 |
| 403 | `FORBIDDEN` | 无权限 | 当前用户无权访问该资源 | 尝试访问他人诊断案例 |
| 404 | `NOT_FOUND` | 资源不存在 | 请求的资源不存在 | `caseId` 不存在 |
| 409 | `CONFLICT` | 资源冲突 | 资源状态冲突，无法完成请求 | 重复创建同名案例 |
| 422 | `UNPROCESSABLE_ENTITY` | 业务规则校验失败 | 请求格式正确，但不满足业务规则 | 上传文件大小超限 |
| 429 | `RATE_LIMITED` | 触发限流 | 过短时间内过多请求 | 高频调用生成解决方案接口 |
| 500 | `INTERNAL_ERROR` | 服务器错误 | 非预期后端异常 | 数据库连接失败 |
| 503 | `SERVICE_UNAVAILABLE` | 服务暂不可用 | 下游依赖服务不可用或维护中 | 向量数据库超时 |

错误响应示例（含业务错误码）：
```json
{
  "code": 404,
  "status": "error",
  "error": {
    "type": "NOT_FOUND",
    "message": "指定的 caseId 不存在"
  }
}
```

## 核心概念与数据结构

每一次独立的诊断过程视为一个“**诊断案例 (Case)**”。每个`Case`在逻辑上是一个有向图（Diagnostic Path），由多个“**节点 (Node)**”和连接它们的“**边 (Edge)**”组成。

### Node (节点)
```json
{
  "id": "node_a1b2c3d4",
  "type": "USER_QUERY" | "AI_ANALYSIS" | "AI_CLARIFICATION" | "USER_RESPONSE" | "SOLUTION",
  "title": "标题/节点摘要",
  "status": "COMPLETED" | "AWAITING_USER_INPUT" | "PROCESSING",
  "content": {
    // 当 type 为 USER_QUERY / USER_RESPONSE 时
    "text": "问题文本或用户补充信息",
    "attachments": [ { "type": "image", "url": "https://...", "name": "screenshot.png" } ],

    // 当 type 为 AI_ANALYSIS 时
    "analysis": "AI 的分析摘要",
    "suggestedQuestions": [ "下一步该检查什么？" ],

    // 当 type 为 SOLUTION 时
    "answer": "为解决OSPF邻居ExStart状态问题，您需要检查接口的MTU值 [doc1]。在华为设备上，可以使用命令 `display ip interface brief` 来查看 [doc2]。",
    "sources": [
        { "id": "doc1", "source_document_name": "HUAWEI-OSPF故障排查手册.pdf", "page_number": 25 },
        { "id": "doc2", "source_document_name": "华为VRP命令参考.pdf", "page_number": 112 }
    ],
    "commands": {
      "Huawei": [ "display ospf", "display interface" ],
      "Cisco": [ "show ip ospf", "show interfaces" ]
    }
  },
  "metadata": {
    "tags": ["OSPF", "MTU"],
    "vendor": "Huawei",
    "relevance": 0.95
  }
}
```

#### Node 字段说明

| 字段 | 类型 | 是否必填 | 取值 / 格式 | 说明 |
| ---- | ---- | -------- | ----------- | ---- |
| `id` | string | 是 | `node_<UUID>` | 节点唯一标识，由后端生成，保证在全局范围内唯一。 |
| `type` | enum | 是 | `USER_QUERY` / `AI_ANALYSIS` / `AI_CLARIFICATION` / `USER_RESPONSE` / `SOLUTION` | 节点业务角色，不同类型驱动不同的前端渲染逻辑。 |
| `title` | string | 是 | 任意 | 节点标题或摘要，用于列表及画布展示。 |
| `status` | enum | 是 | `COMPLETED` / `AWAITING_USER_INPUT` / `PROCESSING` | 节点处理状态：• `PROCESSING` 表示 AI 正在生成内容；• `AWAITING_USER_INPUT` 表示等待用户补充信息；• `COMPLETED` 表示节点已完成。 |
| `content` | object | 视 `type` 而定 | — | 节点主体内容，结构随 `type` 不同，见下表。 |
| `metadata` | object | 否 | — | 搜索与过滤所用的附加信息，如标签、厂商、相关度等。 |

> 以下子字段只在满足对应 `type` 时出现

| `type` | 子字段 | 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| `USER_QUERY` / `USER_RESPONSE` | `text` | string | 用户输入的文本内容 |
|  | `attachments` | Attachment[] | 用户上传的附件，结构见“Attachment 说明” |
| `AI_ANALYSIS` | `analysis` | string | AI 对当前信息的诊断分析摘要 |
|  | `suggestedQuestions` | string[] | AI 建议的下一步追问列表 |
| `SOLUTION` | `answer` | string | 解决方案正文，可能包含引用标记如 `[docX]` |
|  | `sources` | Source[] | 解决方案所引用的知识片段元数据，结构见“Source 说明” |
|  | `commands` | object | 各厂商对应的命令集合，键为厂商名，值为命令字符串数组 |

**Attachment 说明**

| 字段 | 类型 | 说明 |
| ---- | ---- | ---- |
| `type` | enum | `image` / `topo` / `log` / `config` / `other` |
| `url` | string | 文件的可访问 URL |
| `name` | string | 原文件名 |

**Source 说明**

| 字段 | 类型 | 说明 |
| ---- | ---- | ---- |
| `id` | string | 引用标记 ID，与 `answer` 中的 `[docX]` 对应 |
| `source_document_name` | string | 原始文档文件名 |
| `page_number` | integer | 页码（从 1 开始） |

### Edge (边)
```json
{
  "source": "node_a1b2c3d4", // 源节点ID
  "target": "node_e5f6g7h8"  // 目标节点ID
}
```

#### Edge 字段说明

| 字段 | 类型 | 是否必填 | 说明 |
| ---- | ---- | -------- | ---- |
| `source` | string | 是 | 边的起始节点 ID，应指向现有的 `Node.id` |
| `target` | string | 是 | 边的目标节点 ID，应指向现有的 `Node.id` |

---

## API接口列表

### 1. 认证接口 (Authentication)

#### 1.1 用户登录
- **Endpoint**: `POST /auth/login`
- **功能**: 用户使用凭据登录，获取`access_token`。
- **请求体**:
  ```json
  {
    "username": "admin",
    "password": "password123"
  }
  ```
- **响应体**:
  ```json
  {
    "access_token": "eyJhbGciOi...",
    "token_type": "Bearer",
    "user_info": { "id": 1, "username": "admin" }
  }
  ```

#### 1.2 用户登出
- **Endpoint**: `POST /auth/logout`
- **功能**: 使当前用户的`access_token`失效。
- **认证**: 需要
- **响应**: `204 No Content`

#### 1.3 刷新 Token
- **Endpoint**: `POST /auth/refresh`
- **功能**: 使用有效的 `refresh_token` 换取新的 `access_token`，延长会话。
- **请求体**:
  ```json
  {
    "refresh_token": "eyJhbGciOi..."
  }
  ```
- **响应体**:
  ```json
  {
    "access_token": "eyJhbGciOi...",
    "token_type": "Bearer",
    "expires_in": 3600
  }
  ```

#### 1.4 获取当前用户信息
- **Endpoint**: `GET /auth/me`
- **功能**: 获取当前登录用户的基本信息，用于前端初始化。
- **认证**: 需要
- **响应体**:
  ```json
  {
    "id": 1,
    "username": "admin",
    "roles": ["admin"]
  }
  ```

### 2. 诊断案例接口 (Case)

#### 2.1 获取案例列表
- **Endpoint**: `GET /cases`
- **功能**: 获取当前用户的诊断案例列表。
- **认证**: 需要
- **查询参数**:
  - `status` (string, 可选): `open` | `solved`。
  - `vendor` (string, 可选): 过滤指定厂商，如 `Huawei`、`Cisco`、`Juniper`。
  - `category` (string, 可选): 故障分类过滤，例如 `OSPF`、`BGP`、`MPLS`。
  - `attachmentType` (string, 可选): `topo` | `log` | `config` | `none`。
- **响应体**:
  ```json
  [
    {
      "caseId": "case_12345",
      "title": "MPLS VPN 跨域不通",
      "status": "open",
      "createdAt": "2025-07-15T10:30:00Z",
      "updatedAt": "2025-07-15T11:00:00Z"
    }
  ]
  ```

#### 2.2 创建新案例
- **Endpoint**: `POST /cases`
- **功能**: 用户输入第一个问题，创建新诊断案例及初始图。
- **认证**: 需要
- **请求体**:
  ```json
  {
    "query": "我的OSPF邻居状态为什么卡在ExStart？",
    "attachments": [
      { "type": "image", "url": "https://example.com/topology.png", "name": "topology.png" }
    ]
  }
  ```
- **响应体**: 包含`caseId`, `title`, 初始`nodes`和`edges`。
  ```json
  {
    "caseId": "case_67890",
    "title": "OSPF邻居状态卡在ExStart",
    "nodes": [ /* 初始节点数组, 如用户问题节点和AI首次分析节点 */ ],
    "edges": [ /* 初始边数组 */ ]
  }
  ```

#### 2.3 获取案例详情
- **Endpoint**: `GET /cases/{caseId}`
- **功能**: 获取指定案例的完整图谱信息。
- **认证**: 需要
- **响应体**: 同`POST /cases`响应。

#### 2.4 删除案例
- **Endpoint**: `DELETE /cases/{caseId}`
- **功能**: 删除一个诊断案例。
- **认证**: 需要
- **响应**: `204 No Content`

#### 2.5 驱动诊断路径
- **Endpoint**: `POST /cases/{caseId}/interactions`
- **功能**: 在某节点上交互，驱动AI生成后续节点，增量更新图。
- **认证**: 需要
- **请求体**:
  ```json
  {
    "parentNodeId": "node_a1b2c3d4",
    "response": {
      "text": "这是我获取的debug日志。",
      "attachments": [{ "type": "log", "url": "https://example.com/debug.log", "name": "debug.log" }]
    },
    "retrievalWeight": 0.7,          // 可选，0（关键词）~1（语义）
    "filterTags": ["OSPF", "日志"] // 可选，数组形式
  }
  ```
- **响应体**:
  ```json
  {
    "newNodes": [ /* 新增节点数组 */ ],
    "newEdges": [ /* 新增边数组 */ ],
    "processingNodeId": "node_ai_analysis_2" // 可选，表示当前正在处理中的节点ID
  }
  ```

#### 2.6 获取节点详情
- **Endpoint**: `GET /cases/{caseId}/nodes/{nodeId}`
- **功能**: 获取图中某个节点的详细信息。
- **认证**: 需要
- **响应体**: 单个`Node`对象。

#### 2.7 提交案例反馈
- **Endpoint**: `POST /cases/{caseId}/feedback`
- **功能**: 对整个案例的最终结果进行反馈，支持知识自进化流程。
- **认证**: 需要
- **请求体**:
  ```json
  {
    "outcome": "solved" | "unsolved" | "partially_solved",
    "rating": 5,
    "comment": "这个解决方案非常有效！",
    "corrected_solution": {
      "steps": [
        "第一步：检查MTU值是否一致。",
        "第二步：重启OSPF进程。",
        "第三步：验证邻居状态。"
      ],
      "explanation": "原始方案缺少了重启进程的关键步骤和验证环节。",
      "commands": {
        "Huawei": [
          "display interface mtu",
          "reset ospf process",
          "display ospf peer"
        ],
        "Cisco": [
          "show interface mtu",
          "clear ip ospf process",
          "show ip ospf neighbor"
        ]
      },
      "references": ["RFC2328", "华为VRP配置指南第3章"],
      "testResults": {
        "beforeFix": "邻居状态ExStart",
        "afterFix": "邻居状态Full",
        "verificationCommands": ["display ospf peer", "ping 192.168.1.2"]
      }
    },
    "knowledgeContribution": {
      "isNewSolution": true,
      "applicableScenarios": ["MTU不匹配", "OSPF邻居异常"],
      "difficultyLevel": "intermediate",
      "estimatedImpact": "high"
    },
    "additionalContext": {
      "deviceModels": ["NE40E-X8", "ASR9000"],
      "networkScale": "enterprise",
      "resolutionTime": 1800
    }
  }
  ```
- **响应体**:
  ```json
  {
    "status": "success",
    "message": "反馈已收到，感谢您的贡献！",
    "feedbackId": "feedback_001",
    "knowledgeStatus": "pending_review",
    "estimatedReviewTime": "2-3个工作日",
    "contributionPoints": 50
  }
  ```

#### 2.8 获取节点知识溯源
- **Endpoint**: `GET /cases/{caseId}/nodes/{nodeId}/knowledge`
- **功能**: 获取指定节点背后命中的文档片段及相似度，用于“知识溯源”Tab。
- **认证**: 需要
- **查询参数**:
  - `topK` (integer, 可选，默认 5): 返回的文档片段数量
  - `vendor` (string, 可选): 按指定厂商过滤，如 `Huawei`
  - `retrievalWeight` (float, 可选): 检索权重，0(关键词优先)~1(语义优先)
- **响应体**:
  ```json
  {
    "nodeId": "node_ai_analysis_1",
    "sources": [
      {
        "id": "doc_001",
        "title": "RFC 2328 - OSPFv2",
        "snippet": "If the neighbors are stuck in ExStart state, check MTU...",
        "relevance": 0.92,
        "url": "https://example.com/rfc2328",
        "highlights": [
          {
            "text": "ExStart state",
            "start": 45,
            "end": 57,
            "matchType": "keyword"
          },
          {
            "text": "check MTU",
            "start": 59,
            "end": 68,
            "matchType": "semantic"
          }
        ],
        "matchScore": {
          "keywordScore": 0.85,
          "semanticScore": 0.92,
          "finalScore": 0.89,
          "explanation": "关键词'ExStart'完全匹配，语义理解'MTU检查'高度相关"
        },
        "sourceDocument": {
          "fileName": "RFC2328-OSPF.pdf",
          "pageNumber": 25,
          "section": "7.2 The Neighbor Data Structure"
        }
      }
    ],
    "retrievalMetadata": {
      "totalCandidates": 50,
      "retrievalTime": 245,
      "rerankTime": 89,
      "strategy": "hybrid_search"
    }
  }
  ```

#### 2.9 获取节点厂商命令
- **Endpoint**: `GET /cases/{caseId}/nodes/{nodeId}/commands`
- **功能**: 根据设备厂商返回对应的可复制命令集，用于“可复制命令”Tab。
- **认证**: 需要
- **查询参数**:
  - `vendor` (string, 必需): 设备厂商，如 `Huawei`、`Cisco`、`Juniper`。
- **响应体**:
  ```json
  {
    "vendor": "Huawei",
    "commands": [
      "display ospf",
      "display interface GigabitEthernet0/0/0"
    ]
  }
  ```

#### 2.10 保存画布布局
- **Endpoint**: `PUT /cases/{caseId}/layout`
- **功能**: 保存用户自定义的画布节点位置布局。
- **认证**: 需要
- **请求体**:
  ```json
  {
    "nodePositions": [
      { "nodeId": "node_a1b2c3d4", "x": 100, "y": 200 },
      { "nodeId": "node_e5f6g7h8", "x": 300, "y": 150 }
    ],
    "viewportState": {
      "zoom": 1.2,
      "centerX": 250,
      "centerY": 180
    }
  }
  ```
- **响应体**: `204 No Content`

#### 2.11 获取画布布局
- **Endpoint**: `GET /cases/{caseId}/layout`
- **功能**: 获取之前保存的画布节点位置布局。
- **认证**: 需要
- **响应体**:
  ```json
  {
    "nodePositions": [
      { "nodeId": "node_a1b2c3d4", "x": 100, "y": 200 },
      { "nodeId": "node_e5f6g7h8", "x": 300, "y": 150 }
    ],
    "viewportState": {
      "zoom": 1.2,
      "centerX": 250,
      "centerY": 180
    }
  }
  ```

#### 2.12 获取节点状态
- **Endpoint**: `GET /cases/{caseId}/status`
- **功能**: 获取当前诊断案例的整体状态，包括是否有节点正在处理中。
- **认证**: 需要
- **响应体**:
  ```json
  {
    "caseStatus": "IN_PROGRESS" | "COMPLETED" | "AWAITING_INPUT",
    "processingNodeId": "node_ai_analysis_2", // 可选，当前正在处理中的节点ID
    "awaitingInputNodeId": "node_ai_clarification_1" // 可选，当前等待用户输入的节点ID
  }
  ```



### 3. 数据看板接口 (Dashboard)

#### 3.1 获取统计数据
- **Endpoint**: `GET /statistics`
- **功能**: 获取用于渲染数据看板的聚合统计数据。
- **认证**: 需要
- **查询参数**:
  - `timeRange` (string, 可选): `7d` | `30d` | `90d`。默认为 `30d`。
- **响应体**:
  ```json
  {
    "faultCategories": [
      { "name": "VPN", "value": 120, "percentage": 34.3, "trend": "+5.2%" },
      { "name": "OSPF", "value": 95, "percentage": 27.1, "trend": "-2.1%" },
      { "name": "BGP", "value": 60, "percentage": 17.1, "trend": "+1.8%" },
      { "name": "IPsec", "value": 45, "percentage": 12.9, "trend": "+0.5%" },
      { "name": "Other", "value": 30, "percentage": 8.6, "trend": "-1.2%" }
    ],
    "resolutionTrend": [
      { "date": "2025-07-01", "rate": 0.80, "totalCases": 45, "resolvedCases": 36 },
      { "date": "2025-07-02", "rate": 0.82, "totalCases": 52, "resolvedCases": 43 },
      { "date": "2025-07-03", "rate": 0.85, "totalCases": 48, "resolvedCases": 41 }
    ],
    "knowledgeCoverage": {
      "heatmapData": [
        {
          "topic": "OSPF",
          "vendor": "Huawei",
          "coverage": 95,
          "documentCount": 45,
          "lastUpdated": "2025-07-15T10:00:00Z",
          "qualityScore": 0.92,
          "gaps": ["OSPF v3配置", "NSSA区域故障"]
        },
        {
          "topic": "OSPF",
          "vendor": "Cisco",
          "coverage": 80,
          "documentCount": 32,
          "lastUpdated": "2025-07-14T15:30:00Z",
          "qualityScore": 0.85,
          "gaps": ["OSPF LSA类型7", "虚链路配置"]
        }
      ],
      "overallStats": {
        "totalTopics": 25,
        "averageCoverage": 78.5,
        "criticalGaps": 3,
        "recentlyUpdated": 12,
        "topVendors": ["Huawei", "Cisco", "Juniper"]
      }
    },
    "performanceMetrics": {
      "averageResolutionTime": 1245,
      "userSatisfactionRate": 0.87,
      "knowledgeBaseUsage": {
        "totalQueries": 2340,
        "hitRate": 0.92,
        "averageRetrievalTime": 234
      }
    },
    "userActivity": {
      "activeUsers": 156,
      "newCasesToday": 23,
      "pendingCases": 8,
      "topUsers": [
        {"username": "engineer_zhang", "casesResolved": 45},
        {"username": "admin_li", "casesResolved": 38}
      ]
    }
  }
  ```
- **数据结构说明**:
  - `faultCategories`: 故障分类饼图数据，增加百分比和趋势信息。
  - `resolutionTrend`: 解决率趋势图，增加案例总数和解决数量。
  - `knowledgeCoverage`: 知识覆盖度热力图，增加质量评分和缺口分析。
  - `performanceMetrics`: 系统性能指标。
  - `userActivity`: 用户活跃度统计。

### 4. 知识文档接口 (Knowledge Documents)

> 用户可通过以下端点上传原始知识文档（PDF、Word、图片、Markdown 等）并查看其解析状态。后台会将文件保存到**本地文件系统**，并异步触发 IDP 解析 → 语义切分 → 向量化 → 入库流水线。

#### 4.1 上传知识文档
- **Endpoint**: `POST /knowledge/documents`
- **功能**: 上传一个新的知识文档。
- **认证**: 需要
- **请求格式**: `multipart/form-data`
- **表单字段**:
  | 字段 | 类型 | 是否必填 | 说明 |
  | ---- | ---- | ------ | ---- |
  | `file` | file | 是 | 待上传的文档文件 |
  | `tags` | string[] | 否 | 标签数组，用于元数据过滤（如["OSPF","Huawei"]） |
  | `vendor` | string | 否 | 设备厂商标识，如 `Huawei` / `Cisco` |
- **响应体**:
  ```json
  {
    "docId": "doc_123456",
    "status": "QUEUED",
    "message": "文档已加入处理队列"
  }
  ```

#### 4.2 获取文档列表
- **Endpoint**: `GET /knowledge/documents`
- **功能**: 分页获取当前用户已上传的知识文档。
- **认证**: 需要
- **查询参数**:
  - `status` (string, 可选): `QUEUED` | `PARSING` | `INDEXED` | `FAILED`
  - `vendor` (string, 可选): 过滤指定厂商文档
  - `page` / `pageSize` (int, 可选): 分页参数
- **响应体**:
  ```json
  {
    "items": [
      {
        "docId": "doc_123456",
        "fileName": "OSPF_RFC2328.pdf",
        "vendor": "Huawei",
        "tags": ["OSPF"],
        "status": "INDEXED",
        "uploadedAt": "2025-07-20T09:00:00Z"
      }
    ],
    "total": 40,
    "page": 1,
    "pageSize": 10
  }
  ```

#### 4.3 获取单个文档详情
- **Endpoint**: `GET /knowledge/documents/{docId}`
- **功能**: 获取指定知识文档的元数据及解析进度。
- **认证**: 需要
- **响应体**:
  ```json
  {
    "docId": "doc_123456",
    "fileName": "OSPF_RFC2328.pdf",
    "vendor": "Huawei",
    "tags": ["OSPF"],
    "status": "PARSING",
    "progress": 45,
    "message": "正在进行OCR识别",
    "createdAt": "2025-07-20T09:00:00Z",
    "updatedAt": "2025-07-20T09:05:00Z"
  }
  ```

#### 4.4 删除知识文档
- **Endpoint**: `DELETE /knowledge/documents/{docId}`
- **功能**: 删除一个知识文档及其索引数据（需要管理员或文档上传者权限）。
- **认证**: 需要
- **响应**: `204 No Content`

#### 4.5 重新解析知识文档
- **Endpoint**: `PUT /knowledge/documents/{docId}/reparse`
- **功能**: 对解析失败或需要重新 OCR 的文档重新触发解析流水线。
- **认证**: 需要
- **响应体**:
  ```json
  {
    "docId": "doc_123456",
    "status": "PARSING",
    "message": "已触发重新解析"
  }
  ```

#### 4.6 更新文档元数据
- **Endpoint**: `PATCH /knowledge/documents/{docId}`
- **功能**: 更新文档的标签 (`tags`) 或厂商 (`vendor`) 等元数据。
- **认证**: 需要
- **请求体**:
  ```json
  {
    "tags": ["OSPF", "BGP"],
    "vendor": "Huawei"
  }
  ```
- **响应体**:
  ```json
  {
    "status": "success"
  }
  ```



### 5. 文件与附件接口 (Files)

#### 5.1 上传附件
- **Endpoint**: `POST /files`
- **功能**: 上传单个附件（图片 / 拓扑 / 日志压缩包等），返回文件`fileId`及访问 URL。
- **认证**: 需要
- **请求**: `multipart/form-data`
- **表单字段**:
  | 字段 | 类型 | 是否必填 | 说明 |
  | ---- | ---- | ------ | ---- |
  | `file` | file | 是 | 待上传的文件 |
  | `fileType` | string | 否 | 文件类型标识：`image`/`topo`/`log`/`config`/`other` |
  | `description` | string | 否 | 文件描述信息 |
- **文件限制**:
  - 支持格式：PDF, PNG, JPG, JPEG, GIF, TXT, LOG, ZIP, TAR.GZ
  - 单文件最大：50MB
  - 图片文件最大：10MB
- **响应体**:
  ```json
  {
    "fileId": "file_abc123",
    "url": "/api/v1/files/file_abc123",
    "fileName": "network_topology.png",
    "fileSize": 2048576,
    "fileType": "image",
    "mimeType": "image/png",
    "uploadedAt": "2025-07-15T10:30:00Z",
    "securityScan": {
      "status": "clean",
      "scanTime": "2025-07-15T10:30:05Z"
    }
  }
  ```

#### 5.2 获取附件
- **Endpoint**: `GET /files/{fileId}`
- **功能**: 下载 / 预览附件文件。
- **认证**: 需要（文件权限继承案例权限）
- **查询参数**:
  - `download` (boolean, 可选): 是否强制下载，默认为预览
  - `thumbnail` (boolean, 可选): 是否返回缩略图（仅图片文件）
- **响应**: `200 OK`, 文件流
- **响应头**:
  ```
  Content-Type: image/png
  Content-Length: 2048576
  Content-Disposition: inline; filename="network_topology.png"
  X-File-Security-Status: clean
  ```

#### 5.3 获取文件元数据
- **Endpoint**: `GET /files/{fileId}/metadata`
- **功能**: 获取文件的详细元数据信息。
- **认证**: 需要
- **响应体**:
  ```json
  {
    "fileId": "file_abc123",
    "fileName": "network_topology.png",
    "fileSize": 2048576,
    "fileType": "image",
    "mimeType": "image/png",
    "uploadedAt": "2025-07-15T10:30:00Z",
    "uploadedBy": "user_001",
    "associatedCases": ["case_001", "case_002"],
    "downloadCount": 15,
    "lastAccessed": "2025-07-15T14:20:00Z",
    "securityScan": {
      "status": "clean",
      "scanTime": "2025-07-15T10:30:05Z",
      "scanEngine": "ClamAV"
    }
  }
  ```

#### 5.4 批量上传附件
- **Endpoint**: `POST /files/batch`
- **功能**: 批量上传多个附件文件。
- **认证**: 需要
- **请求**: `multipart/form-data`，支持多个 `files[]` 字段
- **响应体**:
  ```json
  {
    "uploadResults": [
      {
        "fileName": "config1.txt",
        "status": "success",
        "fileId": "file_001",
        "url": "/api/v1/files/file_001"
      },
      {
        "fileName": "large_file.zip",
        "status": "failed",
        "error": "文件大小超过限制"
      }
    ],
    "summary": {
      "total": 2,
      "successful": 1,
      "failed": 1
    }
  }
  ```

### 6. 用户设置接口 (User Settings)

#### 6.1 获取用户设置
- **Endpoint**: `GET /user/settings`
- **功能**: 获取当前登录用户的个性化配置（主题、通知偏好等）。
- **认证**: 需要
- **响应体**:
  ```json
  {
    "theme": "system",          // light | dark | system
    "notifications": {
      "solution": true,
      "mention": false
    }
  }
  ```
#### 6.2 更新用户设置
- **Endpoint**: `PUT /user/settings`
- **功能**: 更新用户个性化配置。
- **认证**: 需要
- **请求体**: 与 `GET /user/settings` 响应体结构相同，可部分字段更新。
- **响应体**:
  ```json
  {
    "status": "success"
  }
  ```

### 7. 通知接口 (Notifications)

#### 7.1 获取通知列表
- **Endpoint**: `GET /notifications`
- **功能**: 分页获取当前用户的历史通知。
- **认证**: 需要
- **查询参数**:
  - `page` (integer, 可选, 默认 1)
  - `pageSize` (integer, 可选, 默认 20)
- **响应体**:
  ```json
  {
    "page": 1,
    "pageSize": 20,
    "total": 45,
    "items": [
      {
        "id": "noti_001",
        "type": "solution",          // solution | mention | system
        "title": "诊断案例已生成解决方案",
        "read": false,
        "createdAt": "2025-07-15T11:30:00Z"
      }
    ]
  }
  ```

#### 7.2 标记通知已读
- **Endpoint**: `POST /notifications/{notificationId}/read`
- **功能**: 将指定通知标记为已读。
- **认证**: 需要
- **响应**: `204 No Content`

### 8. 智能分析接口 (AI Analysis)

#### 8.1 解析技术日志
- **Endpoint**: `POST /analysis/log-parsing`
- **功能**: 使用大模型解析网络设备日志，提取关键信息和异常点。
- **认证**: 需要
- **请求体**:
  ```json
  {
    "logType": "debug_ip_packet" | "ospf_debug" | "bgp_debug" | "system_log",
    "vendor": "Huawei" | "Cisco" | "Juniper",
    "logContent": "原始日志内容...",
    "contextInfo": {
      "deviceModel": "NE40E-X8",
      "problemDescription": "用户报告的问题描述"
    }
  }
  ```
- **响应体**:
  ```json
  {
    "summary": "检测到OSPF邻居建立过程中的MTU不匹配问题",
    "anomalies": [
      {
        "type": "MTU_MISMATCH",
        "severity": "high",
        "location": "interface GE0/0/1",
        "description": "检测到MTU不匹配问题，本端1500，对端1400"
      }
    ],
    "suggestedActions": [
      {
        "action": "检查接口MTU配置",
        "commands": {
          "Huawei": ["display interface GE0/0/1", "interface GE0/0/1", "mtu 1500"],
          "Cisco": ["show interface GigabitEthernet0/0/1", "interface GigabitEthernet0/0/1", "mtu 1500"]
        }
      }
    ]
  }
  ```


