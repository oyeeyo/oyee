# SQLBot 运营平台对接文档

> 本文档面向运营平台开发人员，说明如何对接 SQLBot 接口。
> 无需 JWT、无需共享密钥、无需运营平台后端参与——前端直接调用即可。

---

## 一、整体架构

```
运营平台用户 → 运营平台前端 → SQLBot 后端 → 云端大模型（生成 SQL）
                                          ↓
                                    用户的业务数据库（查数据）
```

**运营平台前端直接和 SQLBot 后端交互**，不需要运营平台后端中转，不需要关心大模型和数据库的细节。

---

## 二、核心概念：account（工号）

运营平台只需要在每次请求中传入当前用户的 **account（工号）** 和 **name（姓名）**，SQLBot 会自动识别用户：

- 用户已存在 → 直接使用该用户身份
- 用户不存在 → 自动创建用户并分配到默认工作空间

```
运营平台前端
    ↓  每次请求带上 account + name
SQLBot 后端 → 按 account 查找/创建用户 → 执行操作 → 返回结果
```

---

## 三、对接步骤

### 步骤 1：确认 SQLBot 配置

SQLBot 的 `.env` 文件中需要配置默认工作空间 ID：

```env
OPS_PLATFORM_DEFAULT_OID=1    # 自动创建用户时分配的工作空间 ID
```

> 运营平台侧无需任何配置，不需要密钥、不需要证书。

### 步骤 2：前端直接调用接口

所有接口都是 **POST** 请求，在 body 中传 `account` 和 `name` 即可。

```javascript
// 统一的请求方法
const SQLBOT_URL = "http://<SQLBot地址>:8000"

async function sqlbotRequest(path, account, name, params = {}) {
  const resp = await fetch(`${SQLBOT_URL}/api/v1/ops/${path}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ account, name: name || '', ...params })
  })
  const result = await resp.json()

  // 成功时返回 data 部分，失败时抛出错误
  if (result.code !== 0) {
    throw new Error(result.msg || '请求失败')
  }
  return result.data
}
```

---

## 四、通用响应格式

所有成功接口的响应都遵循统一格式：

```json
{
    "code": 0,
    "data": { ... },
    "msg": null
}
```

| 字段 | 类型 | 说明 |
|------|------|------|
| `code` | int | `0` 表示成功，非 `0` 表示失败 |
| `data` | any | 实际的业务数据（各接口不同） |
| `msg` | string\|null | 错误信息，成功时为 `null` |

> 下面各接口的响应示例只展示 `data` 部分的内容。

---

## 五、接口文档

> 基础地址：`http://<SQLBot地址>:8000/api/v1/ops`
> 所有接口均为 **POST** 请求，Content-Type 为 `application/json`
> 所有接口都需要在 body 中传 `account` 字段，`name` 可选（首次调用时用于自动创建用户）

---

### 5.1 提问接口

**发送自然语言问题，返回 SQL + 查询结果 + 图表配置 + 思考过程。**

```
POST /ops/chat
```

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `account` | string | ✅ | 用户工号（唯一标识） |
| `name` | string | ❌ | 用户姓名（首次调用时用于自动创建用户） |
| `question` | string | ✅ | 用户的自然语言问题 |
| `chat_id` | int | ❌ | 对话 ID。不传则自动创建新对话 |
| `datasource_id` | int | ❌ | 数据源 ID。不传则自动使用工作空间下第一个可用数据源 |
| `stream` | bool | ❌ | 是否流式返回，默认 false |

**默认数据源：** 不传 `datasource_id` 时，系统自动选择当前工作空间下的第一个可用数据源。如果工作空间下没有数据源且没有指定 `chat_id`，则返回 400 错误。

**请求示例：**

```json
{
    "account": "001",
    "name": "张三",
    "question": "查一下上个月销售额前10的客户",
    "datasource_id": 1
}
```

**响应示例（data 部分，`stream=false` 时）：**

