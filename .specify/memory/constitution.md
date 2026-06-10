<!--
Sync Impact Report
==================
- Version change: (template, 未制定) → 1.0.0
- Modified principles: 初次制定，無原則重新命名
- Added sections:
  - Core Principles（四項原則：I. 程式碼品質與可讀性、II. 測試標準、
    III. 使用者體驗一致性、IV. 效能要求）
  - 附加限制（Additional Constraints）
  - 開發工作流程與品質門檻（Development Workflow & Quality Gates）
  - Governance
- Removed sections: 範本之第五項原則佔位符（本專案依使用者輸入僅定義四項原則）
- Templates requiring updates:
  - ✅ .specify/templates/plan-template.md — Constitution Check 為動態門檻，
    由本憲章決定，無需修改
  - ✅ .specify/templates/spec-template.md — Success Criteria 已強制要求可量測
    指標，與原則 IV 一致，無需修改
  - ✅ .specify/templates/tasks-template.md — 已更新：核心邏輯單元測試由
    OPTIONAL 改為依原則 II 強制
- Follow-up TODOs: 無
-->

# my-project Constitution

## Core Principles

### I. 程式碼品質與可讀性（Code Quality & Readability）

- 所有程式碼 MUST 通過專案設定的 linter 與 formatter 檢查後方可合併；
  工具設定一經建立即為強制標準，不得以個人偏好繞過。
- 命名 MUST 清楚表達意圖；禁止使用無意義或僅自己理解的縮寫。
- 函式與模組 MUST 維持單一職責；超出單一職責或可讀性明顯下降的程式碼，
  MUST 拆分或在 code review 中提出書面理由。
- 註解 MUST 解釋「為什麼」（限制、取捨、非顯而易見的前提），
  而非重述程式碼「做什麼」。
- 任何合併至主幹的變更 MUST 經過至少一次 code review。

理由：程式碼被閱讀的次數遠多於被撰寫的次數。一致且可讀的程式碼
降低維護成本與缺陷率，並使後續貢獻者能安全地修改。

### II. 測試標準（Testing Standards）

- 核心商業邏輯 MUST 具備單元測試；「核心邏輯」指資料轉換、商業規則、
  演算法與狀態管理等，不含純樣板或框架黏合程式碼。
- 修復缺陷時 MUST 先新增能重現該缺陷的測試，再進行修復（紅 → 綠）。
- 測試 MUST 具決定性（deterministic）：不得依賴真實網路、真實時鐘
  或執行順序；外部相依 MUST 以替身（mock/stub/fake)隔離。
- 合併前 MUST 全數測試通過；不得以略過（skip）失敗測試的方式合併。
- 契約測試與整合測試為 SHOULD：在跨服務介面或共用 schema 變更時補上。

理由：核心邏輯是缺陷代價最高的區域。強制單元測試使重構有安全網，
並讓回歸在合併前即被攔截，而非在使用者端爆發。

### III. 使用者體驗一致性（UX Consistency）

- 介面元件、版面結構、文案語氣與互動模式 MUST 在整個產品中保持一致；
  相同操作在不同畫面 MUST 產生一致的行為與回饋。
- 所有使用者可見的狀態（載入、成功、失敗、空狀態）MUST 有明確且
  一致的視覺回饋。
- 錯誤訊息 MUST 一致、具體且可行動（actionable）：說明發生什麼事
  以及使用者下一步可以做什麼。
- 新增 UI 模式前 MUST 先確認既有模式無法滿足需求；
  偏離既有慣例者 MUST 在規格中記錄理由。

理由：一致性降低使用者的學習成本與操作錯誤。每一個「特例」介面
都是使用者必須重新學習的負擔，也是日後維護的分歧點。

### IV. 效能要求（Performance Requirements）

- 每個功能規格 MUST 在 Success Criteria 中定義可量測的效能目標
  （例如回應時間、處理量、資源上限）；未知時 MUST 標註
  NEEDS CLARIFICATION，不得默默略過。
- 預設基準（規格未另行定義時）：使用者互動操作 MUST 在 200ms 內
  給予回饋；一般請求 p95 MUST < 500ms。偏離預設者 MUST 在規格中
  記錄目標與理由。
- 程式碼 MUST 避免明顯的效能反模式：N+1 查詢、不必要的同步阻塞、
  未分頁的大量資料載入、迴圈內重複 I/O。
- code review 中發現的效能退化 MUST 修正，或以書面記錄豁免理由
  與後續追蹤項目。

理由：效能是 UX 的一部分，事後補救的成本遠高於設計時納入。
明確的數值門檻讓「夠快」成為可驗證的事實而非主觀感受。

## 附加限制（Additional Constraints）

- 技術選型（語言、框架、儲存、測試工具）MUST 於各功能的 plan.md
  Technical Context 中明確記錄；尚未決定者標註 NEEDS CLARIFICATION。
- 引入新的外部相依套件 SHOULD 優先評估既有相依能否滿足需求；
  新增相依 MUST 在 plan.md 中說明用途。
- 涉及使用者資料的功能 MUST 在規格中載明資料保存與存取邊界。

## 開發工作流程與品質門檻（Development Workflow & Quality Gates）

- 規格先行：功能開發 MUST 依 spec → plan → tasks → implement 流程進行；
  plan.md 的 Constitution Check MUST 以本憲章為門檻，於 Phase 0 研究前
  與 Phase 1 設計後各檢查一次。
- 合併門檻（全部 MUST 滿足）：
  1. lint / format 檢查通過（原則 I）；
  2. 核心邏輯單元測試存在且全數通過（原則 II）；
  3. 使用者可見變更符合既有 UX 慣例或已記錄偏離理由（原則 III）；
  4. 符合效能目標或已記錄豁免（原則 IV）。
- 違反憲章的複雜度 MUST 記錄於 plan.md 的 Complexity Tracking 表，
  含必要性與被否決的較簡單替代方案。

## Governance

- 本憲章優先於其他開發慣例與個人偏好；衝突時以本憲章為準。
- 修訂程序：任何修訂 MUST 以書面提出，包含變更內容、理由與影響範圍；
  修訂通過後 MUST 同步更新版本號、Last Amended 日期，並檢查
  `.specify/templates/` 下相依範本的一致性。
- 版本政策（語意化版本）：
  - MAJOR：移除或重新定義原則等不向後相容的治理變更；
  - MINOR：新增原則／章節，或對既有指引的實質擴充；
  - PATCH：措辭澄清、錯字修正等非語意性調整。
- 合規審查：所有 PR 與 code review MUST 驗證是否符合各原則；
  review 者發現違反時 MUST 要求修正或記錄豁免，不得默許通過。

**Version**: 1.0.0 | **Ratified**: 2026-06-10 | **Last Amended**: 2026-06-10
