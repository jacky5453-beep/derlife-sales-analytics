# SmartWeb MCP API 串接使用手冊

## 版本修訂紀錄

| 版本 | 日期 | 更新內容摘要 |
| :--- | :--- | :--- |
| **1.0** | 2026-03-11 | 最初版本發布，支援產品庫存查詢與更新等共 5 種 API 工具。 |
| **2.0** | 2026-03-18 | 改為標準 MCP Protocol（JSON-RPC 2.0），新增 `initialize`、`ping`、`tools/list` 方法，支援直接掛載至 Claude Code / Claude Desktop。 |
| **3.0** | 2026-04-10 | 新增訂單查詢工具（`list_orders`、`get_order`） |

---

本手冊提供外部系統串接 SmartWeb MCP (Model Context Protocol) API 的技術說明。

目前系統提供以下七種工具 (Tools)：

1. **`list_product`**：產品列表查詢，支援分頁與關鍵字。
2. **`get_product_stock`**：查詢單一產品庫存，有規格時一併回傳所有規格。
3. **`update_product_stock`**：更新產品/規格庫存，支援流水號或產品全編號。
4. **`batch_get_product_stock`**：批次查詢多個產品庫存。
5. **`batch_update_product_stock`**：批次更新多組商品或規格庫存。
6. **`list_orders`**：訂單列表查詢，支援分頁、狀態篩選、日期範圍與關鍵字。
7. **`get_order`**：依訂單編號查詢完整訂單資訊，含商品明細。

---

## 1. 系統架構與認證

### Endpoint

| 項目 | 說明 |
|---|---|
| **URL** | `https://[YOUR_DOMAIN]/mcp/index.php` |
| **HTTP 方法** | `POST` |
| **資料格式** | `Content-Type: application/json` |
| **協議** | JSON-RPC 2.0（MCP Protocol） |

### 身分驗證

除 `initialize` 與 `ping` 外，所有請求都需要帶入有效的 API Key（依優先順序）：

1. **Authorization Header（推薦）**：`Authorization: Bearer <YOUR_API_KEY>`
2. **自訂 Header**：`X-API-Key: <YOUR_API_KEY>`
3. **URL 參數**：`?api_key=<YOUR_API_KEY>`

> **取得 API Key**：至 SmartWeb 後台 → **MCP API Key 管理**（需總管理權限）→ 新增後系統自動產生，完整金鑰**僅於建立當下顯示一次**，請務必複製保存。

所有 `tools/call` 請求皆會自動記錄至 `mcp_api_log` 供後台稽核。

### 用量限制（Quota）

Quota 由平台管理員統一設定，客戶後台不可自行修改。

| 限制類型 | 說明 |
|---|---|
| **每分鐘呼叫上限（Rate Limit）** | 超過時回傳 HTTP `-32029` Too Many Requests |
| **每日呼叫上限** | 超過時拒絕請求 |
| **每月呼叫上限** | 超過時拒絕請求 |

> 上限值為 `0` 時表示無限制。當前用量可在後台 MCP API Key 管理頁面查閱。

---

## 2. JSON-RPC 2.0 請求格式

所有請求皆使用 JSON-RPC 2.0 格式：

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "<方法名稱>",
    "params": { ... }
}
```

| 欄位 | 說明 |
|---|---|
| `jsonrpc` | 固定為 `"2.0"` |
| `id` | 請求識別碼（任意整數或字串），回應會帶回相同的 `id` |
| `method` | 呼叫的方法，見下方方法列表 |
| `params` | 方法參數，依各方法而異 |

### 回應格式

**成功**
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            { "type": "text", "text": "{...工具回傳的 JSON 字串...}" }
        ]
    }
}
```

**工具執行失敗**（如找不到產品）
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            { "type": "text", "text": "{\"error\": \"Product not found\"}" }
        ],
        "isError": true
    }
}
```

**協議/認證錯誤**
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "error": {
        "code": -32001,
        "message": "Unauthorized: Invalid or missing API Key"
    }
}
```

---

## 3. 系統方法

### 3.1 握手初始化（`initialize`）

建立連線時呼叫，**不需要 API Key**。

**請求：**
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
        "protocolVersion": "2024-11-05",
        "clientInfo": { "name": "my-client", "version": "1.0.0" }
    }
}
```

**回應：**
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "protocolVersion": "2024-11-05",
        "capabilities": { "tools": {} },
        "serverInfo": { "name": "smartweb-mcp", "version": "1.0.0" }
    }
}
```