```json
{
    "success": true,
    "record_id": 12345,
    "chat_id": 67,
    "title": "上月销售TOP10客户",
    "sql": "SELECT customer_name, SUM(amount) as total FROM orders WHERE order_date >= '2026-07-01' GROUP BY customer_name ORDER BY total DESC LIMIT 10",
    "data": {
        "fields": ["customer_name", "total"],
        "data": [
            {"customer_name": "张三公司", "total": 150000},
            {"customer_name": "李四集团", "total": 120000}
        ]
    },
    "chart": {
        "type": "bar",
        "columns": [{"name": "客户", "value": "customer_name"}],
        "axis": {
            "x": {"name": "客户名", "value": "customer_name"},
            "y": {"name": "销售额", "value": "total"}
        }
    },
    "reasoning": {
        "choose_datasource": "思考过程：1. 用户未指定数据源，自动选择工作空间下的第一个数据源...",
        "generate_sql": "思考过程：1. 分析用户需求：查询上个月销售额前10的客户\n2. 确定表：orders 表\n3. 生成 SQL...",
        "generate_chart": "思考过程：1. SQL 返回客户名和销售额\n2. 推荐图表类型：柱状图..."
    }
}
```

> **`reasoning` 字段说明：** 非流式模式（`stream=false`）下，响应会额外包含 `reasoning` 字段，记录 AI 在各个步骤的思考过程。每个步骤对应一个 key，内容为 LLM 的推理文字。如果某步骤没有思考内容，则该 key 不会出现。

**流式模式（`stream=true`）：**

流式模式下，响应为 SSE（Server-Sent Events）格式，每个 chunk 实时推送。当 AI 生成思考内容时，chunk 中会包含 `reasoning_content` 字段：

```
data:{"content":"","reasoning_content":"思考过程：1. 分析用户需求...","type":"sql-result"}

data:{"content":"","reasoning_content":"2. 确定表：orders 表...","type":"sql-result"}

data:{"content":"","reasoning_content":"","type":"sql-result"}

data:{"content":"SELECT customer_name...","reasoning_content":"","type":"sql"}
```

| 事件类型 | 说明 |
|---------|------|
| `datasource-result` | 数据源选择的思考过程 |
| `sql-result` | SQL 生成的思考过程 + SQL 正文 |
| `chart-result` | 图表生成的思考过程 |
| `analysis-result` | 数据分析的思考过程 |
| `predict-result` | 数据预测的思考过程 |
| `sql` | 格式化后的 SQL |
| `sql-data` | SQL 执行成功 |
| `chart` | 图表配置（JSON） |
| `finish` | 全部完成 |

> 运营平台前端可用 `fetch` + `ReadableStream` 接收 SSE 事件，实时展示思考过程。

**追问（同一轮对话）：**

```json
{
    "account": "001",
    "name": "张三",
    "question": "按区域分组看一下",
    "chat_id": 67
}
```

---

### 5.2 历史对话列表

**获取当前用户的所有对话记录。**

```
POST /ops/history
```

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `account` | string | ✅ | 用户工号 |
| `name` | string | ❌ | 用户姓名 |

**请求示例：**

```json
{"account": "001", "name": "张三"}
```

**响应示例（data 部分）：**

```json
[
    {
        "id": 123,
        "brief": "上月销售TOP10客户",
        "create_time": "2026-08-13T10:30:00",
        "datasource": 1,
        "engine_type": "mysql",
        "latest_record_time": "2026-08-13T11:00:00"
    },
    {
        "id": 124,
        "brief": "各部门预算执行情况",
        "create_time": "2026-08-12T14:00:00",
        "datasource": 2,
        "engine_type": "pg"
    }
]
```

---

### 5.3 对话详情

**获取某个对话的完整问答记录。**

```
POST /ops/chat/detail
```

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `account` | string | ✅ | 用户工号 |
| `name` | string | ❌ | 用户姓名 |
| `chat_id` | int | ✅ | 对话 ID |

**请求示例：**

```json
{
    "account": "001",
    "name": "张三",
    "chat_id": 123
}
```

**响应示例（data 部分）：**

```json
{
    "id": 123,
    "brief": "上月销售TOP10客户",
    "datasource": 1,
    "datasource_name": "订单数据库",
    "records": [
        {
            "id": 456,
            "question": "查一下上个月销售额前10的客户",
            "sql_content": "SELECT ...",
            "create_time": "2026-08-13T10:30:00"
        }
    ]
}
```

---

### 5.4 导出 Excel

**导出指定问答记录的查询结果为 Excel 文件。**

```
POST /ops/export
```

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `account` | string | ✅ | 用户工号 |
| `name` | string | ❌ | 用户姓名 |
| `chat_record_id` | int | ✅ | 问答记录 ID（从对话详情中获取） |
| `chat_id` | int | ✅ | 对话 ID |

**请求示例：**

```json
{
    "account": "001",
    "name": "张三",
    "chat_record_id": 456,
    "chat_id": 123
}
```

