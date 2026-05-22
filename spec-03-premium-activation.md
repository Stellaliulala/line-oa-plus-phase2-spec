# 開發規格文件 #03：付款訂閱啟用進階版（Premium Activation）

**版本**：1.1 | **日期**：2026-05-22 | **依據文件**：DX Solution Phase 2 Product Guide § 7–13、§ 18；技術文件 v2.0.0 § Open API

---

## 1. 功能概述

當使用者完成付費訂閱後，12CM 系統須根據 OA Plus 平台回傳的 Subscription 狀態與 Plugin Level，開通對應的付費功能權益（進階版），並在訂閱失效時即時停止相關功能存取。

> **12CM 的角色：被動接收，不主動追蹤**
> 12CM 不需主動查詢訂閱狀態，只需根據 LINE 平台透過 Webhook 推送的事件同步調整系統可用版本。用量等級變更、付費版啟用、降級、寬限期進入與到期，均由 LINE 平台判斷後通知。

---

## 2. 訂閱狀態模型

### 2.1 核心狀態欄位

| 欄位 | 可能值 | 說明 |
|------|--------|------|
| `activatedStatus` | `ACTIVATED` / `ACTIVATION_INITIATED` / `ACTIVATION_FAILED` | 服務是否已啟用 |
| `pluginLevel` | `FREE` / `PREMIUM` | 目前使用的方案層級 |
| `subscriptionStatus` | `active` / `trialing` / `inactive` | Subscription 是否有效。`trialing` 為 Trial 期間專屬狀態，視同 `active`，進階版功能有效 |

### 2.2 狀態轉換說明

| 觸發事件 | activatedStatus 變化 | pluginLevel 變化 | 備註 |
|---------|---------------------|----------------|------|
| 使用者發起訂閱 | → `ACTIVATION_INITIATED` | 維持 `FREE` | 等待付款結果 |
| Trial 授權成功 | 待向 LINE 技術窗口確認 | → `PREMIUM` | Subscription → `TRIALING`；進階版立即啟用，Trial 期間（30 天）視同訂閱中 |
| Trial 到期首次扣款成功 | → `ACTIVATED` | 維持 `PREMIUM` | 正式訂閱開始，收到 `PAYMENT_SUCCEEDED` |
| 付款成功（無 Trial） | → `ACTIVATED` | → `PREMIUM` | 進階版啟用 |
| 付款失敗（無 Trial） | → `ACTIVATION_FAILED` | 維持 `FREE` | 使用者需重新訂閱 |
| Trial 到期首次扣款失敗 | → `ACTIVATED`（寬限期）→ `EXPIRED` | → `FREE`（寬限期結束後） | 進入寬限期（主狀態 `ACTIVATED`），retry 仍失敗後降回免費版 |
| 寬限期 retry 仍失敗 | → `EXPIRED` | → `FREE` | 功能立即停止 |
| 使用者取消（到期後） | → `EXPIRED` | → `FREE` | 本期結束前仍 `ACTIVATED` |

---

## 3. Webhook 事件處理

### 3.1 ACTIVATION_CHANGED

**觸發時機**：activation 狀態或 plugin level 發生變更時。

**12CM 處理步驟**：
1. 驗證 `X-DX-Signature`
2. 呼叫 **Retrieve Subscription V2 API** 取得最新 Subscription 狀態與 activation 詳細資訊
3. 依查詢結果同步 12CM 系統中的服務權益

> Webhook 僅作為事件通知，不直接提供完整訂閱明細，**必須**另行查詢 API。

---

## 4. Open API 查詢

