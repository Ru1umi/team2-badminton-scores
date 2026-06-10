# Phase 0 Research: 羽球成績輸入系統

**Date**: 2026-06-10
**Input**: spec.md（v5）、tech-notes.md、使用者於 /speckit.plan 指定：
前端用 HTML、主要技術棧 .NET Framework 4.8 C#、串接 MSSQL。

本文件將 Technical Context 的所有未知數收斂為明確決策；
無遺留 NEEDS CLARIFICATION。

## R-01 Web 應用框架

- **Decision**: ASP.NET MVC 5（System.Web，classic csproj）on .NET Framework 4.8；
  頁面以 Razor 產生 server-rendered HTML。
- **Rationale**: 使用者指定 .NET Framework 4.8 與「前端用 HTML」——
  Razor 直接輸出 HTML、無 SPA 建置鏈，符合工讀生導向的簡單頁面流程
  （FR-013：單一明確操作路徑）。MVC 5 是 Framework 4.8 上最成熟的
  選項，可測試性優於 WebForms。
- **Alternatives considered**:
  - ASP.NET WebForms — 事件模型與 ViewState 難以單元測試，違反憲章原則 II 的隔離要求，否決。
  - 靜態 HTML + SPA（Vue/React）— 引入前端建置鏈與框架學習成本，
    與「前端用 HTML」的指示及本期規模不符，否決。
  - ASP.NET Core — 與指定的 .NET Framework 4.8 不相容，否決。

## R-02 AJAX／JSON 介面形式

- **Decision**: 使用 MVC 5 Controller 的 JsonResult actions 處理 AJAX
  （草稿自動儲存、OCR 上傳、爬蟲預填、覆核操作）；不另外引入 Web API 2。
- **Rationale**: 同一路由與序列化堆疊，少一套設定與套件；
  本期端點數量少（約 10 個），JsonResult 足夠。
- **Alternatives considered**: Web API 2（ApiController）— 內容協商與
  attribute routing 較完整，但對本期規模是多餘的第二套堆疊，否決。

## R-03 前端 UI 與 RWD

- **Decision**: Bootstrap 5 + 原生 JavaScript（必要時 jQuery 3，MVC 5
  驗證腳本本身相依 jQuery）；行動優先版型，大按鈕、單欄流程。
- **Rationale**: FR-020／SC-008 要求桌機、手機（直式）、平板皆可完成
  全流程且不水平捲動；Bootstrap 格線與表單元件以 CDN／本機檔即可使用，
  無建置步驟。FR-013 的大按鈕、即時回饋以其元件實現並維持一致
  （憲章原則 III）。
- **Alternatives considered**: 手寫 CSS — RWD 成本高且易不一致，否決；
  Tailwind — 需建置鏈，否決。

## R-04 資料存取

- **Decision**: Dapper（NuGet，支援 net48）+ Repository 介面 +
  手寫參數化 T-SQL。
- **Rationale**: 連線字串為 ADO.NET 格式（tech-notes），Dapper 直接建立在
  ADO.NET 上；查詢明確可見，易於避免 N+1（憲章原則 IV）；Repository
  介面讓服務層以替身隔離測試（憲章原則 II）。
- **Alternatives considered**:
  - 純 ADO.NET — 大量樣板 mapping 程式碼，可讀性差（原則 I），否決。
  - Entity Framework 6 — 功能完整但對 9 張表的規模偏重，
    且 LINQ 轉譯易隱藏 N+1，否決。

## R-05 資料庫 schema 與版本管理

- **Decision**: 於既有資料庫 `Badminton_T2_Dev` 內建立專屬 schema `bse`
  （badminton score entry）；建表與種子資料以編號 T-SQL 腳本置於
  `database/schema/`、`database/seed/`，手動依序執行。
- **Rationale**: 既有資料庫可能含其他系統物件，schema 隔離避免命名
  衝突、便於整批檢視與移除；本期無自動 migration 需求，編號腳本
  即可追溯。三場示範場次（spec FR-001）以種子腳本建立。
- **Alternatives considered**: DbUp／FluentMigrator — 自動化 migration
  對單一環境、一次性賽事系統屬過度工程，否決。