**响应：** Excel 文件流（`application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`）

> 导出接口返回的是文件流，不是 JSON。

包含两个 Sheet：
- **Sheet 1 "明细"**：原始查询数据
- **Sheet 2 "汇总"**：按第一列分组，其他列合并去重值

**下载示例（JavaScript）：**

```javascript
async function downloadExcel(account, name, chatRecordId, chatId) {
  const resp = await fetch(`${SQLBOT_URL}/api/v1/ops/export`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ account, name, chat_record_id: chatRecordId, chat_id: chatId })
  })
  const blob = await resp.blob()
  const url = URL.createObjectURL(blob)
  const a = document.createElement('a')
  a.href = url
  a.download = '导出.xlsx'
  a.click()
  URL.revokeObjectURL(url)
}
```

---

### 5.5 数据源列表

**获取当前用户工作空间下的所有数据源。**

```
POST /ops/datasources
```

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `account` | string | ✅ | 用户工号 |
| `name` | string | ❌ | 用户姓名 |

**请求示例：**

```json
{"account": "001"}
```

**响应示例（data 部分）：**

```json
[
    {
        "id": 1,
        "name": "订单数据库",
        "type": "mysql",
        "type_name": "MySQL",
        "description": "生产环境订单库",
        "status": "Success",
        "num": "15/20"
    },
    {
        "id": 2,
        "name": "财务数据库",
        "type": "pg",
        "type_name": "PostgreSQL",
        "status": "Success",
        "num": "8/8"
    }
]
```

---

### 5.6 仪表盘列表

**获取当前用户的仪表盘列表。**

```
POST /ops/dashboards
```

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `account` | string | ✅ | 用户工号 |
| `name` | string | ❌ | 用户姓名 |

**请求示例：**

```json
{"account": "001"}
```

**响应示例（data 部分）：**

```json
[
    {
        "id": "dashboard_001",
        "name": "运营数据总览",
        "node_type": "dashboard",
        "create_time": 1720000000
    }
]
```

---

### 5.7 创建仪表盘（添加到仪表板）

**将提问结果的图表添加到仪表盘。**

```
POST /ops/dashboard/create
```

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `account` | string | ✅ | 用户工号 |
| `name` | string | ❌ | 仪表盘名称（不传则用问答标题） |
| `record_id` | int | ✅ | 问答记录 ID（提问接口返回的 `record_id`） |

**请求示例：**

```json
{
    "account": "001",
    "name": "我的仪表盘",
    "record_id": 199
}
```

**响应示例（data 部分）：**

```json
{
    "id": "e202596aa550451986a68b018a0eba69",
    "name": "我的仪表盘",
    "create_time": 1787213808
}
```

> **权限校验**：只能操作自己创建的问答记录，否则会返回 403。

---

### 5.8 加载仪表盘详情

**加载仪表盘的完整信息（含图表数据）。**

```
POST /ops/dashboard/load
```

**请求参数：**

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `account` | string | ✅ | 用户工号 |
| `name` | string |  | 用户姓名 |
| `dashboard_id` | string | ✅ | 仪表盘 ID（从创建仪表盘或仪表盘列表接口获取） |

**请求示例：**

```json
{
    "account": "001",
    "dashboard_id": "e202596aa550451986a68b018a0eba69"
}
```

**响应示例（data 部分）：**

```json
{
    "id": "e202596aa550451986a68b018a0eba69",
    "name": "我的仪表盘",
    "type": "chart",
    "node_type": "dashboard",
    "component_data": "{\"content\": \"{\\\"type\\\": \\\"table\\\", ...}\"}",
    "canvas_style_data": "{}",
    "canvas_view_info": "{}",
    "pid": "",
    "create_time": 1787213808
}
```

> **权限校验**：只能访问自己创建的仪表盘，否则会返回 404。
> `component_data` 是 JSON 字符串，包含图表的完整配置（类型、标题、字段、轴配置等）。

---

## 六、错误处理

### 请求参数错误（HTTP 422）

缺少必填字段时返回：

```json
{
    "detail": [
        {
            "type": "missing",
            "loc": ["body", "account"],
            "msg": "Field required",
            "input": {"name": "test"}
        }
    ]
}
```

### 业务错误（HTTP 500）

业务逻辑错误时返回错误描述字符串，例如：

```
"Chat with id 99999 not found"
"问答记录不存在"
"数据为空，无法导出"
```