| API | 說明 | Swagger |
|-----|------|---------|
| Retrieve Subscription V2（推薦） | 查詢 Subscription 狀態 + activation 詳細資訊 + plugin level | [Staging Swagger](https://vos.line-scdn.net/oa-plus-dx-swagger-ui-web/index.html?urls.primaryName=Staging#/Plugin's%20Subscription%20Status%20and%20Activation%20Details/getPluginSubscriptionStatus) |
| Retrieve Subscription（舊版） | 查詢訂閱詳細資料 | [Swagger](https://vos.line-scdn.net/oa-plus-dx-swagger-ui-web/index.html#/Plugin%20Subscription/getPluginSubscriptionStatus) |

> Response Schema 請直接參閱 Swagger，欄位定義由 LINE 維護，不在本文件內嵌。

---

## 5. 寬限期（Grace Period）說明

| 項目 | 說明 |
|------|------|
| 寬限期長度 | 約 **7–10 天**，由 LINE 平台決定，12CM 無需設定 |
| 期間訂閱狀態 | `ACTIVATED`（維持不變，使用者仍可使用付費功能） |
| retry 機制 | 由 LINE 平台自動執行，12CM 無需介入 |
| retry 成功 | 訂閱恢復正常，LINE 發送 `PAYMENT_SUCCEEDED` |
| 寬限期結束仍失敗 | 訂閱轉為 `EXPIRED`，LINE 發送 `ACTIVATION_CHANGED`，12CM 據此停止付費功能 |

---

## 6. 權益控管邏輯

12CM 須**同時**判斷三個維度：OA Paid/Unpaid × OA 用量等級（輕用量/中高用量）× DX Subscription 狀態（active/inactive）

### 6.1 用量等級說明

LINE OA 帳號分為**輕用量**與**中高用量**兩種等級（Phase 1 既有概念）。用量等級變更由 LINE 透過 Webhook 通知，12CM 已有對應處理。

| 用量等級 | 免費版功能範圍 | 備註 |
|---------|-------------|------|
| 中高用量 | 全功能開放 | 正常免費版 |
| 輕用量 | 優惠券**不可新增/編輯**，其餘功能正常 | Phase 1 限制，若升級付費版則跳過此限制 |

### 6.2 完整權益判斷矩陣

| OA 付費狀態 | OA 用量等級 | DX Subscription | 12CM 系統狀態 |
|-----------|-----------|----------------|-------------|
| Paid | 中高用量 | Active（PREMIUM） | ✅ 完整付費功能 |
| Paid | 輕用量 | Active（PREMIUM） | ✅ 完整付費功能（跳過輕用量限制） |
| Paid | 中高用量 | Inactive / EXPIRED | 降回正常免費版（全功能可用） |
| Paid | 輕用量 | Inactive / EXPIRED | ⚠️ 回到 Phase 1 限制（優惠券不可新增/編輯）並顯示輕用量升級提示 |
| Unpaid | 任何 | 任何 | ⛔ 限制功能並提示升級 OA 付費方案 |

> **輕用量用戶取消付費後**：訂閱到期或取消且本期結束後，若仍為輕用量，需回到 Phase 1 限制狀態並顯示輕用量升級提示，而非降回一般免費版。

---

## 7. 付費版啟用成功判斷條件

收到 `ACTIVATION_CHANGED` 並查詢後，須滿足以下**全部條件**才視為進階版啟用成功：

- `activatedStatus` = `ACTIVATED`
- `pluginLevel` = `PREMIUM`
- Subscription `subscriptionStatus` = `active` 或 `trialing`

> Trial 期間 `subscriptionStatus = trialing` 視同 `active`，進階版有效。
>
> ⚠️ **待確認**：Trial 授權成功後 `activatedStatus` 的實際值，Product Guide §8.2 只說「Subscription 進入 TRIALING」，未明確說明此時 `activatedStatus` 的值。實作前請向 LINE 技術窗口確認。

---

## 8. Idempotency 處理

12CM 發送事件時，須帶入唯一 `eventId` 作為 idempotency key：

**建議格式**：`{merchantId}:{orderId}:{eventType}:{timestamp}`

範例：`abc123:12345:Buy:1752122552`

---

## 9. 錯誤處理

| 情境 | 建議處理 |
|------|---------|
| API 回傳 429 | 實作指數退避重試 |
| Webhook 驗簽失敗 | 回傳 401，記錄日誌，不處理事件 |
| 查詢 API 失敗 | 保留當前狀態，排入重試佇列 |
| 所有重試失敗 | 告警通知，人工確認 |

---

## 10. 開發與測試

LINE 提供兩種方案等級的測試帳號：

| 帳號類型 | 用途 |
|---------|------|
| Light | 驗證免費版行為及功能限制情境 |
| Medium / High | 驗證付費版（PREMIUM）行為與升降級情境 |

Development Mode 目前僅支援 RC（Staging）環境（`https://oaplus-staging.line.biz/`），Production 不支援。

---

*本規格以 LINE OA Plus TW DX Solution 技術文件 v2.0.0 及 DX Solution Phase 2 Product Guide 為準。*
