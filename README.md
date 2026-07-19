# SEC 13F & Form 4 Monitoring

以 SEC EDGAR 公開申報資料為來源，透過 Python 與 `edgartools` 建立美股
**機構持股（13F）** 與 **內部人交易（Form 4）** 的輔助監控資料。

本模組定位為在既有「負面新聞資訊監控」下的補充資訊；後續可依Ticker與負面新聞資料合併，供 Dashboard
顯示機構與內部人動向。

**最後異動時間：2026/08/18 08:00（台灣時間，UTC+8）**

------------------------------------------------------------------------

## 1. 專案架構

``` text
D:\Python\SEC13F\
│
├─ 13F_FINAL.py
├─ Form4_FINAL.py
├─ 13F_Manager_List.xlsx
├─ 13F_Ticker_Universe.xlsx
│
├─ institutional_13f\
│  └─ 2026Q2\
│     └─ 13F_2026Q2_Full_Universe.xlsx
│
└─ insider_form4\
   └─ 2026-08-18\
      └─ Form4_2026-08-18_30D_Full_Universe.xlsx
```

### 主要程式

``` text  
  程式                                用途
  ----------------------------------- -----------------------------------
  `13F_FINAL.py`                      取得指定機構的 SEC
                                      13F-HR，整理本季與前季持股變化

  `Form4_FINAL.py`                    取得股票 Universe 最近 30 日 SEC
                                      Form 4，整理內部人 P/S 交易
  -----------------------------------------------------------------------
```

### 共用設定檔

``` text  
  檔案                         用途
  ---------------------------- -------------------------------------------
  `13F_Manager_List.xlsx`      13F 追蹤機構清單、CIK、啟用與納入彙總設定
  `13F_Ticker_Universe.xlsx`   13F 與 Form 4 共用的股票 Universe

------------------------------------------------------------------------
```

## 2. 環境需求

建議使用 Python 3.10 以上版本。

主要套件：

``` bash
pip install edgartools pandas openpyxl
```

程式透過 SEC EDGAR 取得公開申報資料。執行前請於兩支程式中設定 SEC
Identity：

``` python
SEC_IDENTITY = "your_email@example.com"
```

並確認：

``` python
BASE_DIR = Path(r"D:\Python\SEC13F")
```

如專案放置於其他路徑，請同步修改 `BASE_DIR`。

------------------------------------------------------------------------

# 3. 13F 機構持股模組

## 3.1 資料來源

使用 SEC **Form 13F-Holdings Report（機構投資人季度持股申報）**。

程式以 `13F_Manager_List.xlsx` 中：

-   `Active = Y`
-   `Validation Status = OK`

的機構作為候選清單，並依 `Include in Aggregate` 判斷是否納入彙總。

程式會自動以 Manager List 中最常見的 `Report Period` 判定共同目標季度。

------------------------------------------------------------------------

## 3.2 13F 處理邏輯

流程：

``` text
Manager List
    ↓
取得各機構最新 13F-HR
    ↓
確認 Report Period
    ↓
取得本季 holdings
    ↓
取得 previous_holding_report()
    ↓
排除 PUT / CALL
    ↓
僅保留 Ticker Universe
    ↓
Institution × Ticker 彙總
    ↓
本季 vs 前季
    ↓
Ticker Summary / Dashboard Summary
```

### PUT / CALL

若 13F holdings 的 `PutCall` 為：

``` text
PUT
CALL
```

則不納入一般股票持股統計，避免選擇權部位與普通股持股混合計算。

------------------------------------------------------------------------

## 3.3 單一機構 × 股票狀態

依本季 Shares 與前季 Shares 判定：

  Status        定義
  ------------- -------------------
  `NEW`         前季 0、本季 \> 0
  `CLOSED`      前季 \> 0、本季 0
  `INCREASED`   本季 \> 前季
  `DECREASED`   本季 \< 前季
  `UNCHANGED`   本季 = 前季

持股變動率：

``` text
Change % = (Current Shares - Previous Shares) / Previous Shares
```

前季為 0 且本季新建倉時，不計一般 Change %。

------------------------------------------------------------------------

## 3.4 Ticker-level Summary

13F 最終以股票為單位彙整：

-   Current Shares
-   Previous Shares
-   機構持股 QoQ
-   增持家數
-   減持家數
-   新建倉
-   清倉
-   持股不變家數
-   本季持有機構數
-   前季持有機構數
-   淨增持家數
-   持股量方向
-   機構持股方向
-   13F 訊號