### 用户状态错误（HTTP 400）

```
"用户已被禁用"
```

### 权限错误（HTTP 403）

```
"无权导出此记录"
```

### 错误码汇总

| HTTP 状态码 | 含义 | 处理建议 |
|-------------|------|---------|
| 200 | 成功 | — |
| 400 | 用户被禁用 | 联系管理员 |
| 403 | 无权访问 | 检查用户是否有对应资源权限 |
| 404 | 资源不存在 | 检查 chat_id / record_id 是否正确 |
| 422 | 请求参数错误 | 检查 JSON 格式，确认 account 字段存在 |
| 500 | 业务错误 | 根据返回的错误信息处理 |

---

## 七、运营平台前端完整代码示例

```javascript
/**
 * 运营平台前端 - SQLBot 集成模块
 */
const SQLBOT_URL = "http://192.168.x.x:8000"   // SQLBot 服务器地址

// ============ 统一请求方法 ============

async function sqlbotRequest(path, account, name, params = {}) {
  const resp = await fetch(`${SQLBOT_URL}/api/v1/ops/${path}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ account, name: name || '', ...params })
  })
  const result = await resp.json()
  if (result.code !== 0) {
    throw new Error(result.msg || `请求失败: ${resp.status}`)
  }
  return result.data
}

// ============ 业务接口封装 ============

class SQLBotClient {
  constructor(account, name) {
    this.account = account
    this.name = name || ''
  }

  async ask(question, options = {}) {
    const params = { question }
    if (options.chatId) params.chat_id = options.chatId
    if (options.datasourceId) params.datasource_id = options.datasourceId
    if (options.stream !== undefined) params.stream = options.stream
    return sqlbotRequest('chat', this.account, this.name, params)
  }

  /**
   * 流式提问（返回 SSE 事件，实时推送思考过程和结果）
   * @returns {ReadableStream}
   */
  async askStream(question, options = {}) {
    const params = { question, stream: true }
    if (options.chatId) params.chat_id = options.chatId
    if (options.datasourceId) params.datasource_id = options.datasourceId
    return fetch(`${SQLBOT_URL}/api/v1/ops/chat`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ account: this.account, name: this.name, ...params })
    })
  }

  async history() {
    return sqlbotRequest('history', this.account, this.name)
  }

  async chatDetail(chatId) {
    return sqlbotRequest('chat/detail', this.account, this.name, { chat_id: chatId })
  }

  async exportExcel(chatRecordId, chatId) {
    const resp = await fetch(`${SQLBOT_URL}/api/v1/ops/export`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        account: this.account, name: this.name,
        chat_record_id: chatRecordId, chat_id: chatId
      })
    })
    return resp.blob()
  }

  async datasources() {
    return sqlbotRequest('datasources', this.account, this.name)
  }

  async dashboards() {
    return sqlbotRequest('dashboards', this.account, this.name)
  }

  async createDashboard(recordId, name = '') {
    return sqlbotRequest('dashboard/create', this.account, this.name, { record_id: recordId, name })
  }

  async loadDashboard(dashboardId) {
    return sqlbotRequest('dashboard/load', this.account, this.name, { dashboard_id: dashboardId })
  }
}

// ============ 使用示例 ============

// 1. 用户登录运营平台后，创建 SQLBot 客户端
//    account 对应用户工号，name 对应用户姓名
const client = new SQLBotClient("001", "张三")

// 2. 查看有哪些数据源（首次调用会自动在 SQLBot 创建用户）
const dsList = await client.datasources()
console.log("可用数据源:", dsList.map(ds => ds.name))

// 3. 提问（不传 datasource_id，自动使用默认数据源）
const result = await client.ask("查一下上个月销售额前10的客户")
console.log("SQL:", result.sql)
console.log("数据:", result.data)
console.log("思考过程:", result.reasoning)  // 各步骤的 AI 推理内容
console.log("record_id:", result.record_id, "chat_id:", result.chat_id)

// 4. 追问（使用上一次返回的 chat_id）
const result2 = await client.ask("按区域分组看一下", { chatId: result.chat_id })

// 5. 查历史
const history = await client.history()
console.log("对话列表:", history.map(h => h.brief))

// 6. 查对话详情
const detail = await client.chatDetail(result.chat_id)
console.log("问答记录:", detail.records)