---

### 3.2 Ping（`ping`）

用於 keepalive 確認連線，**不需要 API Key**。

**請求：**
```json
{ "jsonrpc": "2.0", "id": 2, "method": "ping" }
```

**回應：**
```json
{ "jsonrpc": "2.0", "id": 2, "result": {} }
```

---

### 3.3 列出工具（`tools/list`）

取得所有可用工具的名稱、描述與參數 Schema，**需要 API Key**。

**請求：**
```json
{ "jsonrpc": "2.0", "id": 3, "method": "tools/list" }
```

**回應（節錄）：**
```json
{
    "jsonrpc": "2.0",
    "id": 3,
    "result": {
        "tools": [
            {
                "name": "list_product",
                "description": "查詢產品列表，支援分頁與關鍵字，每次最多回傳 100 筆。",
                "inputSchema": {
                    "type": "object",
                    "properties": {
                        "page":    { "type": "integer", "description": "頁碼，預設 1" },
                        "keyword": { "type": "string",  "description": "搜尋關鍵字（產品名稱或編號）" }
                    }
                }
            }
        ]
    }
}
```

---

## 4. 工具呼叫（`tools/call`）

所有工具都透過 `method: "tools/call"` 呼叫，並在 `params` 中指定工具名稱與參數：

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
        "name": "<工具名稱>",
        "arguments": { ... }
    }
}
```

---

### 4.1 產品列表查詢（`list_product`）

**請求：**
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
        "name": "list_product",
        "arguments": {
            "page": 1,
            "keyword": "搜尋關鍵字"
        }
    }
}
```

**回應（`result.content[0].text` 解析後）：**
```json
{
    "page": 1,
    "per_page": 100,
    "total_count": 55,
    "total_pages": 1,
    "data": [
        {
            "product_serial": 123,
            "title": "測試產品",
            "item_number": "P123",
            "stock": 50,
            "safety_stock": 5,
            "has_spec": "Y",
            "specifications": [
                {
                    "serial": 456,
                    "name": "紅色 L號",
                    "stock": 10,
                    "safety_stock": 2,
                    "item_number": "R-L",
                    "full_item_number": "P123R-L"
                }
            ]
        }
    ]
}
```

---

### 4.2 取得單一產品庫存（`get_product_stock`）

**請求：**
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
        "name": "get_product_stock",
        "arguments": { "product_serial": 123 }
    }
}
```

**回應 - 有規格：**
```json
{
    "product_serial": 123,
    "title": "測試產品名稱",
    "has_spec": "Y",
    "base_item_number": "PROD-",
    "specifications": [
        {
            "serial": 456,
            "name": "紅色 L號",
            "stock": 10,
            "safety_stock": 2,
            "item_number": "R-L",
            "full_item_number": "PROD-R-L"
        }
    ]
}
```

**回應 - 無規格：**
```json
{
    "product_serial": 123,
    "title": "測試產品名稱",
    "has_spec": "N",
    "full_item_number": "PROD-001",
    "stock": 50,
    "safety_stock": 5
}
```

---

### 4.3 更新產品/規格庫存（`update_product_stock`）

支援兩種識別方式（擇一）：
- `product_serial`：無規格主產品 ID
- `spec_serial`：規格 ID
- `full_item_number`：產品全編號（系統自動辨識主產品或規格）

**請求（透過全編號更新）：**
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
        "name": "update_product_stock",
        "arguments": {
            "full_item_number": "P123R-L",
            "new_stock": 99
        }
    }
}
```

**回應：**
```json
{
    "msg": "Specification stock updated",
    "spec_serial": 456,
    "new_stock": 99
}
```

---

### 4.4 批次取得產品庫存（`batch_get_product_stock`）

**請求：**
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
        "name": "batch_get_product_stock",
        "arguments": { "product_serials": [123, 124, 125] }
    }
}
```

**回應：**

以 `product_serial` 為 Key，內部結構與 `get_product_stock` 相同。
```json
{
    "123": { "...單一產品資料..." },
    "124": { "...單一產品資料..." },
    "125": { "...單一產品資料..." }
}
```

---

### 4.5 批次更新庫存（`batch_update_product_stock`）

**請求：**
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
        "name": "batch_update_product_stock",
        "arguments": {
            "updates": [
                { "product_serial": 123, "new_stock": 100 },
                { "full_item_number": "PROD-2-BLUE", "new_stock": 50 }
            ]
        }
    }
}
```

**回應：**