## R-06 成績單佐證圖檔儲存

- **Decision**: 存入資料庫（`varbinary(max)`），單檔上限 10 MB，
  僅接受 JPEG/PNG/HEIC 轉檔後之 JPEG/PNG。
- **Rationale**: 賽事規模（本期 3 場示範、實務單一賽事數百筆）資料量小；
  與成績同庫使備份還原為單一單元，且避免 IIS 檔案目錄權限與
  路徑失聯問題。
- **Alternatives considered**: 檔案系統 + 路徑欄位 — 需另行處理備份一致性
  與目錄權限，效益在此規模不明顯，否決；FILESTREAM — 過度工程，否決。

## R-07 OCR 影像判讀

- **Decision**: 以 `HttpClient` 直接呼叫 Gemini API REST 端點，模型
  `gemini-3.1-flash-lite`（tech-notes 指定）；請求採 structured output
  （JSON schema）回傳場次資訊、各局比分與逐欄位信心值；信心值低於
  門檻（預設 0.8）之欄位於 UI 標示提醒（FR-009）。API 金鑰存於
  未進版控之外部組態檔。逾時上限 30 秒，失敗時依 FR 以白話訊息
  引導改用人工輸入。
- **Rationale**: 官方 .NET SDK 以 .NET Standard 2.0 為主，REST 在
  net48 上最直接且無相依風險；structured output 免除自由文字解析。
- **Alternatives considered**: Google.Cloud.AIPlatform 套件 — 相依鏈重且
  非 Gemini API（AI Studio）路徑，否決；本機 OCR（Tesseract）— 手寫
  辨識品質不足，否決。

## R-08 爬蟲自動抓取

- **Decision**: `HttpClient` 取回
  `http://172.105.210.232/liveresultst/liveresultst.html?openid=755276`，
  以 HtmlAgilityPack 解析 DOM；依「日期、時間、場次編號」三鍵比對
  選定場次。詳細比分位於來源頁之彈跳視窗——其資料載入方式
  （同頁 DOM 或另一 HTTP 請求）於實作期以瀏覽器 DevTools 確認後
  定案，解析器以介面（`ILiveResultSource`）隔離以便屆時替換與測試。
  逾時 15 秒；查無資料、多筆相符、來源異常時回白話訊息並保留
  人工路徑（FR-010）。輔助來源 `matchesst.html?openid=755276` 作為
  備援解析目標。
- **Rationale**: 來源為 HTTP 明文、IP 直連、賽後可能下線
  （tech-notes 風險備註），故解析邏輯必須可替換、失敗必須可降級；
  HtmlAgilityPack 為 net48 上最通用的 HTML 解析器。
- **Alternatives considered**: Selenium/無頭瀏覽器取彈窗內容 — 部署與
  執行成本高，僅在彈窗資料無法以 HTTP 重現時才考慮（記為實作期
  風險決策點），暫否決。

## R-09 並行編輯防護（FR-015）

- **Decision**: `ScoreRecord` 資料表加 `rowversion` 欄位做樂觀並行控制；
  送出／儲存時帶版本，不符即回白話衝突訊息，由後送出者重新載入。
- **Rationale**: 衝突頻率低（同場次多人同時輸入屬例外），樂觀鎖
  實作簡單且不持有連線；MSSQL `rowversion` 原生支援。
- **Alternatives considered**: 悲觀鎖／簽出機制 — 需處理遺留鎖與逾時，
  複雜度不符風險，否決。

## R-10 草稿自動保存（FR-014）

- **Decision**: 以伺服器端草稿為主：輸入畫面每 15 秒及欄位失焦時
  將表單以 AJAX 寫入狀態為「草稿」的 ScoreRecord；瀏覽器端以
  localStorage 暫存最近一次表單內容作斷線備援，連線恢復或重新進入
  時比對時間戳提示還原。
- **Rationale**: 伺服器草稿讓換裝置／換人接手也能續作；localStorage
  補住短暫斷線（spec 假設：短暫斷線以草稿機制因應）。
