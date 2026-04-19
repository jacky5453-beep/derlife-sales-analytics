# 官網銷售分析系統 — 交接文件

## 系統資訊
- **網址：** https://jacky5453-beep.github.io/derlife-sales-analytics/
- **GitHub Repo：** https://github.com/jacky5453-beep/derlife-sales-analytics
- **Firebase 專案：** `derlife-audit`（共用）
- **類型：** 單一 HTML 檔案 + GitHub Pages 部署
- **MCP Proxy：** https://mcp-proxy-kappa.vercel.app（Vercel，解決 CORS）

---

## 目前狀態

### ✅ 已完成
- [x] HTML 骨架 + 5 個分頁 Tab 切換 + 品牌色系 RWD
- [x] Logo 修正為橫式全名顯示（`h-8 w-auto`）
- [x] Google 登入 + 白名單權限管理（collection: `sales-whitelist`）
- [x] 第一位登入者自動成為 admin
- [x] 權限管理 Tab（admin 可新增/移除白名單）
- [x] 儀表板（營業額/訂單數/客單價/銷售量 KPI + 4 張圖表）
- [x] 商品分析（銷售排行數量&金額 + 帕累托圖）
- [x] 顧客分析（用「姓名」識別顧客，貢獻排行/CLV/購買次數&金額分佈）
- [x] 資料匯入（Excel 拖拽上傳 + 自動欄位對應 + doc ID 去重 + 超時機制）
- [x] 資料管理（瀏覽/搜尋/分頁/批次篩選）
- [x] GitHub Pages 部署 + Actions workflow
- [x] **官網 MCP API 同步**（2026-04-15 新增）
  - 從官網 SmartWeb MCP API 直接撈訂單，免手動匯出 Excel
  - API Key 存 localStorage，透過 Vercel proxy 解決 CORS
  - 手動選日期範圍同步 + 登入後自動同步（超過 24 小時撈最近 7 天）
  - 分頁撈取（每頁 50 筆）+ 進度條 + 同步紀錄
  - 不儲存顧客手機號碼（個資保護）
- [x] **Claude Code 每日排程任務**（daily-sync-orders）
  - 每天早上 9:05 自動同步昨天訂單
  - 需要 Claude Code 保持運行
- [x] **有效訂單狀態過濾**（2026-04-17 新增）
  - 新增 `VALID_ORDER_STATUSES` 白名單：訂單處理中、訂單已付款、備貨中準備出貨、訂單已出貨、商品已送達、理貨中
  - API 同步時自動過濾掉無效訂單（取消/退貨/退款/拒收/門市取貨等），不寫入資料庫
  - 儀表板預設篩選改為「✅ 有效訂單」，KPI 與圖表只計算有效訂單
  - 下拉選單新增「全部狀態（含取消/退貨）」選項，個別無效狀態會加 ⚠️ 標記
  - 解決差異：4/1~4/16 官網後台 648,787 / 363 筆，修正前系統顯示 753,620 / 406 筆（多算了無效訂單）
- [x] **成長率顏色改為台股慣例**（2026-04-20 新增）
  - 年度總覽成長率：▲ 紅色（上漲）、▼ 綠色（下跌），符合台灣股市慣例
  - 位置：`fmtGrowth()` 函式（index.html）
- [x] **修正首次載入儀表板未套用狀態篩選的 Bug**（2026-04-20 修正）
  - 問題：`loadAllData()` 直接 `filteredData = [...allData]`，未排除無效訂單，導致儀表板 4 月營業額 $853,253 與年度總覽 $761,454 不一致（差 $91,799 = 取消/退貨訂單）
  - 修法：改呼叫 `applyFilters()` 套用預設「有效訂單」篩選
  - 影響：儀表板與年度總覽現在結果一致

