# Tasks: 羽球成績輸入系統

**Input**: Design documents from `/specs/001-badminton-score-entry/`

**Prerequisites**: plan.md、spec.md、research.md、data-model.md、contracts/http-api.md、quickstart.md

**Tests**: 依憲章原則 II，核心邏輯（比分規則、勝方判定、排點、狀態流轉、OCR 映射、爬蟲解析）之單元測試為強制任務，與實作成對列出；契約／整合測試未被要求，不列入。

**Organization**: 任務依使用者故事分組，每個故事可獨立實作與驗證。

## Format: `[ID] [P?] [Story] Description`

- **[P]**: 可平行執行（不同檔案、無未完成依賴）
- **[Story]**: 所屬使用者故事（US1–US5）
- 路徑依 plan.md 專案結構（`src/BadmintonScore.Core`、`src/BadmintonScore.Web`、`tests/BadmintonScore.Core.Tests`、`database/`）

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: 方案與專案初始化、品質工具、機密管理骨架

- [ ] T001 建立方案與三個 net48 專案：`BadmintonScore.sln`、`src/BadmintonScore.Core/BadmintonScore.Core.csproj`（類庫）、`src/BadmintonScore.Web/BadmintonScore.Web.csproj`（ASP.NET MVC 5）、`tests/BadmintonScore.Core.Tests/BadmintonScore.Core.Tests.csproj`（xUnit），設定專案參考（Web→Core、Tests→Core）
- [ ] T002 安裝 NuGet 相依：Core（Dapper、HtmlAgilityPack、Newtonsoft.Json）、Web（Microsoft.AspNet.Mvc 5.x、jQuery 3、bootstrap 5）、Tests（xunit、xunit.runner.visualstudio、Moq）；各 csproj 啟用 analyzers
- [ ] T003 [P] 建立 `.editorconfig`（C# 命名/格式/風格規則與 severity）並於三個專案加入 `Microsoft.CodeAnalysis.NetAnalyzers` 與 `StyleCop.Analyzers`（research R-13）
- [ ] T004 [P] 機密管理骨架：`src/BadmintonScore.Web/Web.config` 之 connectionStrings/appSettings 改用 `configSource` 指向 `Secrets.config`；建立 `src/BadmintonScore.Web/Secrets.config.example`（含 `UMallSport` 連線字串與 `Gemini:ApiKey` 佔位）；建立 `.gitignore`（排除 Secrets.config、bin、obj、.omc）

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: 所有故事共用的資料庫、領域模型、MVC 骨架、身分與稽核基礎

**⚠️ CRITICAL**: 本階段完成前不得開始任何使用者故事

- [ ] T005 建立 `database/schema/001_create_schema.sql`：`bse` schema 與 9 張資料表（MatchSession、SessionSide、Player、SideMember、ScoreRecord 含 rowversion 與 SessionId 唯一索引、Game、Rubber、RubberPlayer、EvidenceImage、AuditEntry）及外鍵／唯一約束（data-model.md）
- [ ] T006 建立 `database/seed/001_demo_sessions.sql`：三場示範場次（團體／單打／雙打決賽）與兩方選手名單（FR-001、data-model 種子資料節）
- [ ] T007 [P] 建立領域模型與列舉於 `src/BadmintonScore.Core/Domain/`（MatchSession.cs、SessionSide.cs、Player.cs、ScoreRecord.cs、Game.cs、Rubber.cs、EvidenceImage.cs、AuditEntry.cs、Enums.cs：Discipline/RecordStatus/ScoreSource/AuditAction）
- [ ] T008 [P] 建立 `src/BadmintonScore.Core/Data/DbConnectionFactory.cs`：讀取 `UMallSport` 連線字串、提供 `IDbConnection` 工廠介面
- [ ] T009 建立 Repository 介面與 Dapper 實作於 `src/BadmintonScore.Core/Data/`：`ISessionRepository`（清單含成績狀態、單筆含兩方名單）、`IScoreRecordRepository`（依場次載入完整紀錄、儲存含 rowversion 檢查）、`IAuditRepository`（append-only 寫入、依場次查詢）
- [ ] T010 建立 MVC 骨架於 `src/BadmintonScore.Web/`：`App_Start/RouteConfig.cs`、組合根（簡易 DI）`App_Start/CompositionRoot.cs`、`Views/Shared/_Layout.cshtml`（Bootstrap 5、行動優先 viewport）、`Controllers/BaseController.cs`（統一 `{ok,data}/{ok,message,field}` JSON 輔助、AntiForgery、409 衝突轉譯——contracts 共用約定）
- [ ] T011 身分機制（FR-016 基礎）：`Controllers/HomeController.cs` 角色選擇頁 `/` 與 `Views/Home/Index.cshtml`（姓名輸入＋兩角色大按鈕）、`POST /api/identity`、加密 cookie 寫讀（`Infrastructure/IdentityCookie.cs`）、未帶身分之頁面導回 `/`／JSON 回 401 的過濾器
- [ ] T012 建立 `src/BadmintonScore.Core/Services/AuditService.cs`：寫入操作紀錄（含狀態轉移與 JSON 快照）與依場次查詢（FR-012；自動儲存節流規則見 data-model）
- [ ] T013 全域錯誤處理與日誌於 `src/BadmintonScore.Web/`：例外過濾器（JSON 端點回白話 `{ok:false}`、頁面導向友善錯誤頁）、伺服器端檔案日誌（Trace listener 設定於 Web.config）

