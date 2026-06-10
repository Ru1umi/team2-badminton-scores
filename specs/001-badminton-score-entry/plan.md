# Implementation Plan: 羽球成績輸入系統

**Branch**: `001-badminton-score-entry` | **Date**: 2026-06-10 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-badminton-score-entry/spec.md`

## Summary

為羽球賽事建立成績輸入與覆核系統：成績輸入者於同一輸入畫面以
人工輸入、爬蟲自動抓取、OCR 影像判讀三種方式登錄單打／雙打／團體賽
成績（系統依比分自動判定勝方），送審前必附成績單圖檔佐證；
成績覆核者於待審核／已審核雙頁籤清單執行通過／駁回／拉回，
全程操作歷程可稽核。技術做法：.NET Framework 4.8 + C# 之
ASP.NET MVC 5 單體 Web 應用，Razor 輸出 HTML + Bootstrap 5 RWD，
Dapper 串接 MSSQL（`Badminton_T2_Dev`，schema `bse`），
OCR 走 Gemini API REST（`gemini-3.1-flash-lite`），爬蟲以
HttpClient + HtmlAgilityPack 解析賽事即時成績頁。

## Technical Context

**Language/Version**: C# 7.3 on .NET Framework 4.8（使用者指定）

**Primary Dependencies**: ASP.NET MVC 5（Razor server-rendered HTML）、
Bootstrap 5 + jQuery 3（前端 HTML／RWD）、Dapper（資料存取）、
HtmlAgilityPack（爬蟲解析）、Gemini API REST via HttpClient（OCR）、
Newtonsoft.Json（序列化）

**Storage**: Microsoft SQL Server — 內網 `192.168.66.99`，資料庫
`Badminton_T2_Dev`，專屬 schema `bse`；連線字串（key `UMallSport`）
存於未進版控之 `Secrets.config`（佐證圖檔以 `varbinary(max)` 入庫）

**Testing**: xUnit 2.x + Moq（net48；核心規則引擎、狀態流轉、
OCR 映射與爬蟲解析以 fixture 隔離測試）

**Target Platform**: Windows Server + IIS（開發：IIS Express）；
用戶端為桌機、手機（直式）、平板之現代瀏覽器（FR-020）

**Project Type**: Web application（單體 MVC，無獨立前端建置鏈）

**Performance Goals**: 操作回饋 < 0.2s、頁面切換 < 1s（SC-006）；
一般 AJAX p95 < 500ms（憲章預設）；OCR ≤ 30s、爬蟲 ≤ 15s
（外部呼叫，UI 顯示處理中狀態，偏離理由見 research.md R-16）

**Constraints**: .NET Framework 4.8 限定（不可用 ASP.NET Core 生態）；
行動裝置不水平捲動（SC-008）；爬蟲來源為 HTTP 明文、IP 直連、
賽後可能下線（解析器以介面隔離，可降級人工輸入）；
機密一律不入版控（web.config `configSource`）

**Scale/Scope**: 本期 3 場示範場次、賽事現場 ≤ 約 20 名併發操作人員、
約 6 個頁面 + 12 個 JSON 端點；單賽事成績量為數百筆等級

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | 原則 | 評估 | 結果 |
|---|------|------|------|
| I | 程式碼品質與可讀性 | `.editorconfig` + NetAnalyzers/StyleCop.Analyzers 列入專案初始化（R-13）；三專案分層（Web/Core/Tests）維持單一職責 | PASS |
| II | 測試標準 | 比分驗證、勝方判定、排點規則、狀態流轉為核心邏輯，xUnit 必測；DB/HTTP 以 Repository 與 `ILiveResultSource`/`IOcrClient` 介面替身隔離，測試具決定性 | PASS |
| III | UX 一致性 | 統一 JSON 回應格式與白話錯誤訊息約定（contracts）；Bootstrap 元件一致；載入／成功／失敗／空狀態皆有定義（處理中狀態含 OCR/爬蟲） | PASS |
| IV | 效能要求 | SC-006 已量化；外部呼叫（OCR/爬蟲）偏離預設之目標與理由記錄於 R-16；Dapper 明確 SQL 避免 N+1；本期資料量小、清單不分頁之豁免記錄於 R-16 | PASS |
| — | 附加限制 | 技術選型記錄於本檔 Technical Context；新相依（Dapper、HtmlAgilityPack、StyleCop.Analyzers…）用途記錄於 research.md；使用者資料（姓名、操作紀錄）保存邊界載明於 spec 假設與 data-model | PASS |

**Pre-Phase 0 gate**: PASS（無違規，無需 Complexity Tracking）

**Post-Phase 1 re-check**: PASS — 設計後新增之實體（SessionSide、
RubberPlayer、EvidenceImage）皆由 FR 直接驅動，未引入規格外複雜度；
單一 MVC 專案 + Core 類庫 + 測試專案共 3 個專案，未超出 Web 應用
常規結構。

## Project Structure

### Documentation (this feature)

```text
specs/001-badminton-score-entry/
├── plan.md              # 本檔（/speckit-plan 輸出）
├── research.md          # Phase 0 輸出（16 項技術決策 R-01…R-16）
├── data-model.md        # Phase 1 輸出（bse schema、狀態機、驗證規則）
├── quickstart.md        # Phase 1 輸出（環境、啟動、V1–V7 驗證情境）
├── contracts/
│   └── http-api.md      # Phase 1 輸出（頁面 + JSON 端點契約）
├── tech-notes.md        # specify 階段實作層資訊（已併入本計畫）
└── tasks.md             # Phase 2 輸出（/speckit-tasks 產生，非本指令）
```

### Source Code (repository root)

```text
src/
├── BadmintonScore.Core/             # 類庫（net48）：商業邏輯與整合
│   ├── Domain/                      # 實體、狀態與列舉（對應 data-model）
│   ├── Rules/                       # 比分驗證、勝方判定、團體排點（核心邏輯）
│   ├── Services/                    # 成績流程、覆核流程、稽核寫入
│   ├── Data/                        # Dapper repositories（介面 + 實作）
│   └── External/                    # IOcrClient(Gemini REST)、ILiveResultSource(爬蟲)
└── BadmintonScore.Web/              # ASP.NET MVC 5（net48）
    ├── App_Start/                   # 路由、DI（簡易組合根）、過濾器
    ├── Controllers/                 # 頁面 + JSON 端點（contracts/http-api.md）
    ├── Views/                       # Razor：角色選擇/場次/輸入/覆核
    ├── Content/                     # Bootstrap 5、site.css
    ├── Scripts/                     # jQuery、site.js（自動儲存、即時驗證、預填）
    ├── Web.config                   # configSource → Secrets.config（不入版控）
    └── Secrets.config.example      # 機密範本

tests/
└── BadmintonScore.Core.Tests/       # xUnit + Moq（net48）
    ├── Rules/                       # 比分/排點/狀態流轉測試
    ├── Services/                    # 流程服務（替身隔離）
    └── External/                    # 爬蟲解析（HTML fixture）、OCR 映射

database/
├── schema/                          # 001_…  bse schema 與資料表（含 rowversion）
└── seed/                            # 001_demo_sessions.sql（三場示範場次）
```

**Structure Decision**: Web 應用採單一 MVC 專案（前端為 Razor 輸出之
HTML + 靜態資源，無獨立 frontend/ 目錄與建置鏈）；商業邏輯全數下沉
`BadmintonScore.Core` 類庫，使核心規則可脫離 System.Web 測試
（憲章原則 II）；資料庫物件以 `database/` 腳本管理（R-05）。

## Complexity Tracking

> 無憲章違規——本表無項目。
