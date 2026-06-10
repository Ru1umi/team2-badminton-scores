# Data Model: 羽球成績輸入系統

**Date**: 2026-06-10
**Source**: spec.md Key Entities + Functional Requirements；技術決策見 research.md

所有資料表置於 `Badminton_T2_Dev` 資料庫之 `bse` schema（R-05）。
主鍵一律 `int identity`（標示 PK）；時間欄位 `datetime2`；
名稱類欄位 `nvarchar`。

## 實體總覽與關聯

```text
MatchSession 1 ──── 2 SessionSide ──── * SideMember ──── 1 Player
     │
     1
     │
     * (實務上 0..1 有效) ScoreRecord ──── * Game
     │                        │ ├──────── * Rubber ──── * RubberPlayer
     │                        │ └──────── * EvidenceImage
     └──────────────────────  * AuditEntry（亦可關聯 ScoreRecord）
```

## bse.MatchSession — 場次

既有賽程資料；本期由種子腳本建立三場示範場次（FR-001），
系統不提供建立／編輯功能。

| 欄位 | 型別 | 說明 |
|------|------|------|
| Id | int PK | |
| Division | nvarchar(50) | 組別／項目，如「公開男生組」 |
| Discipline | tinyint | 賽制：1 單打、2 雙打、3 團體（FR-003/004） |
| Round | nvarchar(20) | 賽別，如「決賽」 |
| MatchNo | int | 場次編號（爬蟲比對鍵之一，FR-010） |
| ScheduledAt | datetime2 | 日期時間（爬蟲比對鍵之一） |
| Venue | nvarchar(50) | 場地 |

## bse.SessionSide — 場次參賽方

每場次固定兩方（Side 1／Side 2）；單打＝1 名成員、雙打＝2 名、
團體＝隊伍全名單。以單一結構涵蓋三種賽制（Key Entities：選手／配對／隊伍）。

| 欄位 | 型別 | 說明 |
|------|------|------|
| Id | int PK | |
| SessionId | int FK→MatchSession | |
| Side | tinyint | 1 或 2；(SessionId, Side) 唯一 |
| DisplayName | nvarchar(100) | 顯示名稱（選手名／配對名／隊名） |

## bse.Player — 選手

| 欄位 | 型別 | 說明 |
|------|------|------|
| Id | int PK | |
| Name | nvarchar(50) | |

## bse.SideMember — 參賽方成員

| 欄位 | 型別 | 說明 |
|------|------|------|
| Id | int PK | |
| SideId | int FK→SessionSide | |
| PlayerId | int FK→Player | (SideId, PlayerId) 唯一 |

## bse.ScoreRecord — 成績紀錄

一場次的完整成績與狀態。同一場次同時只允許一筆「有效」紀錄
（Edge case：再次進入即編輯既有紀錄，不建立重複資料）。

| 欄位 | 型別 | 說明 |
|------|------|------|
| Id | int PK | |
| SessionId | int FK→MatchSession | 加唯一索引（本期一場次一紀錄） |
| Status | tinyint | 1 草稿、2 待覆核、3 已發佈、4 已駁回（FR-006） |
| Source | tinyint | 1 人工、2 OCR、3 爬蟲（FR-011；以最後帶入方式為準） |
| WinnerSide | tinyint NULL | 1／2；由系統依比分判定（FR-003/004），送審時必填 |
| EnteredBy | nvarchar(50) | 輸入者姓名（FR-016） |
| ReviewedBy | nvarchar(50) NULL | 覆核者姓名 |
| CrawlerSourceUrl | nvarchar(500) NULL | 爬蟲來源連結（FR-007 佐證） |
| SubmittedAt | datetime2 NULL | 最近一次送審時間 |
| ReviewedAt | datetime2 NULL | 最近一次覆核時間 |
| CreatedAt / UpdatedAt | datetime2 | |
| RowVersion | rowversion | 樂觀並行控制（FR-015，R-09） |

### 狀態機（FR-006、FR-019）

```text
草稿(1) ──送出(需佐證圖檔, FR-018)──▶ 待覆核(2)
待覆核(2) ──通過(覆核者≠輸入者, FR-008)──▶ 已發佈(3)
待覆核(2) ──駁回──▶ 已駁回(4)
已駁回(4) ──輸入者重新開啟(帶原內容)──▶ 草稿(1) ──…──▶ 待覆核(2)
已發佈(3)／已駁回(4) ──覆核者拉回──▶ 待覆核(2)
```

- 已發佈紀錄不可被一般輸入操作修改（FR-006）；唯一出口為覆核者拉回。
- 每次轉移皆寫入 AuditEntry，並附當下成績快照（歷次送審內容查證需求）。

