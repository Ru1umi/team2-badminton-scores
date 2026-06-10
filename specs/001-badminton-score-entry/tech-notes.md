# 技術備註（供 /speckit.plan 的 Technical Context 使用）

> 本檔記錄使用者於 /speckit.specify 階段提供的實作層資訊。
> 依規格撰寫原則，這些細節不寫入 spec.md，於規劃階段正式納入 plan.md。

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

## 歷史紀錄

- 2026-06-10：初次提供爬蟲網址 `matchesst.html`，後更正為 `liveresultst.html`；
  OCR 指定 Gemini `gemini-3.1-flash-lite`。