**Checkpoint**: 基礎就緒——可啟動站台、選角色、連線資料庫

---

## Phase 3: User Story 1 - 選擇角色與場次並人工輸入單打／雙打成績 (Priority: P1) 🎯 MVP

**Goal**: 輸入者選場次後於單一畫面人工輸入單打／雙打比分，系統自動判定勝方、即時驗證、強制佐證圖檔後送審；含草稿自動保存、並行衝突防護與操作歷程檢視

**Independent Test**: quickstart.md V1（示範單打／雙打場次完整輸入送審）＋ V6（草稿還原、雙視窗衝突）

### Implementation for User Story 1

- [ ] T014 [P] [US1] 實作單局比分規則 `src/BadmintonScore.Core/Rules/GameScoreRules.cs`：21 分制、20 平領先 2、30 封頂、不合法原因以白話訊息列舉（FR-005）
- [ ] T015 [P] [US1] 單元測試 `tests/BadmintonScore.Core.Tests/Rules/GameScoreRulesTests.cs`：合法（21:18、22:20、30:29）、不合法（21:20、35:33、同分、負分）各情境
- [ ] T016 [US1] 實作三戰兩勝判定 `src/BadmintonScore.Core/Rules/MatchOutcomeCalculator.cs`：逐局勝方、兩局先勝即整場結束、第三局僅限 1:1（FR-003）
- [ ] T017 [US1] 單元測試 `tests/BadmintonScore.Core.Tests/Rules/MatchOutcomeCalculatorTests.cs`：2:0、2:1、未完成、非法第三局
- [ ] T018 [US1] 實作 `src/BadmintonScore.Core/Services/ScoreEntryService.cs`：載入場次與既有紀錄（含已駁回帶原內容）、草稿儲存（建立或更新、rowversion 檢查、audit 節流）、送審（規則全驗證＋至少一張佐證 FR-018＋狀態 草稿/已駁回→待覆核＋快照 audit）
- [ ] T019 [US1] 單元測試 `tests/BadmintonScore.Core.Tests/Services/ScoreEntryServiceTests.cs`（Moq 替身）：無佐證送審被拒、駁回重開帶原內容、rowversion 不符擲衝突、狀態轉移正確
- [ ] T020 [US1] 場次選擇頁 `src/BadmintonScore.Web/Controllers/SessionsController.cs` 與 `Views/Sessions/Index.cshtml`：依日期/場地/場次列出三場示範場次、顯示各場成績狀態（FR-001）
- [ ] T021 [US1] 成績輸入頁 `src/BadmintonScore.Web/Controllers/ScoreController.cs`（GET `/score/{sessionId}`）與 `Views/Score/Entry.cshtml`：自動帶入場次與名單、預設人工模式、預留「爬蟲」「OCR」兩鈕（先以停用狀態佔位）、佐證圖上傳區、操作歷程區塊、送出確認步驟（FR-002、FR-013、FR-017）
- [ ] T022 [US1] JSON 端點於 `src/BadmintonScore.Web/Controllers/ScoreController.cs`：`GET /api/sessions/{id}`、`POST /api/score/draft`（即時逐局驗證結果回傳）、`POST /api/score/submit`（contracts/http-api.md 格式）
- [ ] T023 [US1] 佐證圖檔端點於 `src/BadmintonScore.Web/Controllers/EvidenceController.cs`：multipart 上傳（JPEG/PNG、≤10MB、413 處理）、刪除（限草稿/已駁回）、`GET /evidence/{imageId}` 圖檔輸出（FR-018、R-06）
- [ ] T024 [US1] 前端輸入模組 `src/BadmintonScore.Web/Scripts/score-entry.js`：逐欄即時驗證提示（<0.2s 回饋）、每 15 秒＋失焦自動儲存、localStorage 斷線備援與還原提示、409 衝突白話處理、送出前確認（FR-005、FR-014、FR-015、SC-006）
- [ ] T025 [US1] 操作歷程：`GET /api/sessions/{id}/audit` 端點與輸入頁歷程區塊渲染（誰/何時/動作/狀態變化，FR-012）