陣列依序對應每筆更新結果。
```json
[
    { "msg": "Product stock updated",      "product_serial": 123, "new_stock": 100 },
    { "msg": "Specification stock updated", "spec_serial": 456,    "new_stock": 50  }
]
```

---

### 4.6 訂單列表查詢（`list_orders`）

每次最多回傳 50 筆，預設不含商品明細；傳入 `include_items=true` 可一併回傳，免再逐筆呼叫 `get_order`。

#### 參數

| 參數 | 型別 | 必填 | 說明 |
|---|---|---|---|
| `page` | integer | 否 | 頁碼，預設 1 |
| `status` | string | 否 | 訂單狀態名稱（依各商家設定而異，如「處理中」、「已出貨」）；省略表示不限 |
| `date_from` | string | 否 | 訂單日期起，格式 `YYYY-MM-DD`；省略則不限 |
| `date_to` | string | 否 | 訂單日期迄，格式 `YYYY-MM-DD`；省略則不限 |
| `keyword` | string | 否 | 搜尋訂單編號或收件人姓名 |
| `include_items` | boolean | 否 | 是否一併回傳商品明細，預設 `false`；設為 `true` 時每筆訂單會包含 `items` 陣列，免再逐筆呼叫 `get_order` |

> **注意**：`status` 為各商家自訂的狀態名稱（字串），非固定代碼。可先呼叫 `tools/list` 確認，或依後台「訂單狀態設定」中定義的名稱填入。若填入的名稱不存在，回傳空結果。

**請求（不含明細）：**
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
        "name": "list_orders",
        "arguments": {
            "page": 1,
            "status": "已出貨",
            "date_from": "2026-04-01",
            "date_to": "2026-04-10"
        }
    }
}
```

**請求（含商品明細）：**
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
        "name": "list_orders",
        "arguments": {
            "page": 1,
            "status": "已出貨",
            "date_from": "2026-04-01",
            "date_to": "2026-04-10",
            "include_items": true
        }
    }
}
```

**回應（`result.content[0].text` 解析後，`include_items=true`）：**
```json
{
    "page": 1,
    "per_page": 50,
    "total_count": 120,
    "total_pages": 3,
    "data": [
        {
            "orderno": "20260410001234",
            "order_time": "2026-04-10 14:30:00",
            "status_id": 13,
            "status_name": "已出貨",
            "member_account": "user@example.com",
            "buyer_name": "王小明",
            "buyer_mobile": "0912345678",
            "buyer_email": "user@example.com",
            "receiver_name": "王小明",
            "receiver_mobile": "0912345678",
            "receiver_address": "台北市信義區信義路五段7號",
            "portage": 60,
            "paid_amount": 1560,
            "total_bonus": 0,
            "use_bonus": 0,
            "deliver_time": "2026-04-11 09:00:00",
            "note": "",
            "items": [
                {
                    "item_number": "R-L",
                    "product_name": "測試產品",
                    "spec": "紅色 L號",
                    "quantity": 2,
                    "market_price": 900,
                    "price": 750,
                    "subtotal": 1500,
                    "type": "normal"
                }
            ]
        }
    ]
}
```

> `include_items=false`（預設）時，每筆訂單資料不含 `items` 欄位。

#### 回應欄位說明

| 欄位 | 說明 |
|---|---|
| `orderno` | 訂單編號 |
| `order_time` | 下單時間 |
| `status_id` | 訂單狀態 ID（對應 `cart_order_status.id`） |
| `status_name` | 訂單狀態名稱（依商家設定） |
| `member_account` | 會員帳號（訪客訂單為空字串） |
| `buyer_name` | 購買人姓名 |
| `buyer_mobile` | 購買人電話 |
| `buyer_email` | 購買人 Email |
| `receiver_name` | 收件人姓名 |
| `receiver_mobile` | 收件人電話 |
| `receiver_address` | 收件地址（縣市＋區＋地址合併） |
| `portage` | 運費（元） |
| `paid_amount` | 實付金額（商品金額＋運費＋金流手續費－折抵） |
| `total_bonus` | 本單獲得紅利點數 |
| `use_bonus` | 本單折抵紅利點數 |
| `deliver_time` | 出貨時間（未出貨為空字串） |
| `note` | 備註 |

---

### 4.7 取得單筆訂單完整資訊（`get_order`）

包含購買人、收件人、商品明細、金額明細、紅利點數。

#### 參數

| 參數 | 型別 | 必填 | 說明 |
|---|---|---|---|
| `orderno` | string | 是 | 訂單編號 |

