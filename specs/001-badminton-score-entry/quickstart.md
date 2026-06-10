# Quickstart: 羽球成績輸入系統

**Date**: 2026-06-10
**用途**: 環境準備、啟動方式與端到端驗證情境。
契約細節見 [contracts/http-api.md](contracts/http-api.md)，
資料結構見 [data-model.md](data-model.md)。

## 先決條件

- Windows 10/11 或 Windows Server，已安裝 **.NET Framework 4.8**
  Developer Pack
- **Visual Studio 2022**（ASP.NET 與網頁開發工作負載；含 IIS Express）
- 可連線之 **SQL Server**：內網 `192.168.66.99` / 資料庫
  `Badminton_T2_Dev`（或本機 SQL Server Express 建立同名資料庫開發）
- **Gemini API 金鑰**（OCR 功能；無金鑰時其餘功能仍可驗證）
- 爬蟲驗證需可連到
  `http://172.105.210.232/liveresultst/liveresultst.html?openid=755276`
  （來源為賽事期間之外部頁面，離線時以本機 HTML fixture 驗證解析器）

## 初始設定

1. 還原 NuGet 套件（Visual Studio 開啟方案自動還原）。
2. 機密設定：複製 `src/BadmintonScore.Web/Secrets.config.example` 為
   `Secrets.config`（已被 .gitignore 排除），填入：
   - 連線字串 `UMallSport`（ADO.NET 格式）
   - `Gemini:ApiKey`
3. 建立資料庫物件與種子資料（依序執行）：

   ```text
   database/schema/001_create_schema.sql … （bse schema 與各資料表）
   database/seed/001_demo_sessions.sql     （三場示範場次與選手名單）
   ```

## 啟動

- Visual Studio 設 `BadmintonScore.Web` 為啟始專案 → F5（IIS Express）。
- 首頁即角色選擇畫面。

## 執行測試

- Visual Studio Test Explorer 執行全部測試，或命令列：

  ```text
  vstest.console tests\BadmintonScore.Core.Tests\bin\Debug\BadmintonScore.Core.Tests.dll
  ```

- 合併門檻：全數通過、無 analyzer 警告（憲章原則 I、II）。

## 驗證情境

### V1 人工輸入單打（US1，MVP）

1. 首頁輸入姓名「測試輸入員」、選「成績輸入者」→ 進入場次清單，
   可見三場示範場次。
2. 選「公開男生組 單打 決賽（場次四，2026/05/01 09:00）」→
   選手與場次資訊已自動帶入，畫面為人工輸入模式，
   且可見「爬蟲自動抓取」「OCR 影像判讀」兩鈕。
3. 輸入第 1 局 21:18、第 2 局 21:15 → 系統自動判定勝方並顯示，
   無第三局必填欄位。
4. 輸入 21:20 → 即時出現白話錯誤訊息，無法送出。
5. 未上傳圖檔按送出 → 提示需先上傳成績單圖檔。
6. 上傳一張 JPEG → 送出成功，狀態「待覆核」。
7. 開啟操作歷程 → 可見建立、送審紀錄。

### V2 團體賽輸入（US2）

1. 選「一般女生組 團體賽 決賽（場次三，2026/05/04 11:00）」。
2. 依 5 點制（3 單 2 雙）排點；嘗試讓同一選手排第 3 點單打與
   另一點單打 → 系統阻止並說明。
3. 逐點輸入比分至某隊先取 3 點 → 自動判定獲勝隊伍，
   剩餘點次可標記「未賽」後送出。

### V3 覆核與發佈（US3）

1. 以另一姓名「測試覆核員」選「成績覆核者」→ 預設「待審核」頁籤
   可見 V1 送出的紀錄（含來源「人工」）。
2. 開啟該筆 → 可見成績與佐證圖檔。按「通過」→ 移入「已審核」，
   狀態「已發佈」。
3. 於「已審核」對該筆執行「拉回」→ 回到「待審核」；改按「駁回」→
   狀態「已駁回」。
4. 回到輸入者流程 → 場次清單出現待重輸提示，開啟後表單帶原內容，
   修改後可再次送審。
5. 以「測試輸入員」身分進覆核流程開啟自己輸入的紀錄 →
   「通過」被拒（四眼原則訊息）。

### V4 OCR 預填（US4）

1. 於輸入畫面點「OCR 影像判讀」、上傳清晰手寫成績單照片 →
   表單預填，低信心欄位有標示；該影像自動列為佐證圖。
2. 上傳模糊影像 → 白話失敗訊息，引導人工輸入。

### V5 爬蟲預填（US5）

1. 於輸入畫面點「爬蟲自動抓取」→ 依日期/時間/場次比對成功時
   表單預填並顯示來源連結；逐欄確認後送出，覆核畫面可見
   來源「爬蟲」與連結。
2. 對來源查無的場次操作 → 白話「查無資料」訊息，人工路徑不受影響。

### V6 草稿與並行（FR-014、FR-015）

1. 輸入一半關閉分頁 → 重新進入該場次，內容已還原（伺服器草稿）。
2. 兩個瀏覽器同開同場次，先後送出 → 後送出者收到衝突提示，
   重新載入後可繼續。

### V7 裝置支援（SC-008）

以手機（直式）與平板瀏覽器跑 V1 與 V3 主流程：全程可讀、可點按、
無水平捲動。

## 效能查核（SC-006、憲章原則 IV）

- 操作回饋 < 0.2s、頁面切換 < 1s（DevTools Network/Performance 抽查）。
- OCR ≤ 30s、爬蟲 ≤ 15s，期間 UI 顯示處理中狀態。
