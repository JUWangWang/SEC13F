# 手動更新 Form 4

本專案同時保留13F機構持股與Form4內部人交易。新聞專案每日07:00的工作只同步這裡已發布的成果，不會重跑SEC擷取；更新網站程式也不會重跑。

## GitHub手動補跑

Actions → **Generate Form4 (manual only)** → Run workflow → main。

這項工作只執行Form4，使用13F_Ticker_Universe.xlsx中Active=Y的股票，按台北日期分析最近30個日曆日；不執行13F_FINAL.py，也沒有自動排程。提交程式碼與成果不會觸發此擷取工作。

成功後，工作會將Excel與執行版本metadata提交到insider_form4/YYYY-MM-DD/，並保留Actions下載成果。若所有股票擷取都失敗，工作會失敗，不提交空成果。

若要立即顯示於新聞網站，接著到RMSecfb/negative-news的Actions，執行 **Institutional and Insider Results Sync**；或等待下次07:00同步。網站重新查詢新聞、重新讀取機構內部人成果後即可看到更新。

## 本機執行

安裝requirements-form4.txt，再執行 `python run_form4.py`。這個入口使用專案相對路徑及原程式既有SEC Identity。原始Form4_FINAL.py保留，run_form4.py以語法樹調整執行設定。

目前測試套件為edgartools 5.62.1、pandas 3.1.0、openpyxl 3.1.5。SEC請求上限設4次／秒。原始申報查詢明確設amendments=False；單獨4/A查詢使用amendments=True，避免套件將4/A字串移除而錯數成原始申報。4/A只揭露數量及待覆核提示，不直接加入P/S。

P/S沿用原專案market_trades、主體核對、交易日窗口、移除重複及NPR定義。Excluded Transactions改用原始non_derivative_table／derivative_table逐筆Date，不用to_dataframe的reporting_date當交易日。10b5-1仍保留、不排除；Form4/A未做完整修正版對帳。有修正版或其他解析／主體品質問題時，新聞頁面會顯示待覆核。

資料快取可重用已取得的SEC內容，但每次手動執行仍會重新計算當次30日窗口；這不等於自動增量同步SEC。13F是另一個季度流程，這個入口不會更新它。