**請求：**
```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
        "name": "get_order",
        "arguments": { "orderno": "20260410001234" }
    }
}
```

**回應（`result.content[0].text` 解析後）：**
```json
{
    "orderno": "20260410001234",
    "order_time": "2026-04-10 14:30:00",
    "status_id": 13,
    "status_name": "已出貨",
    "order_source": "web",
    "member_account": "user@example.com",
    "buyer_name": "王小明",
    "buyer_mobile": "0912345678",
    "buyer_email": "user@example.com",
    "receiver_name": "王小明",
    "receiver_mobile": "0912345678",
    "receiver_address": "台北市信義區信義路五段7號",
    "portage": 60,
    "paid_amount": 1560,
    "total_bonus": 15,
    "use_bonus": 0,
    "deliver_time": "2026-04-11 09:00:00",
    "note": "請勿折疊",
    "items": [
        {
            "item_number": "R-L",
            "product_name": "測試產品",
            "spec": "紅色 L號",
            "quantity": 2,
            "market_price": 900,
            "price": 750,
            "subtotal": 1500,
            "type": "normal"
        },
        {
            "item_number": "ADDON-01",
            "product_name": "加購品",
            "spec": "",
            "quantity": 1,
            "market_price": 200,
            "price": 0,
            "subtotal": 0,
            "type": "add_on"
        }
    ]
}
```

#### 商品明細欄位說明（`items[]`）

| 欄位 | 說明 |
|---|---|
| `item_number` | 商品規格編號 |
| `product_name` | 商品名稱 |
| `spec` | 規格名稱（無規格為空字串） |
| `quantity` | 數量 |
| `market_price` | 市價（元） |
| `price` | 成交單價（元） |
| `subtotal` | 小計（`price × quantity`，元） |
| `type` | `normal`：一般商品；`add_on`：加購商品 |

> 贈品計算項（`type = 3`）不列入回傳結果。

**訂單不存在時的回應：**
```json
{
    "error": "Order not found",
    "orderno": "NOTEXIST"
}
```

---

## 5. 錯誤處理

### 協議層錯誤（`error` 欄位）

| Code | 說明 |
|---|---|
| `-32700` | Parse error：請求 Body 非合法 JSON |
| `-32600` | Invalid Request：`jsonrpc` 欄位不為 `"2.0"` |
| `-32601` | Method not found：`method` 或工具名稱不存在 |
| `-32001` | Unauthorized：API Key 無效或未提供 |
| `-32029` | Too Many Requests / Quota Exceeded：超過每分鐘 rate limit、每日呼叫上限或每月呼叫上限 |
| `-32603` | Internal Error：MCP API 功能已停用 |

### 工具執行錯誤（`isError: true`）

工具本身的業務邏輯錯誤（如找不到產品、缺少參數），回應結構仍為 `result`，但帶有 `"isError": true`：

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "content": [
            { "type": "text", "text": "{\"error\": \"Product has specifications. Please provide spec_serial to update.\", \"product_serial\": 123}" }
        ],
        "isError": true
    }
}
```

---

## 6. 在 Claude Code 中設定 MCP Server

編輯 `~/.claude/settings.json`，加入以下設定即可讓 Claude 直接使用 SmartWeb 工具：

```json
{
    "mcpServers": {
        "smartweb": {
            "type": "http",
            "url": "https://[YOUR_DOMAIN]/mcp/index.php",
            "headers": {
                "Authorization": "Bearer [YOUR_API_KEY]"
            }
        }
    }
}
```

設定完成後重啟 Claude Code，執行 `/mcp` 可確認 `smartweb` server 已載入，並列出 7 個可用工具。

---

## 附錄：訂單狀態名稱

`list_orders` 的 `status` 參數使用各商家在後台「**訂單狀態設定**」中自訂的名稱，非固定代碼。以下為常見預設名稱（各商家可能不同）：

| 常見狀態名稱 | 說明 |
|---|---|
| 處理中 | 剛成立的訂單，尚未付款確認 |
| 已付款 | 付款已確認 |
| 備貨中 | 倉庫備貨 |
| 已出貨 | 已交付物流 |
| 訂單已取消 | 訂單取消 |
| 交易完成 | 收貨完成 |

> 實際可用的狀態名稱以各商家後台設定為準。若不確定，可先以不帶 `status` 參數呼叫 `list_orders`，從回傳資料的 `status_name` 欄位確認各訂單的實際狀態名稱。
