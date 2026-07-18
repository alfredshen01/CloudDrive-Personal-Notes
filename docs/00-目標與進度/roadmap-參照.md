# CloudDrive 現況分析與未來規劃

> 2026-07-07。本文件是專案的現況盤點與路線圖,供報告/答辯與後續開發定向。
> 資料來源:progress.md、DEC-001~033、E7/E8 實驗數據、各任務文件。

## 一、現況分析

### 產品層:功能面基本完工

- **核心雲端硬碟 28 模組全數完成**:auth / drive / upload / download / preview / share /
  trash / search / file-version / activity-log + 前端 10 模組 + 整合驗收。搜尋含全文
  (tsvector)+ pgvector 語意搜尋。
- **三個擴充系統完成**:AI 助理(HARNESS 引擎 + Workflow 管線 + 自我撰寫技能沙箱)、
  eval harness(E1–E4)、時光機快照(S1–S5)。
- 測試:618 unit + 40 integration 全綠、無隱藏 skip/xfail;33 條 DEC 決策全有記錄。

**判讀:專案已過「做功能」階段,主戰場轉為「AI 助理的可靠性與體驗」。**

### 可靠性層:三個「量化→修正→驗證」循環完成,瓶頸已轉移

| 問題 | 對策 | 狀態 |
|---|---|---|
| 跳針卡死(temp=0 之 greedy decoding repetition loop) | DEC-031 num_predict + 非零溫度 | ✅ 有界失敗 |
| 幻覺技能名 | DEC-032 schema enum(grammar 級不可生成) | ✅ 根治 |
| 殘餘跳針(thinking 段) | DEC-033 planner think:false | ✅ 歸零,快 ~10 倍 |
| **規劃品質(寫入意圖遺漏)** | — | ❌ 困難集 M3/M5 仍 47%,現任瓶頸 |
| **多輪失憶** | proposal-assistant-memory(待實作) | ❌ 使用者實測痛點 |

### 技術債清單(誠實盤點)

| 債 | 嚴重度 | 出處 |
|---|---|---|
| 助理無對話記憶(有存沒回讀) | 高(使用者體感) | proposal-assistant-memory |
| planner 寫入類規劃 47% | 高(功能可靠性) | E8 |
| codegen 無系統化 pass-rate(僅 spot-check)、對 context 大小敏感 | 中 | E8 後記 |
| Ollama 單併發級聯(忙碌時體驗未定義,需產品決策) | 中 | — |
| 生成/儲存 schema 分家(version/handler 程式注入) | 低(乾淨化) | proposal-planner-skill-enum |
| 搜尋 backfill 未背景自動化 | 低 | progress.md |
| 時光機還原硬配額檢查待補 | 低 | progress.md |

### 流程層(專案最大的隱形資產)

文件先行 + DEC 決策鏈 + 真模型驗證鐵律 + eval 基礎設施(sweep 每輪秒級)。
**「量化→修正→驗證」×3 的完整循環(DEC-031/032/033)是最強的工程敘事,勝過任何單一功能。**

## 二、未來規劃(路線圖)

### 近期——收斂與合併

1. **開 PR 合併 `feat/codegen-structured-output`**(7 commits:codegen 保護 + DEC-033 + E8)。
   現在就開,別再疊——PR 越大越難審。記憶另開新分支。

### 中期——記憶 + 多輪驗證閉環

2. **記憶 v1 前置驗證**:真模型 sanity check `tool` 角色訊息 gemma4 是否正常消化
   (proposal 裡唯一未驗證的假設,先驗再定 detailed-design,避免返工)。
3. **記憶 v1 實作**(新分支):照 proposal-assistant-memory——planner 加 history →
   service 透傳 → router 載入 + 寫 tool 摘要訊息 → `assistant_history_max_messages=12`。
4. **eval 升級為多輪 + 記憶落地後重跑 sweep**:E8 數據全是單輪量的,加 6 輪歷史後 prompt
   分佈改變,必須重驗單輪不退化 + 新增多輪指涉案例(第四個驗證循環,亦為記憶 before/after 素材)。
5. **planner 寫入意圖 prompt 工程**:排記憶之後——記憶可能無需額外投入即修復部分多輪 case,
   先落地再量,避免無效投入。目標把困難集 47% 往上推,用 sweep 快速迭代。

### 遠期(視報告時程取捨)

6. **併發行為**:先要產品決策(忙碌時排隊 / 拒絕 / 提示),再實作。
7. **小而乾淨的收尾**:schema 分家、codegen 系統化 pass-rate、搜尋 backfill 自動化、
   時光機還原配額——適合填零碎時間。

### 明確「不做」的(避免發散)

- 不換模型、不上雲端 API 當主力——DEC-018 的本地隱私主軸是專案識別。
- 不做 agentic loop / DAG 並行——DEC-029/030 已議決,無新數據支持重開。
- 報告前不開新的大功能面——剩餘時間投「可靠性數據 + 記憶體驗」報酬率最高。

## 三、一句話總結

**功能已完工,專案價值來自「一個小模型如何被工程機制馴服」的完整證據鏈;接下來補上記憶
(體驗)與寫入規劃(可靠性)兩個實測缺口,並讓 eval 進化到多輪,故事就完整了。**