### ⏳ 待處理
- [ ] **用 API 同步補齊歷史資料** — 目前 Firestore 有 ~19,800 筆（Excel 匯入），可用 API 同步更早期或更完整的資料（注意每日 20,000 寫入限制，或升級 Blaze）
- [ ] **考慮升級 Firebase Blaze 方案** — 免費方案每日 20,000 寫入，大量同步時會碰到上限

---

## 重要技術細節

### Firestore Collections
| Collection | 用途 |
|------------|------|
| `website-orders` | 訂單明細，doc ID = `訂單編號_商品編號` |
| `import-batches` | 匯入/同步批次紀錄（含 `source: 'mcp-api'` 區分來源） |
| `sales-whitelist` | 登入白名單，doc ID = email |

### MCP API 串接架構
```
瀏覽器 (GitHub Pages)
    ↓ fetch POST
Vercel Proxy (mcp-proxy-kappa.vercel.app/api/mcp)
    ↓ 轉發 + Authorization header
SmartWeb MCP API (www.dlsveg.com.tw/mcp/index.php)
    ↓ JSON-RPC 2.0
回傳訂單資料 → 寫入 Firestore
```

### MCP API 資訊
| 項目 | 說明 |
|------|------|
| API 網址 | `https://www.dlsveg.com.tw/mcp/index.php` |
| 使用工具 | `list_orders`（含 `include_items: true`） |
| 用量限制 | 每日 500 次、每月 10,000 次 |
| API Key 管理 | SmartWeb 後台 → MCP API Key 管理 |

### API 資料對應（MCP → Firestore）
| API 欄位 | Firestore 欄位 |
|----------|---------------|
| `orderno` | `orderId` |
| `order_time` | `orderDate`（轉 ISO） |
| `status_name` | `orderStatus` |
| `member_account` | `memberId` |
| `buyer_name` | `memberName` |
| `item.item_number` | `productId` |
| `item.product_name` | `productName` |
| `item.quantity` | `quantity` |
| `item.price` | `unitPrice` |
| `item.subtotal` | `subtotal` |
| `paid_amount` | `orderTotal` |
| `portage` | `shippingFee` |
| `total_bonus` | `bonusEarned` |
| `use_bonus` | `bonusUsed` |

> **注意：** `buyer_mobile`（手機號碼）刻意不同步，避免個資外洩風險。

### 顧客識別邏輯
- `會員編號` 只有 5 個值（DCC001~DCC008），是**銷售通路代碼**，不是顧客
- 改用 `姓名` 識別顧客

### 匯入/同步機制
- 每批 200 筆，批次間延遲 300ms
- doc ID = `orderId_productId`，重複自動覆蓋
- 15 秒超時，連續失敗 5 次自動停止

### 官網訂單狀態
**✅ 有效狀態（計入營收）：**
- 訂單處理中、訂單已付款、備貨中準備出貨、訂單已出貨、商品已送達、理貨中

**⚠️ 無效狀態（不計入營收，系統會自動過濾）：**
- 訂單已取消、訂單退貨處理中、訂單已退貨、訂單已退款、訂單退貨已取消
- 消費者拒絕收件、已到門市、已從門市取貨、門市逾期未取貨

> 定義在 `index.html` 的 `VALID_ORDER_STATUSES` 常數，如需調整直接改該陣列即可。

### Firestore 配額注意
- 免費方案每日限制：20,000 寫入 / 50,000 讀取
- 建議升級 Blaze 方案解除限制

### 相關檔案
| 檔案/專案 | 位置 | 說明 |
|----------|------|------|
| 系統主檔 | `index.html` | 單一 HTML，含所有功能 |
| MCP Proxy | `/Users/jacky/Desktop/claude/claude code/mcp-proxy/` | Vercel 專案 |
| MCP 設定 | `/Users/jacky/.claude/.mcp.json` | Claude Code MCP server 設定 |
| 排程任務 | `/Users/jacky/.claude/scheduled-tasks/daily-sync-orders/` | 每日同步 |
| API 手冊 | `mcp_api_manual.md` | SmartWeb MCP API 完整文件 |
