# CloudDrive 個人筆記庫

alfred 的個人學習與開發筆記，涵蓋前置期全端訓練（2026-05 起）與 CloudDrive 專案的 In-App AI 助理（harness）開發歷程。本 repo 只放筆記（程式碼在共用的 CloudDrive repo）；GitHub Pages 於 push 後自動建站，以本頁為導覽入口。

## 00 目標與進度

專案目標、階段追蹤與每週摘要——想快速掌握現況從這裡開始。

- [目標總覽](00-目標與進度/目標總覽.md) — 總目標、目標×階段矩陣、滾動待辦、里程碑
- [每週概述](00-目標與進度/每週概述.md) — 每週 2~4 行摘要，連到各週日誌

## 01 日誌

逐週（前置期為逐日）記錄：本週主軸、完成項目（有問題的項目附 問題／原因／解決）、測試與驗證、待辦。W1–W5 由對話紀錄與 commit 補寫，逐週校對中。

- [2026-05 前置期：全端基礎](01-日誌/2026-05-前置期-全端基礎.md) — 三層架構→Docker→部署→CI/CD→HTTPS
- [2026-06 前置期：Final_Project](01-日誌/2026-06-前置期-FinalProject.md) — 圖片轉換平台改版
- [W1（0616–0622）](01-日誌/W1-0616~0622.md) — 加入 CloudDrive、建立評測系統
- [W2（0623–0629）](01-日誌/W2-0623~0629.md) — 環境建置、助理四功能、執行稽核
- [W3（0630–0706）](01-日誌/W3-0630~0706.md) — 格式約束補全、以執行結果回報、失敗隔離、防跳針兩輪
- [W4（0707–0713）](01-日誌/W4-0707~0713.md) — 關閉 thinking、記憶 v1、回想診斷閉環
- [W5（0714–0720）](01-日誌/W5-0714~0720.md) — V0.2 需求審查、證據分級、筆記整併

## 02 筆記：harness（主責領域）

CloudDrive AI 助理的架構與評測系統——本專案的兩份正本。

- [Harness 架構](02-筆記-harness/Harness架構.md) — 六元件↔九模組、Workflow 管線、模型路由與外送隱私檢查、對話記憶、技能生成、決策鏈、文獻定位
- [Harness 評測](02-筆記-harness/Harness評測.md) — 分兩部：評測系統設計（如何測）／測試結果與分析（防跳針三循環、對話記憶、能力上限）

## 03 筆記：前後端

從日常開發記錄蒸餾出的可重用概念。

- [後端概念](03-筆記-前後端/後端概念.md) — Alembic／pgvector／測試模式／執行稽核／JWT+bcrypt
- [前端概念](03-筆記-前後端/前端概念.md) — React hooks／受控元件／CORS／Vite／CloudDrive 前端架構
- [開發筆記精選](03-筆記-前後端/開發筆記精選.md) — 固定欄位式結論：json_schema 傳遞鏈／回報與授權／失敗隔離／temperature／registry 與沙盒／外部模型實測

## 04 筆記：部署維運

- [部署與環境](04-筆記-部署維運/部署與環境.md) — Docker 陷阱／nginx+HTTPS 完整流程／CI-CD／VPS vs PaaS／開發環境事實

## 外部參照（隊友著作，不搬檔）

CloudDrive 共用 repo 內、由隊友撰寫或混合著作的文件：

- [eval-prompt-log](https://github.com/billwu101/CloudDrive/blob/main/doc/eval-prompt-log.md) — 測試 prompt 索引（billwu101 為主的混合著作）
- `claudeAIChatHistory.md` — claude.ai 對話摘要（作者：billwu101;此檔已不在源 repo 的 [`doc/` 目錄](https://github.com/billwu101/CloudDrive/tree/main/doc)）
- `demo-compress-skill.md` — 壓縮 skill demo（作者：billwu101;此檔已不在源 repo 的 [`doc/` 目錄](https://github.com/billwu101/CloudDrive/tree/main/doc)）
