# PRD：Deposit by Account (Confirmed) 收款帳戶確認彙總報表

**版本**：v1（2026-06-29）　**類型**：新功能（報表）　**負責**：PM / RD
**Mockup**：https://guswei.github.io/spay-bo-mockups/deposit-account-report_mockup.html
**相關**：（Redmine 單號待補）

## 1. 需求背景

營運需要查看「特定 Prefix、特定日期區間內，各收款帳戶實際收到多少錢、幾筆」。現況沒有單一頁面能依收款帳戶彙總「已確認入帳」的存款。本案新增一頁唯讀報表：依 Prefix + 確認時間區間，列出每個收款帳戶的 confirmed 存款總額與筆數，並可點列展開該帳戶的逐筆明細。本案只做收款（deposit）方向，不含提款。

## 2. 核心功能變更

| # | 變更 (FR) | 說明 |
|---|---|---|
| FR-1 | 收款帳戶彙總查詢 | 依 `prefix` + 確認時間區間，將 confirmed 存款 `GROUP BY 收款帳號`，輸出 Bank Code / Receive Account Name / Receive account no. / Total Amount / Total Count，並附合計列 |
| FR-2 | 逐筆明細 drill-down | 點任一帳戶列，inline 同頁展開該帳戶在「同一查詢條件」下的每筆 confirmed 存款；明細分頁沿用站內共用 `Pagination.vue`（每頁 10） |
| FR-3 | 匯出 CSV | 將目前查詢結果（彙總欄位 + 合計列）匯出為 CSV |
| FR-4 | Prefix 範圍權限 | 依 4-tier 控制可查詢的 Prefix 範圍 |

範圍：IN — 新增報表頁「Deposit by Account (Confirmed)」（查詢列、彙總表、inline 明細、CSV 匯出）。OUT — 提款／出款；既有 Deposit List、Transaction Records、ChannelDepositReport 不在本案修改範圍。

## 3. 介面設計

以 Mockup 為準：https://guswei.github.io/spay-bo-mockups/deposit-account-report_mockup.html 。重點：

- 查詢列：Prefix（下拉）、From、To（日期）、Search、Export CSV；上方標明「依確認入帳時間（confirmed/settlement time）· GMT+7」篩選。
- 彙總表：每個收款帳戶一列，欄位 Bank Code｜Receive Account Name｜Receive account no.｜Total Amount｜Total Count of transaction；預設依 Total Amount 由大到小；底部合計列（帳戶數、金額與筆數總和）。
- 明細：點帳戶列向下 inline 展開逐筆表（Confirmed Time｜Order No.｜Deposit Account Name｜Payer Account｜Matched Txn No.｜Source｜Amount），每頁 10 筆、可翻頁。
- 空結果：顯示 empty state，不顯示合計列。

**BO 欄位表**：

| 欄位 | 型別/精度 | 必填 | 說明 / enum / 邊界 |
|---|---|---|---|
| prefix | `enum` | 是 | 單選；可選範圍依角色權限（見第 6 節）；前端＋後端都要擋越權 |
| from / to | `DATE` | 是 | 篩「確認入帳時間」；含端點 `BETWEEN from 00:00:00 AND to 23:59:59`；時區 Asia/Bangkok (GMT+7)；須 `from ≤ to`（前端＋後端擋） |
| bankCode | `enum` | — | 收款方銀行代碼，取自 banks 表（顯示欄） |
| accountName | `VARCHAR` | — | 收款帳戶名（顯示欄） |
| accountNo | `VARCHAR` | — | 收款帳號；彙總分組鍵 |
| Total Amount | `DECIMAL(18,2)` | — | `SUM(amount)`；幣別 THB；禁用 float |
| Total Count | `INT` | — | `COUNT(*)`；計入 Confirmed 集合的筆數 |
| (明細) confirmed_at | `DATETIME` | — | 該筆確認入帳時間（對應現有 `updateTime`） |
| (明細) orderNo | `VARCHAR` | — | 存款單號 |
| (明細) amount | `DECIMAL(18,2)` | — | 該筆金額 |
| (明細) depositAccountName | `VARCHAR` | — | 存款方戶名（付款方帳戶戶名，Deposit List 既有欄位） |
| (明細) counterAccountNo | `VARCHAR` | — | 付款方帳號 |
| (明細) matchedTransactionNo | `VARCHAR` | — | 比對到的交易序號 |
| (明細) source | `enum` | — | 確認來源：Flow A 自動比對（`MATCHED`）／Flow B CH App 確認（`CONFIRM`/`CONFIRMED`） |

## 4. 資料模型