**Checkpoint**: MVP 完成——單打/雙打可全流程輸入送審（quickstart V1、V6 通過）

---

## Phase 4: User Story 2 - 團體賽成績輸入 (Priority: P2)

**Goal**: 團體賽 5 點制（3 單 2 雙）排點、逐點輸入、自動統計勝出點數與「未賽」標記

**Independent Test**: quickstart.md V2（示範團體場次排點→逐點輸入→先 3 點獲勝→送審）

### Implementation for User Story 2

- [ ] T026 [P] [US2] 實作排點規則 `src/BadmintonScore.Core/Rules/LineupRules.cs`：點次人數（單1雙2）、同點次不重複、每人≤2 點且≤1單≤1雙、選手須屬該方名單（FR-004＋spec 假設）
- [ ] T027 [P] [US2] 單元測試 `tests/BadmintonScore.Core.Tests/Rules/LineupRulesTests.cs`：各違規情境與白話訊息
- [ ] T028 [US2] 實作團體賽勝負統計 `src/BadmintonScore.Core/Rules/TeamOutcomeCalculator.cs`：逐點勝方（沿用 MatchOutcomeCalculator）、先 3 點獲勝、剩餘點次可標記未賽且不強制輸入（FR-004）
- [ ] T029 [US2] 單元測試 `tests/BadmintonScore.Core.Tests/Rules/TeamOutcomeCalculatorTests.cs`：3:0、3:2、未賽標記、未達 3 點不得送審
- [ ] T030 [US2] 擴充 `src/BadmintonScore.Core/Services/ScoreEntryService.cs` 與 `Data/ScoreRecordRepository.cs`：Rubber/RubberPlayer 載入、草稿儲存與送審驗證（排點＋逐點比分＋整場判定）
- [ ] T031 [US2] 擴充 `src/BadmintonScore.Web/Controllers/ScoreController.cs` 之 draft/submit payload 支援 `rubbers`（contracts/http-api.md 團體格式）
- [ ] T032 [US2] 團體賽輸入 UI：`Views/Score/Entry.cshtml` 團體區塊與 `src/BadmintonScore.Web/Scripts/team-entry.js`（點次出賽選手下拉、即時排點違規提示、逐點比分、未賽標記、勝出點數即時統計）

**Checkpoint**: US1＋US2 皆可獨立通過（quickstart V1、V2）

---

## Phase 5: User Story 3 - 成績覆核與發佈 (Priority: P2)

**Goal**: 覆核者於待審核／已審核雙頁籤清單對照佐證執行通過／駁回／拉回；四眼原則；駁回回流輸入者重輸

**Independent Test**: quickstart.md V3（通過、駁回、拉回、四眼拒絕、駁回回流重輸全路徑）

### Implementation for User Story 3

- [ ] T033 [US3] 實作 `src/BadmintonScore.Core/Services/ReviewService.cs`：清單查詢（待審核/已審核）、通過（四眼檢查 FR-008、待覆核→已發佈）、駁回（→已駁回）、拉回（已發佈/已駁回→待覆核 FR-019）、全部寫入 audit 與 rowversion 檢查（FR-006 狀態機）
- [ ] T034 [US3] 單元測試 `tests/BadmintonScore.Core.Tests/Services/ReviewServiceTests.cs`：四眼拒絕（同名）、非法狀態轉移拒絕、拉回後可重審、已發佈不可被輸入操作修改
- [ ] T035 [US3] 覆核頁面 `src/BadmintonScore.Web/Controllers/ReviewController.cs` 與 `Views/Review/Index.cshtml`（雙頁籤清單：場次、輸入者、來源、時間）＋ `Views/Review/Detail.cshtml`（成績、佐證圖、爬蟲來源連結——有佐證即顯示，FR-007）
- [ ] T036 [US3] JSON 端點於 `src/BadmintonScore.Web/Controllers/ReviewController.cs`：`GET /api/review/list?tab=`、`POST /api/review/{id}/approve|reject|recall`（contracts 格式、白話四眼訊息）
- [ ] T037 [US3] 駁回回流：`Controllers/SessionsController.cs` 清單加入「已駁回待重輸」醒目提示；確認輸入頁重開帶原內容並於修改後可再次送審（FR-006，串 T018 既有邏輯）
- [ ] T038 [US3] 前端覆核模組 `src/BadmintonScore.Web/Scripts/review.js`：頁籤切換、通過/駁回/拉回之確認步驟（FR-013）、佐證圖放大檢視

