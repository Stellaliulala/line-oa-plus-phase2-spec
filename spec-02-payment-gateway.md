# 開發規格文件 #02：藍新金流整合

**版本**：1.0 | **日期**：2026-05-22 | **依據文件**：DX Solution Phase 2 Product Guide § 6.1、§ 14.1

---

## 1. 功能概述

12CM 是藍新金流的**收款方（Merchant）**，須完成藍新金流帳號申請與綁定，才能在 OA Plus 上支援付費 Plugin。

金流操作（授權、請款、退款、自動續扣、retry）全部由 **OA Plus 平台代為執行**，12CM 不直接呼叫藍新 API。12CM 的角色是透過 Webhook 與 Open API 取得付款結果，並據此處理內部權益開通與發票流程。

---

## 2. 責任範圍劃分

| 角色 | 負責項目 |
|------|---------|
| LINE OA Plus 平台 | 建立 Subscription、建立 Payment、導向藍新付款頁、接收藍新付款結果、管理自動續扣時序、執行 retry / 寬限期、更新 Subscription 狀態、發送 Webhook 通知 12CM |
| 藍新金流 | 信用卡授權、請款 / 補扣、退款處理、回傳付款結果給 LINE |
| 12CM（Partner） | 申請藍新帳號（收款方）、綁定帳號至 OA Plus、接收 Webhook 通知、查詢付款結果（Open API）、判斷是否開票、判斷是否啟用進階版 |

> **12CM 不直接呼叫藍新 API**：所有藍新金流操作均由 LINE 平台代為處理。12CM 只需完成帳號申請與綁定，後續僅需處理 LINE 發出的 Webhook 事件。

---

## 3. 前置條件

> ⛔ **開發封鎖警告**：以下項目須全部完成，工程師才能進行整合測試。

| 項目 | 說明 | 負責方 | 開發依賴 |
|------|------|--------|---------|
| 藍新金流帳號申請 | 12CM 需自行向藍新申請收款帳號 | 12CM | ⛔ 須完成 |
| 藍新審核通過 | 審核通過後付費 Plugin 才能上架 | 藍新 | ⛔ 須完成 |
| 金流帳號綁定 | 在 OA Plus 後台完成綁定設定 | 12CM + LINE Ops | ⛔ 須完成 |
| Phase 2 Plugin Id / Secrets 核發 | LINE 核發 Plugin Id、Secret of Admin Plugin、Secret of Issue Auth Token | LINE | ⛔ 須完成 |
| Webhook URL 設定完成 | `ACTIVATION_CHANGED` 與 `PAYMENT_SUCCEEDED` 兩支 Webhook URL 已在 OA Plus 完成設定 | 12CM + LINE | ⛔ 須完成 |
| Open API 白名單 IP 登記 | 向 LINE 提交 12CM 伺服器 IPv4 CIDR | 12CM + LINE | 整合測試前須完成 |
| Staging Open API 環境上線 | 目前 Not ready, in progress | LINE | 整合測試前須完成 |

---

## 4. 付款流程架構

```
使用者在 Checkout 填寫資訊並同意條款
    ↓
OA Plus 平台建立 Subscription
    ↓
OA Plus 平台建立 Payment，導向藍新付款頁（Trial 情境為授權頁）
    ↓
使用者完成付款 / 授權
    ↓
藍新回傳付款結果至 OA Plus
    ↓
OA Plus 更新 Subscription 狀態，發送 PAYMENT_SUCCEEDED 與 ACTIVATION_CHANGED Webhook
    ↓
12CM 收到 Webhook → 查詢 Open API → 判斷開票 / 啟用進階版
```

---

## 5. Trial（試用）訂閱的付款差異

| 情境 | 付款行為 | 藍新動作 |
|------|---------|---------|
| 無 Trial 一般訂閱 | 立即向使用者請款 | 授權 + 立即請款 |
| 含 Trial 訂閱 | 僅進行付款方式**授權**，不立即扣款；Trial 到期後建立第一期帳單並正式扣款 | 授權綁卡（不請款）→ Trial 到期後請款 |

