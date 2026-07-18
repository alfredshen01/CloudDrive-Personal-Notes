# CloudDrive 個人筆記庫

alfred 的個人學習與開發筆記，涵蓋前置期全端訓練（2026-05 起）與 CloudDrive 專案的 In-App AI 助理（harness）開發歷程。本 repo 只放筆記（程式碼在共用的 CloudDrive repo）；GitHub Pages 於 push 後自動建站，以本頁為導覽入口。

## 00 目標與進度

專案目標、階段追蹤與每週摘要——想快速掌握現況從這裡開始。

- [目標總覽](00-目標與進度/目標總覽.md) — 總目標、目標×階段矩陣、滾動待辦、里程碑
- [每週概述](00-目標與進度/每週概述.md) — 每週 2~4 行摘要，連到各週日誌
- [roadmap-參照](00-目標與進度/roadmap-參照.md) — 2026-07-07 的現況盤點與路線圖（自 cloud_drive/doc/roadmap.md 複製）

## 01 日誌

逐週（前置期為逐日）的完成項目、問題三段式（緣由／原因／解決）、測試與驗證、學到的概念。W1~W5 為初稿（由對話紀錄與 commit 回補），待本人逐週校正。

- [2026-05 前置期：全端基礎](01-日誌/2026-05-前置期-全端基礎.md) — 三層架構→Docker→部署→CI/CD→HTTPS
- [2026-06 前置期：Final_Project](01-日誌/2026-06-前置期-FinalProject.md) — 圖片轉換平台改版
- [W1（0616~0622）](01-日誌/W1-0616~0622.md) — 加入 CloudDrive、建立 eval harness
- [W2（0623~0629）](01-日誌/W2-0623~0629.md) — 環境建置、助理四功能、DB 對帳
- [W3（0630~0706）](01-日誌/W3-0630~0706.md) — 約束解碼、以執行結果回報、失敗隔離、防跳針兩循環
- [W4（0707~0713）](01-日誌/W4-0707~0713.md) — think:false、記憶 v1、recall 診斷閉環
- [W5（0714~0720）](01-日誌/W5-0714~0720.md) — V0.2 需求審查、證據分級、筆記整併

## 02 筆記：harness（主責領域）

CloudDrive AI 助理的架構、研究歷程與評測系統。

- [harness筆記](02-筆記-harness/harness筆記.md) — 三部結構：架構與心法／研究歷程與數據（6/17~7/09）／答辯 Q&A 與速記
- [eval系統速覽](02-筆記-harness/eval系統速覽.md) — 評測系統一頁版，詳細見 99 正式報告
- [V02審查發現與測試現況](02-筆記-harness/V02審查發現與測試現況.md) — 9 項審查發現＋五層測試現況＋驗證計畫
- [openai-oauth-problem](02-筆記-harness/openai-oauth-problem.md) — OpenAI OAuth 問題記錄（自 cloud_drive/doc 複製）

## 03 筆記：前後端

從日常開發記錄蒸餾出的可重用概念。

- [後端概念](03-筆記-前後端/後端概念.md) — Alembic／pgvector／測試模式／三表對帳／JWT+bcrypt
- [前端概念](03-筆記-前後端/前端概念.md) — React hooks／受控元件／CORS／Vite／CloudDrive 前端架構
- [開發筆記精選](03-筆記-前後端/開發筆記精選.md) — 固定欄位式結論：json_schema 傳遞鏈／回報與授權／失敗隔離／temperature／registry 與沙盒／外部模型實測

## 04 筆記：部署維運

- [部署與環境](04-筆記-部署維運/部署與環境.md) — Docker 陷阱／nginx+HTTPS 完整流程／CI-CD／VPS vs PaaS／開發環境事實

## 99 報告

正式報告的 Markdown 版（圖表【圖N】待補，正式版以 docx 為準）。

- [harness架構報告](99-報告/harness架構報告.md)
- [測試系統報告](99-報告/測試系統報告.md)

## 外部參照（隊友著作，不搬檔）

CloudDrive 共用 repo 內、由隊友撰寫或混合著作的文件：

- [eval-prompt-log](https://github.com/billwu101/CloudDrive/blob/main/doc/eval-prompt-log.md) — 測試 prompt 索引（billwu101 為主的混合著作）
- [claudeAIChatHistory](https://github.com/billwu101/CloudDrive/blob/main/doc/claudeAIChatHistory.md) — claude.ai 對話摘要（作者：billwu101）
- [demo-compress-skill](https://github.com/billwu101/CloudDrive/blob/main/doc/demo-compress-skill.md) — 壓縮 skill demo（作者：billwu101）