**Checkpoint**: 輸入→覆核→發佈/駁回→重輸閉環完成（quickstart V3）

---

## Phase 6: User Story 4 - OCR 影像判讀輸入 (Priority: P3)

**Goal**: 上傳手寫成績單影像，Gemini 辨識預填表單、低信心標示、辨識影像自動帶入佐證；失敗白話引導

**Independent Test**: quickstart.md V4（清晰影像預填＋低信心標示；模糊影像白話失敗）

### Implementation for User Story 4

- [ ] T039 [P] [US4] 實作 `src/BadmintonScore.Core/External/IOcrClient.cs` 與 `GeminiOcrClient.cs`：HttpClient 呼叫 Gemini API（模型 `gemini-3.1-flash-lite`、structured output JSON schema、30 秒逾時、金鑰自 appSettings；R-07）
- [ ] T040 [US4] 實作 `src/BadmintonScore.Core/Services/OcrPrefillService.cs`：辨識結果→表單預填映射、逐欄信心值與門檻（0.8）、辨識選手與場次名單衝突偵測（Edge case）、失敗原因白話化（FR-009）
- [ ] T041 [US4] 單元測試 `tests/BadmintonScore.Core.Tests/Services/OcrPrefillServiceTests.cs`：以 fixture JSON 驗證映射、低信心標記、名單衝突、失敗訊息（IOcrClient 以 Moq 替身）
- [ ] T042 [US4] 端點 `POST /api/ocr/recognize` 於 `src/BadmintonScore.Web/Controllers/OcrController.cs`：multipart 接收、辨識影像自動寫入佐證（FR-018）、回傳 prefill＋confidence＋conflicts（contracts 格式）
- [ ] T043 [US4] 前端 OCR 模組：啟用輸入頁「OCR 影像判讀」鈕、`src/BadmintonScore.Web/Scripts/ocr.js`（上傳、處理中狀態、預填寫入表單、低信心欄位醒目標示、未確認不得直接送出之引導）

**Checkpoint**: OCR 預填可用且不影響人工路徑（quickstart V4）

---

## Phase 7: User Story 5 - 爬蟲自動抓取輸入 (Priority: P3)

**Goal**: 依日期/時間/場次編號比對賽事即時成績頁（含彈跳視窗詳細比分）預填表單；查無/異常白話降級

**Independent Test**: quickstart.md V5（命中預填＋來源連結；查無資料白話訊息）

### Implementation for User Story 5

- [ ] T044 [P] [US5] 實作 `src/BadmintonScore.Core/External/ILiveResultSource.cs` 與 `LiveResultCrawler.cs`：HttpClient＋HtmlAgilityPack 解析 `liveresultst.html`、三鍵比對（日期/時間/場次編號）、彈跳視窗資料取得（實作期以 DevTools 確認載入方式）、15 秒逾時、備援 `matchesst.html`、查無/多筆/改版例外分類（R-08、FR-010）
- [ ] T045 [US5] 單元測試 `tests/BadmintonScore.Core.Tests/External/LiveResultCrawlerTests.cs`：以本機 HTML fixture 驗證命中、查無、多筆、版面改變四情境（不依賴真實網路，憲章原則 II）
- [ ] T046 [US5] 端點 `GET /api/crawler/prefill?sessionId=` 於 `src/BadmintonScore.Web/Controllers/CrawlerController.cs`：回傳 prefill＋sourceUrl＋matchedKey；異常回白話訊息並保留人工路徑（contracts 格式）
- [ ] T047 [US5] 前端爬蟲模組：啟用輸入頁「爬蟲自動抓取」鈕、`src/BadmintonScore.Web/Scripts/crawler.js`（處理中狀態、預填寫入、來源連結顯示、逐欄確認後才可送出）；送審後覆核畫面顯示爬蟲來源連結（串 T035）

**Checkpoint**: 全部五個故事獨立可驗（quickstart V1–V6）

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: 跨故事的 RWD、效能、品質與最終驗證