### 持股量方向

依追蹤機構 Aggregate Shares QoQ：

``` text
QoQ > 0  → 持股增加
QoQ < 0  → 持股下降
QoQ = 0  → 持股不變
```

### 機構持股方向

比較增持與減持的機構家數：

``` text
增持家數 > 減持家數 → 增持家數較多
增持家數 < 減持家數 → 減持家數較多
相同                  → 增減持家數相同
```

因此 13F
訊號同時保留「持股量」與「機構家數」兩個面向，而非只看單一指標。

------------------------------------------------------------------------

## 3.5 13F 輸出

輸出位置：

``` text
institutional_13f\<Quarter>\
```

例如：

``` text
institutional_13f\2026Q2\13F_2026Q2_Full_Universe.xlsx
```

Excel 包含：

1.  `Ticker Summary`
2.  `Dashboard Summary`
3.  `13F Detail`
4.  `Run Log`
5.  `Manager Universe`
6.  `Ticker Universe`

------------------------------------------------------------------------

# 4. Form 4 內部人交易模組

## 4.1 資料來源

使用 SEC **Form 4**。

Form 4 與 13F 共用：

``` text
13F_Ticker_Universe.xlsx
```

只處理 `Active = Y` 的股票。

------------------------------------------------------------------------

## 4.2 分析期間

預設：

``` python
LOOKBACK_DAYS = 30
```

訊號以 **Trade Date 最近 30 日** 為分析區間。

考量 SEC Filing Date 可能晚於實際 Trade Date，程式另外設定：

``` python
FILING_BUFFER_DAYS = 15
```

因此 Filing 抓取期間會比正式分析期間多往前 15 日，但最終是否納入仍以
**Trade Date** 判斷。

------------------------------------------------------------------------

## 4.3 P / S 方向性交易

正式內部人交易訊號只使用：

  Code   定義
  ------ ----------
  `P`    Purchase
  `S`    Sale

程式使用：

``` python
form4.market_trades
```

取得 P/S 市場交易。

其他非 P/S 交易則透過：

``` python
form4.to_dataframe()
```

保留至 `Excluded Transactions`，例如 A、M、F、G、C 等，但不納入 NPR。

------------------------------------------------------------------------

## 4.4 Issuer Validation

這是 Form 4 清理的重要防呆。

以某個 Ticker 查詢 Form 4 時，該公司也可能因本身是其他公司的 Reporting
Owner，而出現在其他 Issuer 的 Form 4 中。

因此程式會取得：

``` python
form4.issuer.ticker
```

並比較：

``` text
Issuer Ticker == 目前分析 Ticker
```

若不同：

``` text
→ 排除該 Filing
→ Issuer Mismatch Filings +1
```

避免把「公司持有其他公司股票」誤認為「該公司自己的內部人交易」。

若無法辨識 Issuer Ticker，則不直接刪除，但會記錄於：

``` text
Issuer Unknown Filings
```

供後續人工檢查。

------------------------------------------------------------------------

## 4.5 Transaction Dedup

P/S 與 Excluded Transactions 都會進行 transaction-level 去重。

主要辨識欄位包括：

-   Accession Number
-   Trade Date
-   Ticker
-   Issuer Ticker
-   Insider
-   Transaction Code
-   Security
-   Shares
-   Price

避免同一交易因 Filing/XML/DataFrame 結構重複計入。

------------------------------------------------------------------------

## 4.6 NPR

內部人交易方向採用 Net Purchase Ratio：

``` text
NPR = (Buy Count - Sell Count) / (Buy Count + Sell Count)
```

其中：

``` text
Buy Count  = P 交易筆數
Sell Count = S 交易筆數
```

訊號：

       NPR Signal            中文顯示
  -------- ----------------- -------------
      \> 0 Net Buying        近期淨買入
      \< 0 Net Selling       近期淨賣出
       = 0 Neutral           買賣中性
    無 P/S No P/S Activity   無 P/S 交易

另外保留：

``` text
Net Amount = Buy Amount - Sell Amount
```

Net Amount 為補充資訊，不取代 NPR。

------------------------------------------------------------------------

## 4.7 10b5-1

程式保留：

``` text
10b5-1 Plan
10b5-1 P/S Count
```

用於辨識可能屬於預先安排交易計畫的 P/S 交易。

目前版本：

``` text
保留標記
但不排除
也不調整 NPR 權重
```

因此 Dashboard 解讀時，可將 10b5-1 作為內部人交易意圖的補充資訊。

