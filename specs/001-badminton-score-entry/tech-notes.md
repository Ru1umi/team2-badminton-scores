# 技術備註（供 /speckit.plan 的 Technical Context 使用）

> 本檔記錄使用者於 /speckit.specify 階段提供的實作層資訊。
> 依規格撰寫原則，這些細節不寫入 spec.md，於規劃階段正式納入 plan.md。

## 技術堆疊（主要）

- 使用者指定：主要技術棧採 **.NET Framework 4.8**（2026-06-10 指定）
- 含義與約束（供 plan.md Technical Context 採用）：
  - Windows 限定之傳統完整框架（非 .NET Core／.NET 5+），
    部署目標為 Windows Server + IIS
  - Web 層對應 ASP.NET（MVC 5／Web API 2／WebForms 擇一，於規劃階段決定），
    **非** ASP.NET Core
  - C# 語言版本預設為 7.3
  - 與既有備註一致：資料庫連線為 ADO.NET 格式之 SQL Server（見「資料庫」節）
  - 組態管理採 `web.config`／`app.config`：機密以未進版控的外部
    `configSource` 檔或環境變數管理（.NET Framework 無 ASP.NET Core 之
    User Secrets／appsettings.json 機制，「資料庫」節原備註據此修正）
  - OCR（Gemini API）與爬蟲皆以 HTTP REST 呼叫實作即可，
    與 Framework 4.8 相容性無虞

## OCR 影像判讀

- 影像辨識服務：Google Gemini API
- 指定模型：`gemini-3.1-flash-lite`
- 官方文件：<https://ai.google.dev/gemini-api/docs/models/gemini-3.1-flash-lite?hl=zh-tw>
- 用途：手寫紙本成績單拍照／掃描後的智慧判讀，轉為電子數據預填表單
- 注意：API 金鑰由使用者提供（「圖片辨識的 KEY」），實作時以環境變數等
  安全方式管理，不得寫死在程式碼或提交至版控

## 爬蟲自動抓取

- 目標頁面（即時成績）：
  `http://172.105.210.232/liveresultst/liveresultst.html?openid=755276`
- 比對鍵：成績輸入畫面上選定場次的「日期、時間、場次編號」
- 詳細比分位置：來源頁面上的「彈跳視窗」（popup）內，爬蟲需能取得
  該視窗載入的資料
- 流程約束：抓取結果僅作預填，成績輸入人員逐項檢查無誤後才帶入比分
- 風險備註：HTTP（非加密）、IP 位址直連，賽事結束後頁面可能下線；
  早期曾提供另一頁面 `matchesst.html?openid=755276`（賽程清單），
  必要時可作為輔助來源

## 資料庫

- 類型：Microsoft SQL Server（連線字串為 ADO.NET 格式，暗示後端採 .NET）
- 主機：`192.168.66.99`（內網）
- 資料庫：`Badminton_T2_Dev`
- 連線字串名稱（key）：`UMallSport`
- 帳號：`UTK_DevSuperUser`、密碼：**不記錄於版控** —— 完整連線字串存於
  本機 `.omc/secrets.local.md`（.omc/ 已被 .gitignore 排除）
- 實作時以環境變數或未進版控的外部組態檔
  （`web.config` 之 `configSource` 拆分檔）管理
  （原備註之 User Secrets／appsettings.Development.json 為 .NET Core 作法，
  因主要技術棧定為 .NET Framework 4.8 而修正，見「技術堆疊」節）

## 裝置支援

- 桌機為主要情境；手機（直式）與平板瀏覽器須可完成輸入與覆核全流程
  （對應 spec FR-020、SC-008）

## 歷史紀錄

- 2026-06-10：初次提供爬蟲網址 `matchesst.html`，後更正為 `liveresultst.html`；
  OCR 指定 Gemini `gemini-3.1-flash-lite`。
- 2026-06-10：指定主要技術棧為 .NET Framework 4.8；同步修正資料庫機密
  管理備註（改為 web.config 外部 configSource／環境變數）。
