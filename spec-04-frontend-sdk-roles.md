# 開發規格文件 #04：前端 SDK 串接與角色權限控管

**版本**：1.1 | **日期**：2026-05-22 | **依據文件**：技術文件 v2.0.0 § LINE OA Plus TW SDK；Phase 2 Product Guide § 5

---

## 功能規格

---

## 1. 功能概述

12CM 的 Admin Plugin 透過 LINE OA Plus TW SDK（`dx` 物件）取得當前登入使用者的商戶資訊、Plugin 狀態及角色，並根據這些資訊動態調整 UI 語言、功能可見範圍與操作權限，確保 OA Plus 與 12CM 系統之間的角色體驗一致。

---

## 2. 角色控管

### 2.1 LINE 規範的角色邊界

LINE OA Plus 定義兩種角色，**僅規範「訂閱與帳務」相關操作的權限邊界**，其他功能的權限由 12CM 自行決定。

| 角色 | 訂閱與帳務操作 |
|------|-------------|
| **Admin** 管理者 | 發起訂閱、更新付款方式、取消/恢復訂閱、查看訂閱清單與扣款紀錄 |
| **Operator** 操作員 | 僅可查看訂閱清單、訂閱狀態、扣款紀錄；不可發起訂閱、不可修改付款資料 |

> LINE 透過 SDK 提供角色資訊「供 Partner 參考」，訂閱與帳務以外的功能由 12CM 自行定義 Admin 與 Operator 各自能操作的範圍。

> ⚠️ **角色驗證前後端都需實作**：前端負責根據角色控制 UI 可見性（show/hide）；後端 API 同樣需驗證角色，防止 Operator 繞過前端直接呼叫受限 API。兩層驗證缺一不可。

### 2.2 12CM 系統的 Admin / Operator 權限設計

Admin 對所有模組具備完整權限（查看、新增、編輯、刪除）。Operator 所有模組均不可執行刪除操作。

| 功能模組 | Admin | Operator |
|---------|-------|---------|
| 歡迎頁 | 查看 新增 編輯 刪除 | 查看 新增 編輯 |
| 會員權益 | 查看 新增 編輯 刪除 | 查看 新增 編輯 |
| 會員卡 | 查看 新增 編輯 刪除 | 查看 新增 編輯 |
| 集點卡 | 查看 新增 編輯 刪除 | 查看 新增 編輯 |
| 優惠券 | 查看 新增 編輯 刪除 | 查看 新增 編輯 |
| 儲值卡 | 查看 新增 編輯 刪除 | 查看 新增 編輯 |
| 會員管理 | 查看 新增 編輯 刪除 | 查看 新增 編輯 |
| 門店管理 | 查看 新增 編輯 刪除 | 查看 新增 編輯 |
| 下載區 | 查看 新增 編輯 刪除 | 查看 新增 編輯 |
| 使用手冊 | 查看 新增 編輯 刪除 | 查看 新增 編輯 |

### 2.3 角色判斷建議邏輯

```javascript
// role 欄位來源待後續確認（請洽 LINE 技術窗口）
if (userRole === 'ADMIN') {
  showSubscriptionControls(); // 訂閱、取消、付款方式更新
  // + 12CM 自定義的 Admin 專屬功能
} else {
  hideSubscriptionControls();
  showReadOnlyView();          // 訂閱狀態唯讀
  // + 12CM 自定義的 Operator 可用功能
}
```

---

## SDK 串接 — 技術參考

---

## 3. SDK 載入

| 環境 | URL |
|------|-----|
| Staging | `https://vos.line-scdn.net/dx-sdk-staging/v2.0.0/bundle.min.js` |
| Production | `https://vos.line-scdn.net/dx-sdk/v2.0.0/bundle.min.js` |

```html
<script src="https://vos.line-scdn.net/dx-sdk-staging/v2.0.0/bundle.min.js" async></script>
```

---