------------------------------------------------------------------------

## 4.8 Form 4/A

Form 4/A 為 Form 4 的 Amendment。

目前版本：

``` text
不直接納入 P/S 計算
```

但會在 `Run Log` 記錄：

``` text
Form 4/A Amendments
```

避免原始 Form 4 與 Amendment 被直接重複加總。

------------------------------------------------------------------------

## 4.9 Form 4 輸出

輸出位置：

``` text
insider_form4\<Run Date>\
```

例如：

``` text
insider_form4\2026-08-18\Form4_2026-08-18_30D_Full_Universe.xlsx
```

Excel 包含：

1.  `Ticker Summary`
2.  `Dashboard Summary`
3.  `Form4 Detail`
4.  `Excluded Transactions`
5.  `Run Log`
6.  `Ticker Universe`
7.  `Methodology`

------------------------------------------------------------------------

# 5. Dashboard 整合定位

13F 與 Form 4 都屬於 **輔助訊號**。

核心原則：

``` text
負面新聞
    ↓
Event Classification
    ↓
Risk Level
    ↓
Action Plan
```

13F / Form 4 則由 Ticker 進行補充：

``` text
                    ┌─ 13F：機構持股動向
Ticker ─────────────┼─ Form 4：內部人交易動向
                    └─ Negative News：事件風險
```

建議 Dashboard 顯示：

  Ticker   Risk   13F              Insider
  -------- ------ ---------------- --------------
  NVDA     L4     ↓ 機構持股下降   ↓ 近期淨賣出
  MSFT     L2     ↑ 機構持股增加   ↑ 近期淨買入

**13F 與 Form 4 不直接改變負面新聞 Risk Level。**

它們的用途是提供事件之外的市場參與者行為資訊，協助風險人員判斷：

-   負面新聞是否同時伴隨機構減持
-   是否出現內部人淨賣出
-   籌碼與事件訊號是否一致
-   是否存在訊號分歧

------------------------------------------------------------------------

# 6. 訊號一致性應用

後續 Dashboard 可依三種資訊形成輔助提示，例如：

``` text
L4/L5 + 13F 偏減持 + Insider 偏賣出
→ 多重負面訊號

L4 + 13F 偏增持 + Insider 無異動
→ 事件風險為主

L0/L1 + 13F 偏減持 + Insider 偏賣出
→ 籌碼面觀察

L3 + 13F 偏增持 + Insider 偏買入
→ 訊號分歧
```

此分類為 Dashboard 的輔助解讀邏輯，不應取代原負面新聞事件的 Risk Level。

------------------------------------------------------------------------

# 7. 執行方式

### 執行 13F

``` bash
cd D:\Python\SEC13F
py 13F_FINAL.py
```

### 執行 Form 4

``` bash
cd D:\Python\SEC13F
py Form4_FINAL.py
```

建議執行後先檢查各 Excel 的 `Run Log`。

Form 4 特別注意：

``` text
Status
Form 4 Filings
Form 4/A Amendments
P/S Transactions
Excluded Transactions
Issuer Mismatch Filings
Issuer Unknown Filings
Parse Errors
```

13F 特別注意：

``` text
Status
Aggregate
Target Period
Actual Period
Previous Period
Detail Rows
Message
```

------------------------------------------------------------------------

# 8. 注意事項

1.  13F 為季度申報資料，並非即時機構持股。
2.  13F 的「持股增加／下降」僅代表本專案追蹤機構 Universe 的彙總結果。
3.  Form 4 的 P/S 僅代表 SEC transaction code 的 Purchase / Sale。
4.  內部人賣出可能受到
    10b5-1、稅務、薪酬、資產配置等因素影響，不應單獨視為公司基本面負面訊號。
5.  Form 4/A 目前僅記錄 Amendment 數量，尚未進行完整 Amendment
    reconciliation。
6.  `Issuer Unknown Filings` 若大於 0，建議人工檢查。
7.  本模組為風險監控與研究用途，不構成投資建議。

------------------------------------------------------------------------

# 9. 後續整合方向

目前資料層已拆分為：

``` text
Negative News Dataset
Institutional 13F Dataset
Insider Form 4 Dataset
```

後續可依 `Ticker` 合併至同一 Dashboard，形成：

``` text
事件風險
+
機構持股方向
+
內部人交易方向
=
多維度風險觀察
```

核心設計仍維持：

> **Risk Level 由負面新聞事件決定；13F 與 Form 4
> 作為獨立輔助觀察訊號。**