> Trial 期間為 **1 個月（30 天）**，由 LINE 依確認天數設定。授權→請款的時序均由 LINE 平台自動管理，12CM 無需額外處理。

---

## 6. 自動續扣機制

- Subscription 有效且到達當期結束點時，平台**自動發起**續扣，每月同一天執行
- 12CM 無需主動觸發，只需監聽後續 Webhook

| 續扣結果 | 平台行為 | 12CM 收到 |
|---------|---------|---------|
| ✅ 成功 | Subscription 維持有效，建立下一期計費紀錄 | `PAYMENT_SUCCEEDED` |
| ❌ 失敗 | 進入寬限期（Grace Period），Subscription 主狀態維持 `ACTIVATED` | （無立即通知，等 retry 結果） |

---

## 7. 扣款失敗與寬限期

| 階段 | Subscription 狀態 | 使用者權益 | 12CM 動作 |
|------|-----------------|---------|---------|
| 付款失敗，進入寬限期 | `ACTIVATED`（不變） | ✅ 仍可使用 | 無需動作 |
| 寬限期內 retry 成功 | `ACTIVATED`（繼續） | ✅ 正常 | 收到 `PAYMENT_SUCCEEDED` → 開票 |
| 寬限期到期，所有 retry 仍失敗（約 7–10 天，由 LINE 決定） | `EXPIRED` | ❌ 切回免費版 | 收到 `ACTIVATION_CHANGED` → 停止付費權益 |

**Webhook retry 時序（LINE → 12CM）**：5xx / 429 最多重試 3 次（1s → 2s → 4s），全部失敗後標記為 failed。

---

## 8. 付款方式更新

- 有效訂閱期間，**Admin 角色**可更新付款方式（綁定新卡）
- 更新後僅影響**未來**續扣或補扣，不影響已完成的歷史扣款
- 12CM 系統**無需針對付款方式更新做任何處理**

---

## 9. 12CM 需實作項目

| 項目 | 說明 |
|------|------|
| 接收 `PAYMENT_SUCCEEDED` Webhook | 驗證 `X-DX-Signature`，解析 payload，取得 `paymentOrderId` |
| 呼叫 Retrieve Payment Open API | 取得付款快照，確認付款結果，作為後續開票依據（詳見 Spec 01） |
| 接收 `ACTIVATION_CHANGED` Webhook | 訂閱啟用或失效（EXPIRED）時，同步更新 12CM 系統中的服務權益（詳見 Spec 03） |
| 訂閱狀態同步 | 根據 Subscription 狀態與 plugin level 判斷商戶是否具備付費功能存取權（詳見 Spec 03 §6） |

> Webhook 僅為事件通知，不含完整明細。收到 Webhook 後，必須再呼叫對應 Open API 查詢完整資訊後再處理。

---

## 10. Webhook 回應規範

| Response Status | OA Plus 行為 |
|----------------|------------|
| `2xx` | Success，不重試 |
| `4xx` | Permanent failure，不重試 |
| `5xx` 或 `429` | Transient failure，retry 最多 3 次（1s → 2s → 4s） |

---

## 11. Open API 端點

| 環境 | Endpoint | Rate Limit | 狀態 |
|------|----------|-----------|------|
| Staging | `https://dx-open-api-staging.line-apps.com` | 200 RPS | Not ready |
| Production | `https://dx-open-api.line-apps.com` | 800 RPS | Not ready |

---

## 12. 注意事項

- 禁止在 Staging / Production 環境進行壓力測試
- Open API 僅允許 **IPv4 CIDR 格式**的白名單 IP 存取，需提前向 LINE 登記
- Auth Token 有效期為 **5 分鐘**，需在 token 過期前重新取得
- Reverse Proxy 不支援 3xx redirect（除 304 Not Modified），後端不可回傳 redirect 回應

---

*本規格以 LINE OA Plus TW DX Solution 技術文件 v2.0.0 及 DX Solution Phase 2 Product Guide 為準。*