N/A —— 本案無 schema 變更。報表所需欄位（`prefix`、`bankCode`、`accountName`、`accountNo`、`amount`、`status`、確認時間 `updateTime`）皆已存在於現有 deposit order；本案為唯讀彙總。金額顯示與儲存精度對應 `DECIMAL(18,2)`。

| Table | 欄位 | 型別 | 約束 |
|---|---|---|---|
| deposit_order（既有，無異動） | amount | `DECIMAL(18,2)` | 顯示/彙總精度約束，禁 float |

## 5. 流程圖

N/A —— 無狀態流（唯讀查詢彙總，不變更任何狀態）。動態行為：① 送出查詢 → 後端依 `prefix`＋確認時間區間＋Confirmed 集合過濾 → `GROUP BY accountNo` 回傳彙總；② 點帳戶列 → 以「同一查詢條件」取該帳戶逐筆明細並 inline 展開（分頁 10／頁）。

## 6. 選單位置

| 選單路徑 | 角色 / 權限 | 說明 |
|---|---|---|
| Reports > Deposit by Account (Confirmed) | L0（超管 / Administrator） | 可查詢任一 Prefix |
| Reports > Deposit by Account (Confirmed) | L1–L3（一般角色，具報表檢視權） | 只能查詢自己所屬 Prefix；後端強制範圍，越權須擋 |

## 7. 驗收標準（AC）

| AC-ID | 對應 FR | 驗收條件 |
|---|---|---|
| AC-01 | FR-1 | 給定 Prefix + 日期區間，表格每列為一個收款帳號，Total Amount = 該帳號 `SUM(amount)`、Total Count = `COUNT(*)`，且只計 `status ∈ {MATCHED, CONFIRM, CONFIRMED}`、確認時間落在區間者 |
| AC-02（負向） | FR-1 | `status` 為 `PENDING / ASSIGNED / PENDING_REVIEW / EXPIRED / CANCEL / REJECTED / REJECTED_BY_CARDHOLDER` 之任一者，**不得**計入 Total Amount 或 Total Count |
| AC-03（負向） | FR-1 | 以「確認入帳時間」判定是否落在區間；僅建單時間 `orderTime` 落在區間、但確認時間不在區間者，**不得**計入（反之，確認時間在區間者必須計入） |
| AC-04 | FR-2 | 展開任一帳戶列，明細為該帳戶在同一查詢條件下的逐筆 confirmed 存款；明細筆數加總 = 該列 Total Count、金額加總 = 該列 Total Amount |
| AC-05（負向） | FR-2 | 明細加總（金額或筆數）**不得**與該列彙總不一致；不一致即視為缺陷 |
| AC-06 | FR-3 | 匯出 CSV 內容 = 目前查詢結果的彙總欄位與合計列 |
| AC-07 | FR-4 | L0 帳號可於 Prefix 下拉選任一 Prefix 查詢 |
| AC-08（負向） | FR-4 | 一般角色帳號**不得**查詢非所屬 Prefix；即使前端被繞過（直接打 API），後端仍須擋下越權查詢 |
| AC-09（負向） | FR-1 | 金額以 `DECIMAL(18,2)` 彙總與顯示，**不得**因 float 造成尾差；Total Amount 須等於明細逐筆相加 |

## 8. 非功能需求（NFR）

| NFR-ID | 類別 | 需求 |
|---|---|---|
| NFR-REL | 可靠性 | 本報表為唯讀彙總，**不得**寫入或變更任何存款狀態、餘額、額度；不得影響既有 Deposit List / Transaction Records / 金流流程 |
| NFR-DATA | 金額/資料精度 | 金額一律 `DECIMAL(18,2)`、幣別 THB、顯示兩位小數，禁用 float；彙總值與明細逐筆相加須完全一致 |

## 9. 假設與限制（ASM / CST）

| ID | 內容 |
|---|---|
| ASM-1 | 「確認入帳時間」對應現有 deposit order 的 `updateTime`（狀態轉為 confirmed 當下的時間）；RD 須確認該欄即為入帳確認時間並作為時間軸與明細欄位 |
| ASM-2 | 報表的 Bank Code / Receive Account Name / Receive account no. 取自 deposit order 的「收款方」欄位（`bankCode` / `accountName` / `accountNo`）；RD 須確認這三欄在所有 confirmed 單上填值正確且一致 |
| CST-1 | 唯讀報表：不得改動存款狀態機、餘額、額度或任何既有頁面行為 |
| CST-2 | 僅涵蓋收款（deposit）；提款／出款不在本報表 |
| CST-3 | 本報表為獨立新頁，不以 ChannelDepositReport 為基礎或併入該頁 |