## bse.Game — 局

單打／雙打成績掛 ScoreRecord（RubberId 為 NULL）；
團體賽各局掛點次（RubberId 非 NULL）。

| 欄位 | 型別 | 說明 |
|------|------|------|
| Id | int PK | |
| ScoreRecordId | int FK→ScoreRecord | |
| RubberId | int FK→Rubber NULL | 團體賽時指向所屬點次 |
| GameNo | tinyint | 1–3（三戰兩勝） |
| Side1Points / Side2Points | tinyint | 0–30 |
| WinnerSide | tinyint | 由系統依比分判定 |

### 比分驗證規則（FR-005，核心邏輯、必有單元測試）

- 勝方 ≥ 21 且領先 ≥ 2；或 30:29（29 平後 30 分封頂）。
- 不允許平手、不允許 21:20 等領先不足、單局任何一方不得超過 30。
- 兩局先勝即整場結束；GameNo=3 僅允許存在於前兩局 1:1 時（FR-003）。

## bse.Rubber — 團體賽點次

固定 5 點制：3 單打 + 2 雙打（FR-004）。

| 欄位 | 型別 | 說明 |
|------|------|------|
| Id | int PK | |
| ScoreRecordId | int FK→ScoreRecord | |
| RubberNo | tinyint | 1–5；(ScoreRecordId, RubberNo) 唯一 |
| RubberType | tinyint | 1 單打、2 雙打 |
| WinnerSide | tinyint NULL | 由系統依該點各局判定 |
| IsNotPlayed | bit | 先取 3 點後剩餘點次標記「未賽」（FR-004） |

### 排點規則（FR-004 + spec 假設，核心邏輯、必有單元測試）

- 每點次每方出賽人數：單打 1、雙打 2；同一點次內選手不得重複。
- 每位選手至多出賽 2 點，且至多 1 單、1 雙。
- 任一隊取得 3 點即判定獲勝；剩餘點次可標記 IsNotPlayed，不強制輸入。

## bse.RubberPlayer — 點次出賽選手

| 欄位 | 型別 | 說明 |
|------|------|------|
| Id | int PK | |
| RubberId | int FK→Rubber | |
| Side | tinyint | 1／2 |
| PlayerId | int FK→Player | 須屬於該場次該方之 SideMember |

## bse.EvidenceImage — 成績單佐證圖檔

每筆成績送審前必有至少一張（FR-018）；OCR 辨識影像自動帶入
（SourceKind=2），可替換或補充。

| 欄位 | 型別 | 說明 |
|------|------|------|
| Id | int PK | |
| ScoreRecordId | int FK→ScoreRecord | |
| FileName | nvarchar(255) | |
| ContentType | nvarchar(100) | image/jpeg、image/png |
| Data | varbinary(max) | 上限 10 MB（R-06） |
| SourceKind | tinyint | 1 人工上傳、2 OCR 辨識影像帶入 |
| UploadedBy | nvarchar(50) | |
| UploadedAt | datetime2 | |

## bse.AuditEntry — 操作紀錄

Append-only；不可更新、不可刪除（FR-012）。

| 欄位 | 型別 | 說明 |
|------|------|------|
| Id | int PK | |
| SessionId | int FK→MatchSession | 供輸入畫面依場次檢視歷程 |
| ScoreRecordId | int FK→ScoreRecord NULL | |
| Actor | nvarchar(50) | 操作者姓名 |
| ActorRole | tinyint | 1 輸入者、2 覆核者 |
| Action | tinyint | 建立草稿／自動儲存／送審／通過／駁回／拉回／OCR 帶入／爬蟲帶入／上傳佐證… |
| FromStatus / ToStatus | tinyint NULL | 狀態轉移紀錄 |
| Detail | nvarchar(max) NULL | JSON 快照（送審當下比分等，供歷次送審對照） |
| At | datetime2 | |

## 草稿與並行（FR-014、FR-015）

- 草稿＝Status=1 的 ScoreRecord 及其子資料；自動儲存即更新該紀錄
  （R-10）。Action=自動儲存的 AuditEntry 以節流方式記錄（避免噪音，
  每次送審週期僅記首末）。
- 所有寫入操作帶 RowVersion；不符時回衝突訊息，由後送出者重載
  （R-09）。

## 種子資料（database/seed/）

三場示範場次（FR-001）：

1. 一般女生組 團體賽 決賽 — 場次 3，2026/05/04 11:00（兩隊各 ≥4 名選手，
   可演練 3 單 2 雙排點與重複限制）
2. 公開男生組 單打 決賽 — 場次 4，2026/05/01 09:00
3. 公開男女混合組 雙打 決賽 — 場次 4，2026/05/01 08:30