## 4. API 一覽

| 函式 | 類型 | 說明 |
|------|------|------|
| `dx.getUser()` | Async（Promise） | 取得商戶 ID 與語言設定 |
| `dx.getPluginInfo()` | Sync（Object） | 取得 Plugin 方案等級與啟用狀態 |
| `dx.security.getCSRFToken()` | Sync（String） | 取得 CSRF Token，用於 POST/PUT/DELETE |
| `dx.security.getCSRFTokenHeaderName()` | Sync（String） | 取得 CSRF Header 名稱（固定為 `DX-CSRF-TOKEN`） |

---

## 5. `dx.getUser()` — 商戶資訊

| 欄位 | 類型 | 說明 |
|------|------|------|
| `merchantId` | String | 商戶 ID（例如 `@abc1234`） |
| `language` | String | 使用者語系，支援：`zh-TW`、`en`、`ja`、`ko`、`th`、`id` |

**錯誤碼**：`TIMEOUT`（5,000 ms 超時）、`EXCEPTION`（非預期 runtime 錯誤）

---

## 6. `dx.getPluginInfo()` — Plugin 狀態

| 欄位 | 類型 | 可能值 | 說明 |
|------|------|--------|------|
| `level` | String | `FREE` / `PREMIUM` | 目前方案等級 |
| `activatedStatus` | String | `ACTIVATED` / `ACTIVATION_INITIATED` / `ACTIVATION_FAILED` | Plugin 啟用狀態 |

---

## 7. CSRF 安全機制

所有非 GET 請求（POST / PUT / DELETE）**必須**帶入 CSRF Token：

```javascript
const csrfToken = dx.security.getCSRFToken();
const csrfHeaderName = dx.security.getCSRFTokenHeaderName(); // "DX-CSRF-TOKEN"

fetch('/api/your-endpoint', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', [csrfHeaderName]: csrfToken },
  body: JSON.stringify(payload)
});
```

---

## 8. DX-JWT 後端驗證

OA Plus 反向代理會在每個請求的 cookie 中注入 `dx-jwt`，12CM 後端須驗證此 JWT。

**JWT Payload 範例**：
```json
{
  "pluginId": "12345",
  "merchantId": "123456",
  "pluginInfo": { "level": "FREE", "activatedStatus": "ACTIVATED" },
  "iat": 1717023000,
  "exp": 1717026600
}
```

**驗簽方式**：使用 LINE 提供的 **Secret of Admin Plugin** 以 HMAC-SHA256 驗簽。

---

## 環境限制 — 技術參考

---

## 9. API 請求路由規則

- 所有 API 請求**必須**透過反向代理發送，不得直接呼叫 12CM 後端域名
- 請求 URL 必須使用與 Plugin 頁面相同的 domain
- Reverse Proxy **不支援** 3xx redirect（除 304 Not Modified），後端不可回傳 redirect 回應

---

## 10. UI/UX 實作要點

- 語系支援：`zh-TW`（預設）、`en`，呼叫 `dx.getUser()` 取得後動態切換
- 裝置支援：Mobile、Tablet（≥ 768px）、Desktop
- 瀏覽器：Chrome 119+ 完整支援；Safari 部分支援（可能需停用「防止跨站追蹤」）

---

## 11. iFrame 沙盒限制

Admin Plugin 在 iFrame 中執行，sandbox 僅允許：

- `allow-same-origin`：同源 cookie、localStorage、IndexedDB 操作
- `allow-scripts`：允許執行腳本

---

## 12. CSP 合規要求

12CM Plugin 頁面須遵守平台的 Content Security Policy。所有外部資源須在 ACL 中明確定義，動態注入的 `<script>` 與 `<link>` 標籤須包含有效 nonce，否則將被 CSP 封鎖。

---

*本規格以 LINE OA Plus TW DX Solution 技術文件 v2.0.0 及 DX Solution Phase 2 Product Guide 為準。*