// 7. 导出 Excel（返回 Blob，直接下载）
const blob = await client.exportExcel(result.record_id, result.chat_id)
const url = URL.createObjectURL(blob)
const a = document.createElement('a')
a.href = url
a.download = `${detail.brief || '导出'}.xlsx`
a.click()
URL.revokeObjectURL(url)
```

---

## 八、常见问题

### Q1：首次调用会自动创建用户吗？

**是的。** 如果 account 在 SQLBot 中不存在，会自动创建用户并分配到默认工作空间（由 `OPS_PLATFORM_DEFAULT_OID` 配置）。

### Q2：需要管理 JWT / Token 吗？

**不需要。** 每次请求直接传 `account` 和 `name` 即可，没有过期时间、没有缓存问题。

### Q3：需要运营平台后端参与吗？

**不需要。** 所有接口都可以由前端直接调用。运营平台后端无需任何改动。

### Q4：提问接口返回慢怎么办？

大模型生成 SQL 需要时间（通常 5-15 秒），这是正常的。建议前端显示加载状态。

如果需要实时看到 AI 的思考过程，可以使用流式模式（`stream: true`），响应会逐步推送 SSE 事件，每个事件包含 `reasoning_content`（思考过程）和 `content`（SQL/图表正文），前端可以像打字机一样实时展示。

### Q5：用户只能看到部分数据源？

数据源按工作空间隔离。确保管理员在 SQLBot 后台把数据源分配到了正确的工作空间（`OPS_PLATFORM_DEFAULT_OID` 对应的空间）。

### Q6：之前用的 JWT 方式还能用吗？

**可以。** 旧的 `/login/ops-platform` 登录接口仍然保留，如果运营平台后端想用 JWT 方式也可以继续用。两种方式可以共存。

### Q7：不是 MCP 吗？怎么是普通 API？

这里用的是 SQLBot 的**专用 API 接口**（`/ops/*`），专门为运营平台对接设计。MCP 是 SQLBot 的另一套接口（`/mcp/*`），功能较少。我们用 `/ops/*` 接口是因为它功能更完整、返回格式更友好。

### Q8：数据库里有些字段存的是数字（如 1、2、3），但注释里有对应的文字（如"1待起飞、2飞行中"），返回结果能显示文字吗？

**可以。** SQLBot 的提示词中已配置了枚举字段自动转换规则：当字段注释包含数字与文字的映射关系时，LLM 会自动在生成的 SQL 中使用 `CASE WHEN` 将数字转换为文字。无论分隔符是顿号、逗号、等号还是其他格式，都能识别。

```sql
-- 字段注释: 航班状态：1待起飞、2飞行中、3降落中
-- LLM 自动生成:
CASE flight_status WHEN 1 THEN '待起飞' WHEN 2 THEN '飞行中' WHEN 3 THEN '降落中' ELSE '未知' END AS flight_status
```

如果 LLM 遗漏了 CASE WHEN，可以在 SQLBot 的"自定义提示词"功能中补充一条规则来强制要求。

---

## 九、接口一览表

| 接口 | 路径 | 功能 |
|------|------|------|
| 提问 | `POST /api/v1/ops/chat` | 自然语言提问，返回 SQL + 数据 + 思考过程 |
| 历史 | `POST /api/v1/ops/history` | 获取对话列表 |
| 详情 | `POST /api/v1/ops/chat/detail` | 获取对话问答记录 |
| 导出 | `POST /api/v1/ops/export` | 导出 Excel（文件流） |
| 数据源 | `POST /api/v1/ops/datasources` | 数据源列表 |
| 仪表盘列表 | `POST /api/v1/ops/dashboards` | 仪表盘列表 |
| 创建仪表盘 | `POST /api/v1/ops/dashboard/create` | 将问答结果添加到仪表板 |
| 加载仪表盘 | `POST /api/v1/ops/dashboard/load` | 加载仪表盘详情（含图表数据） |

---

## 十、与旧版（JWT 方式）的对比

| | 旧版（JWT） | 新版（account） |
|---|---|---|
| 运营平台后端 | 需要（签名 + 换 JWT） | **不需要** |
| 认证方式 | JWT Token（8天有效期） | account 工号（无需管理） |
| 请求参数 | `token: "Bearer eyJ..."` | `account: "001"` |
| 用户自动创建 | ✅ | ✅ |
| 用户隔离 | ✅ | ✅ |
| 共享密钥 | 需要配置 | **不需要** |
| 前端依赖 | 无 | 无 |
| 后端依赖 | PyJWT | 无 |
