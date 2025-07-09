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

## 核心概念与数据结构

每一次独立的诊断过程视为一个“**诊断案例 (Case)**”。每个`Case`在逻辑上是一个有向图（Diagnostic Path），由多个“**节点 (Node)**”和连接它们的“**边 (Edge)**”组成。

### Node (节点)
```json
{
  "id": "node_a1b2c3d4",
  "type": "USER_QUERY" | "AI_ANALYSIS" | "AI_CLARIFICATION" | "USER_RESPONSE" | "SOLUTION",
  "title": "标题/节点摘要",
  "status": "COMPLETED" | "AWAITING_USER_INPUT",
  "content": {
    // 当 type 为 USER_QUERY / USER_RESPONSE 时
    "text": "问题文本或用户补充信息",
    "attachments": [ { "type": "image", "url": "https://...", "name": "screenshot.png" } ],

    // 当 type 为 AI_ANALYSIS 时
    "analysis": "AI 的分析摘要",
    "suggestedQuestions": [ "下一步该检查什么？" ],

    // 当 type 为 SOLUTION 时
    "steps": [ "Step 1", "Step 2" ],
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

### Edge (边)
```json
{
  "source": "node_a1b2c3d4", // 源节点ID
  "target": "node_e5f6g7h8"  // 目标节点ID
}
```

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

### 2. 诊断案例接口 (Case)

#### 2.1 获取案例列表
- **Endpoint**: `GET /cases`
- **功能**: 获取当前用户的诊断案例列表。
- **认证**: 需要
- **查询参数**:
  - `status` (string, 可选): `open` | `solved`。
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
    "query": "我的OSPF邻居状态为什么卡在ExStart？"
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
    "parentNodeId": "node_ai_clarification_1",
    "response": {
      "text": "这是我获取的debug日志。",
      "attachments": [{ "type": "log", "content": "..." }]
    },
    "retrievalWeight": 0.7,          // 可选，0（关键词）~1（语义）
    "filterTags": ["OSPF", "日志"] // 可选，数组形式
  }
  ```
- **响应体**:
  ```json
  {
    "newNodes": [ /* 新增节点数组 */ ],
    "newEdges": [ /* 新增边数组 */ ]
  }
  ```

#### 2.6 获取节点详情
- **Endpoint**: `GET /cases/{caseId}/nodes/{nodeId}`
- **功能**: 获取图中某个节点的详细信息。
- **认证**: 需要
- **响应体**: 单个`Node`对象。

#### 2.7 提交案例反馈
- **Endpoint**: `POST /cases/{caseId}/feedback`
- **功能**: 对整个案例的最终结果进行反馈。
- **认证**: 需要
- **请求体**:
  ```json
  {
    "outcome": "solved" | "unsolved",
    "comment": "这个解决方案非常有效！"
  }
  ```
- **响应体**:
  ```json
  {
    "status": "success",
    "message": "反馈已收到。"
  }
  ```

#### 2.8 获取节点知识溯源
- **Endpoint**: `GET /cases/{caseId}/nodes/{nodeId}/knowledge`
- **功能**: 获取指定节点背后命中的文档片段及相似度，用于“知识溯源”Tab。
- **认证**: 需要
- **查询参数**:
  - `topK` (integer, 可选，默认 5): 返回的文档片段数量
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
        "url": "https://example.com/rfc2328"
      }
    ]
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

### 3. 数据看板接口 (Dashboard)

#### 3.1 获取系统统计
- **Endpoint**: `GET /dashboard/stats`
- **功能**: 获取全局统计数据。
- **认证**: 需要
- **响应体**:
  ```json
  {
    "totalCases": 150,
    "solvedCases": 120,
    "averageResolutionTime": 3600
  }
  ```

### 4. 枚举信息接口 (Metadata)

#### 4.1 获取系统标签
- **Endpoint**: `GET /metadata/tags`
- **功能**: 获取系统内可用于过滤的全部标签集合（如厂商、故障类型、文档类型）。
- **认证**: 可选
- **响应体**:
  ```json
  {
    "vendors": ["Huawei", "Cisco", "Juniper"],
    "faultCategories": ["OSPF", "BGP", "MPLS"],
    "docTypes": ["RFC", "官方手册", "社区博客"]
  }
  ``` 