- [ ] T048 [P] RWD 全面調整 `src/BadmintonScore.Web/Content/site.css` 與各 Views：手機直式與平板完成輸入/覆核全流程、無水平捲動、大按鈕（FR-020、SC-008，quickstart V7）
- [ ] T049 [P] 效能查核與修正：操作回饋 <0.2s、頁面切換 <1s、AJAX p95 <500ms 抽查；檢查 repositories 無 N+1（憲章原則 IV、R-16）
- [ ] T050 Analyzer 警告清零與 code cleanup（`.editorconfig` 全套用、命名與單一職責複查，憲章原則 I）
- [ ] T051 依 `specs/001-badminton-score-entry/quickstart.md` 執行 V1–V7 全情境驗證並記錄結果於 quickstart 末節
- [ ] T052 [P] 安全強化複查：AntiForgery 全寫入端點覆蓋、上傳型別/大小白名單、`Secrets.config` 未入版控、錯誤頁不洩漏內部資訊

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: 無依賴，立即可開始
- **Foundational (Phase 2)**: 依賴 Phase 1；**阻擋所有使用者故事**
- **User Stories (Phase 3–7)**: 皆依賴 Phase 2 完成後即可開始
  - US1 (P1) 無故事間依賴
  - US2 (P2) 沿用 US1 的輸入頁與服務（T021、T018）→ 建議於 US1 後進行
  - US3 (P2) 需要 US1 產生的待覆核資料才能端到端驗證，但其服務/頁面可在 Phase 2 後平行開發
  - US4、US5 (P3) 依賴 US1 的輸入表單預填介面（T021、T024）
- **Polish (Phase 8)**: 依賴所有欲交付故事完成

### User Story Dependencies

- **US1 (P1)**: Foundational 後即可，無其他依賴 🎯 MVP
- **US2 (P2)**: 技術上依賴 US1 的 Entry 頁與 ScoreEntryService 擴充點
- **US3 (P2)**: 可與 US2 平行（不同檔案）；端到端驗證需 US1 完成
- **US4 (P3)**: 依賴 US1 表單；與 US5 完全平行（不同檔案）
- **US5 (P3)**: 依賴 US1 表單；與 US4 完全平行（不同檔案）

### Within Each User Story

- 規則（Rules）→ 服務（Services）→ 端點（Controllers）→ 前端（Scripts/Views）
- 單元測試與其標的成對完成，合併門檻為全數通過（憲章原則 II）

### Parallel Opportunities

- Phase 1：T003、T004 平行
- Phase 2：T007、T008 平行（T005→T006 串行；T009 待 T007/T008）
- US1：T014/T015 與 T016/T017 兩組規則可平行起步
- Phase 2 完成後：US3 服務層（T033/T034）可與 US2 平行開發
- US4 與 US5 互不相依，可由不同人力同時進行（T039 與 T044 平行）
- Phase 8：T048、T049、T052 平行

---

## Parallel Example: User Story 1

```bash
# 平行起步兩組核心規則與測試：
Task: "實作單局比分規則 src/BadmintonScore.Core/Rules/GameScoreRules.cs"
Task: "單元測試 tests/BadmintonScore.Core.Tests/Rules/GameScoreRulesTests.cs"

# 規則完成後串行進入服務與端點：
Task: "ScoreEntryService → ScoreController JSON 端點 → score-entry.js"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1 Setup → Phase 2 Foundational（關鍵阻擋）
2. Phase 3 US1 完成 → **停下驗證**：quickstart V1＋V6 獨立通過
3. 可現場示範：單打/雙打人工輸入送審全流程

### Incremental Delivery

1. Setup＋Foundational → 基礎就緒
2. US1 → 驗證 → MVP 示範
3. US2（團體賽）→ 驗證 V2
4. US3（覆核）→ 驗證 V3 → 輸入-覆核閉環可上線試用
5. US4（OCR）、US5（爬蟲）→ 驗證 V4/V5 → 效率強化完備
6. Phase 8 Polish → V1–V7 全驗收

### Parallel Team Strategy

1. 全員完成 Setup＋Foundational
2. 之後：開發者 A 走 US1→US2；開發者 B 於 US1 服務介面定案後開發 US3 服務與頁面；開發者 C 於 US1 表單完成後接 US4/US5
3. 各故事獨立驗證後整合

---

## Notes

- [P] 任務＝不同檔案且無未完成依賴
- 每完成一個任務或邏輯群組即提交（本專案尚未 git init，首次提交前先初始化）
- 任一 Checkpoint 皆可停下獨立驗證該故事
- OCR 金鑰與資料庫連線字串一律放 `Secrets.config`（不入版控）
