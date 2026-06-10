# HTTP 介面契約: 羽球成績輸入系統

**Date**: 2026-06-10
**形式**: ASP.NET MVC 5 — Razor 頁面（server-rendered HTML）+ JSON AJAX
端點（JsonResult，見 research.md R-01/R-02）。資料形狀對應 data-model.md。

## 共用約定

- 身分：加密 cookie 攜帶 `{ name, role }`（R-11）；未設定者一律導向
  角色選擇頁。JSON 端點未帶身分時回 401。
- JSON 回應外層格式（所有 AJAX 端點一致，憲章原則 III）：

  ```json
  // 成功
  { "ok": true, "data": { } }
  // 失敗（message 為白話、可行動訊息；field 指出問題欄位，可省略）
  { "ok": false, "message": "第 2 局比分 21:20 不符合規則：20 平後須領先 2 分。請修正其中一方分數。", "field": "games[1]" }
  ```

- 並行控制：讀取成績時回 `rowVersion`（base64），所有寫入需原樣帶回；
  不符回 `409` + 白話衝突訊息（FR-015）。
- 寫入端點皆要求 AntiForgeryToken。

## 頁面（GET，Razor HTML）

| 路徑 | 用途 | 對應 |
|------|------|------|
| `/` | 角色選擇 + 姓名輸入（第一個畫面） | FR-016 |
| `/sessions` | 場次選擇（依日期、場地、場次列出；含已駁回待重輸提示） | FR-001、US3-駁回回流 |
| `/score/{sessionId}` | 成績輸入畫面（預設人工模式，含爬蟲／OCR 帶入鈕、操作歷程、草稿還原） | FR-017、FR-012 |
| `/review` | 覆核清單（「待審核」／「已審核」頁籤） | FR-019 |
| `/review/{scoreRecordId}` | 單筆覆核畫面（成績 + 佐證圖 + 爬蟲連結） | FR-007 |
| `/evidence/{imageId}` | 佐證圖檔內容（image/*；輸入與覆核畫面引用） | FR-007、FR-018 |

## JSON 端點

### POST `/api/identity` — 設定身分

```json
// 請求
{ "name": "王小明", "role": "recorder" }   // recorder | reviewer
// 回應 data
{ "redirect": "/sessions" }                 // reviewer → "/review"
```

姓名空白即拒絕（field: "name"）。

### GET `/api/sessions/{id}` — 場次資料帶入

回應 data：場次資訊 + 兩方名單 + 既有成績紀錄（含狀態、各局比分、
點次、佐證圖 id 清單、rowVersion）。既有紀錄存在即進入編輯
（Edge case：不建立重複資料）；已駁回者帶原內容（FR-006）。

### POST `/api/score/draft` — 草稿自動儲存（FR-014）

```json
// 請求（單打/雙打）
{ "sessionId": 2, "rowVersion": "AAAA…", "source": "manual",
  "games": [ { "gameNo": 1, "side1": 21, "side2": 18 } ] }
// 請求（團體）另含
{ "rubbers": [ { "rubberNo": 1, "rubberType": "S",
    "side1PlayerIds": [5], "side2PlayerIds": [9],
    "games": [ … ], "isNotPlayed": false } ] }
// 回應 data
{ "scoreRecordId": 7, "rowVersion": "AAAB…", "savedAt": "2026-06-10T14:00:03" }
```

草稿儲存僅做格式檢查（數值範圍），完整規則驗證於送審時強制；
但端點即時回傳逐局／逐點的驗證結果供 UI 即時提示（FR-005）。

### POST `/api/score/submit` — 送出待覆核

請求同 draft（完整內容 + rowVersion）。伺服器端強制：

1. 比分規則全數合法（FR-005）、勝方可判定（FR-003/004）；
2. 至少一張佐證圖檔（FR-018），否則
   `message: "請先上傳成績單圖檔再送出。"`，`field: "evidence"`；
3. 團體賽排點規則（FR-004）。

回應 data：`{ "status": "pending" }`。狀態 草稿/已駁回 → 待覆核，寫入
AuditEntry（含快照）。

### POST `/api/score/{id}/evidence` — 上傳佐證圖（multipart）

`multipart/form-data`，欄位 `file`；JPEG/PNG、≤10 MB。
回應 data：`{ "imageId": 31, "url": "/evidence/31" }`。

### DELETE `/api/score/{id}/evidence/{imageId}` — 移除佐證圖

僅草稿／已駁回狀態允許；OCR 自動帶入之影像可被替換（FR-018）。

### POST `/api/ocr/recognize` — OCR 影像判讀（FR-009）

`multipart/form-data`，欄位 `file` + `sessionId`。同步呼叫 Gemini
（≤30 秒，UI 顯示處理中）。

```json
// 回應 data
{ "evidenceImageId": 32,            // 辨識影像已自動帶入佐證（FR-018）
  "prefill": { "games": [ { "gameNo": 1, "side1": 21, "side2": 18 } ] },
  "confidence": { "games[0].side1": 0.95, "games[0].side2": 0.62 },
  "lowConfidenceThreshold": 0.8,
  "conflicts": [ "辨識到的選手「林○○」不在本場次名單中，請人工確認。" ] }
// 失敗
{ "ok": false, "message": "影像太模糊，無法辨識。請改用人工輸入，或重拍後再試一次。" }
```

預填僅寫入前端表單，不直接落庫；送審仍走 submit 全套驗證
（spec：未經人工確認不得送審）。

### GET `/api/crawler/prefill?sessionId={id}` — 爬蟲預填（FR-010）

依場次「日期、時間、場次編號」比對來源頁（≤15 秒）。

```json
// 回應 data
{ "prefill": { "games": [ … ] },
  "sourceUrl": "http://172.105.210.232/liveresultst/…",
  "matchedKey": { "date": "2026-05-01", "time": "09:00", "matchNo": 4 } }
// 查無 / 多筆 / 來源異常（皆 ok:false + 白話訊息，保留人工路徑）
{ "ok": false, "message": "即時成績網站查無這個場次的資料。請改用人工輸入或 OCR。" }
```

### GET `/api/review/list?tab=pending|reviewed` — 覆核清單（FR-019）

回應 data：陣列，每筆含場次資訊、輸入者、來源（人工/OCR/爬蟲）、
狀態、送審時間。

### POST `/api/review/{id}/approve` — 通過

`{ "rowVersion": "…" }`。伺服器強制四眼原則：覆核者姓名 = 輸入者姓名
即拒絕（FR-008）：`message: "這筆成績是您本人輸入的，依規定不能由本人通過。"`。
待覆核 → 已發佈。

### POST `/api/review/{id}/reject` — 駁回

`{ "rowVersion": "…" }`。待覆核 → 已駁回；輸入流程中該場次顯示
待重輸提示（US3）。

### POST `/api/review/{id}/recall` — 拉回重審（FR-019）

`{ "rowVersion": "…" }`。已發佈/已駁回 → 待覆核；寫入 AuditEntry。

### GET `/api/sessions/{id}/audit` — 操作歷程（FR-012）

回應 data：陣列（時間倒序）：`{ "at", "actor", "actorRole", "action",
"fromStatus", "toStatus", "summary" }`。

## 錯誤碼約定

| HTTP | 情境 |
|------|------|
| 200 | 一律用於業務成功與業務失敗（`ok` 區分），UI 處理單純化 |
| 401 | 未設定身分 → 導向 `/` |
| 409 | RowVersion 衝突（FR-015） |
| 413 | 圖檔超過 10 MB |
| 500 | 未預期錯誤；回白話訊息並記錄伺服器日誌 |