- **Alternatives considered**: 純 localStorage — 換裝置即遺失、覆核者
  看不到輸入中狀態，否決。

## R-11 身分與角色（FR-016、FR-008）

- **Decision**: 進入系統時選角色並自由輸入姓名，存入加密 cookie
  （ASP.NET FormsAuthentication ticket 或 MachineKey 保護之自訂 cookie），
  供後續操作紀錄與四眼原則字串比對。無帳號系統。
- **Rationale**: spec Clarifications 已定案「自由輸入姓名」；cookie 讓
  重新整理與多頁流程不需重輸。同名風險由現場管理控管（spec 假設）。
- **Alternatives considered**: Session（InProc）— IIS 回收即遺失，cookie
  較穩，否決。

## R-12 測試框架與替身

- **Decision**: xUnit 2.x + Moq；測試專案目標框架 net48。
  核心規則（比分驗證、勝方判定、團體賽排點、狀態流轉）與
  解析器（OCR 回應映射、爬蟲 HTML 解析——以本機 HTML fixture 餵入）
  皆為必測之核心邏輯（憲章原則 II）；外部 HTTP 與資料庫以介面替身隔離。
- **Rationale**: xUnit 於 net48 透過 `xunit.runner.visualstudio` 完整支援
  VS Test Explorer 與 `vstest.console`；Moq 為 net48 相容的主流 mock 庫。
- **Alternatives considered**: MSTest v2 — 可行但社群慣例與斷言 API 較弱；
  NUnit — 同樣可行；三者擇一即可，xUnit 取其慣例簡潔。

## R-13 程式碼品質工具（憲章原則 I）

- **Decision**: 版本庫根目錄放 `.editorconfig`（命名、格式、C# 風格規則，
  含 severity 設定）；NuGet 引入 `Microsoft.CodeAnalysis.NetAnalyzers`
  與 `StyleCop.Analyzers`，警告視為建置錯誤（`TreatWarningsAsErrors`，
  逐步啟用）。格式化以 Visual Studio Code Cleanup 套用 `.editorconfig`。
- **Rationale**: classic csproj（MVC 5 必須）無法用 `dotnet format`，
  analyzers + editorconfig 是 net48 上可強制執行的等效機制。
- **Alternatives considered**: ReSharper CLI — 授權與安裝負擔，否決。

## R-14 機密管理

- **Decision**: `web.config` 之 `<connectionStrings>` 與 `<appSettings>`
  以 `configSource` 拆至 `Secrets.config`（已列入 `.gitignore`）；
  版本庫提供 `Secrets.config.example` 範本。涵蓋：MSSQL 連線字串
  （key：`UMallSport`）與 Gemini API 金鑰。
- **Rationale**: tech-notes 既定方向；.NET Framework 無 User Secrets，
  configSource 是原生且部署相容的做法。
- **Alternatives considered**: 環境變數 — IIS 應用程式集區設定較繁瑣、
  現場除錯不直觀，列為次選。

## R-15 部署與執行環境

- **Decision**: 開發以 IIS Express（Visual Studio F5）；正式部署
  Windows Server + IIS（.NET Framework 4.8、整合式管線）。內網存取
  MSSQL `192.168.66.99`／資料庫 `Badminton_T2_Dev`。
- **Rationale**: Framework 4.8 Web 應用的標準託管路徑；
  賽事現場為內網環境（tech-notes）。
- **Alternatives considered**: 無（平台由技術棧決定）。

## R-16 效能目標落地（憲章原則 IV、SC-006）

- **Decision**: 互動回饋 < 200ms（按鈕即時狀態以前端處理）、頁面切換
  < 1s、一般 AJAX p95 < 500ms；OCR（≤30s）與爬蟲（≤15s）為外部呼叫
  例外，UI 全程顯示處理中狀態與可取消；清單查詢一律帶條件與
  合理上限（本期資料量小，無分頁需求，記錄豁免理由於 plan）。
- **Rationale**: 量化門檻直接對應 spec SC-006 與憲章預設基準；
  外部呼叫偏離預設者依憲章記錄目標與理由。
- **Alternatives considered**: 無。
