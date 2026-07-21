# Harness 架構：CloudDrive AI 助理的 Agent Harness 設計

> 這是本專案 Agent Harness 的架構正本。測試與數據見 [Harness 評測](Harness評測.md)。
> 底稿與細節出處：cloud_drive 共用 repo 的 [`doc/` 目錄](https://github.com/billwu101/CloudDrive/tree/main/doc)。主要見 `detailed-design/03-architecture.md`（整體架構）、`detailed-design/08-assistant-engine.md`（assistant 引擎），決策見 `detailed-design/appendix-a-decisions.md`。

---

## 0. 核心原則：機制優先於模型自覺（mechanism over model self-awareness）

- **原則**：凡是能用程式、規則、文法約束保證的正確性,就不交給「模型記得要做對」;把要求從「用 prompt 期望模型遵守」,變成「用機制約束模型、確保正確性」。
- **原因**：本專案採用本地小模型 gemma4:26b,能力有限、輸出不穩定,要求它自我約束並不可靠;將正確性的保證交由程式機制承擔,才能得到穩定的行為。

## 1. 目的與定位

CloudDrive 是一套自架雲端硬碟,內嵌一個在本機運行、不連到雲端的 AI 助理（採用本地模型 gemma4:26b）,
讓使用者以自然語言操作檔案（搜尋、整理、壓縮、還原等）。本文說明助理背後的
**Agent Harness**——把一個語言模型包裝成可靠、可治理、可驗證的代理系統的架構。

設計目的,是讓參數量不大的本地小模型也能穩定完成基本的檔案操作,不必依賴更大的雲端模型。

## 2. 整體架構：六核心元件 ↔ 九實作模組

本專案的 harness 對應到 agent 系統文獻歸納的六個通用核心元件（E/T/C/S/L/V）。其中五個是
「引擎」——在請求路徑上運行的 runtime（Execution Loop、Tool/Skill Registry、Context
Manager、State Store、Lifecycle Hooks）;第六個 Evaluation Interface 是離線評測工具,不在
請求路徑上（圖 1 畫在旁側）。

在程式碼層,這五個 runtime 元件再細分為九個引擎模組。拆分不是為了湊數、也不是每個元件平均
切開,而是**沿著各元件內部真正不同質、需要各自演化與獨立測試的邊界切**：

| 核心元件 | 模組數 | 拆成的模組 | 拆分依據 |
|---|---|---|---|
| **E** Execution Loop | 2 | 主迴圈 while loop／子代理 sub-agent | 主迴圈負責一般請求（送訊息→解析→執行→回填）;子代理為「現場生成技能」而派生的有界執行環境,自帶 prompt 組裝與收斂邊界。二者控制流不同,合併將使主迴圈邏輯與 codegen 細節相互纏繞,且無法各自測試 |
| **T** Tool/Skill Registry | 2 | 登錄機制 registry／內建技能 built-in | registry 為通用的 manifest 與 dispatch 機制,對內建、自建、現場生成三種來源一視同仁;built-in 僅為其中一種來源（具體技能內容）。機制與內容分離後,新增技能無須改動 dispatch |
| **C** Context Manager | 2 | 脈絡管理／system prompt 組裝 | 脈絡管理處理 token 預算、裁切、工具輸出精簡,與內容無關、可重用;prompt 組裝因 agent 而異（規劃器與 codegen 各一套）,於程式碼中內嵌於各 agent |
| **S** State Store | 1 | session 持久化（不拆） | 職責單一——依 `user_id` 隔離的持久化,內部無異質邊界;職責單一者不予拆分 |
| **L** Lifecycle Hooks | 2 | 治理攔截點 hooks／強制機制 permissions & safety | hooks 界定「在何處治理」（攔截點）;permissions/codeguard/sandbox 界定「治理什麼」（強制機制）。二者分離後,新增攔截點無須重寫權限邏輯,權限與沙箱亦可各自測試 |

五個 runtime 元件合計 2＋2＋2＋1＋2 = 9 個引擎模組（V Evaluation Interface 是離線評測工具,
不在請求路徑上,不屬這九個）。九個模組不在同一抽象層級——有主元件（while loop）、有子來源
（built-in 是 registry 的技能來源之一）、也有橫切治理（permissions & safety）。這就是「概念上
分六層、程式碼上分九層」的實質:分層依測試與演化邊界,而非另立一套架構。（後文 `DEC-NNN` 為
本專案的設計決策編號,完整清單見 §7。）

【圖 1：Harness Runtime 六元件架構圖】

```mermaid
flowchart TB
  U["使用者自然語言需求"] --> P
  subgraph P["Workflow Pipeline（要做什麼）"]
    direction LR
    p1["需求解析"] --> p2["規劃"] --> p3["技能檢查"] --> p4["權限/安全"] --> p5["確認"] --> p6["執行"] --> p7["記錄"]
  end
  P --> R
  subgraph R["Agent Harness Runtime（怎麼可靠地跑）"]
    direction TB
    E["E Execution Loop<br/>while loop・workflow executor・<br/>bounded sub-agent・model interface"]
    T["T Tool / Skill Registry<br/>內建・自建・現場生成技能"]
    C["C Context Manager<br/>裁切・prompt 組裝・歷史回放"]
    S["S State Store<br/>sessions・skills・workflows"]
    L["L Lifecycle Hooks 治理層<br/>permissions・codeguard・sandbox・送外部模型前的隱私檢查"]
  end
  V["V Evaluation Interface<br/>離線 evaluation harness（不在請求路徑上）"] -. "評測請求" .-> R
  R --> SV["CloudDrive Service Layer（DEC-017）"]
  SV --> DB[("PostgreSQL + Storage")]
```

> 說明：Workflow Pipeline 在最上、六元件 Runtime 在中、V（評測介面）
> 畫在**旁側**（虛線箭頭指向系統,表示它是離線工具、不在請求路徑上）,最下為 Service 層 →
> DB/Storage。此為全篇的核心主圖。原型圖見 [`doc/` 目錄](https://github.com/billwu101/CloudDrive/tree/main/doc)的 `detailed-design/03-architecture.md`。

| 元件 | 職責（白話） | 主要實作檔 |
|---|---|---|
| **E** Execution Loop | 主迴圈：送訊息→解析→執行→回填;workflow 執行器管步驟相依/錯誤策略;可派生 bounded sub-agent;**模型介面（ModelRouter）亦歸此** | `service.py`、`workflow.py`、`subagent.py`、`llm/router.py` |
| **T** Tool / Skill Registry | 有哪些技能可用（內建、使用者自建、現場生成）;manifest 定義 schema/權限/handler | `skills/registry.py`、`skills/manifest.py`、`skills/authoring.py` |
| **C** Context Manager | 塞什麼進 prompt：token 預算、裁切、工具輸出瘦身、技能清單注入、system prompt 組裝、**讀取歷史對話** | `context.py`、`planner.py`、`memory.py` |
| **S** State Store | 存什麼（session/訊息/技能/workflow）,全依 `user_id` 隔離 | `repository.py` + migrations |
| **L** Lifecycle Hooks | 治理層攔截點;強制執行權限分層、核可、沙箱、稽核、**送外部模型前的隱私檢查** | `hooks.py`、`permissions.py`、`codeguard.py`、`sandbox.py` |
| **V** Evaluation Interface | 離線開發者工具：確定性斷言 + LLM judge + baseline 回歸,API/browser/exec 三模式 | `backend/eval/`（詳見 [Harness 評測](Harness評測.md)） |

**一個補充說明**：sub-agent 目前唯一實例是 CodegenSubAgent,不是通用多代理編排;準確的說法是
「主迴圈可派生 bounded sub-agent,目前實例化為 codegen 子代理」。

## 3. Workflow 執行管線（請求路徑）

助理處理一則自然語言請求的完整流程:

【圖 2：Workflow 執行管線流程圖】

```mermaid
flowchart TD
  A["需求解析"] --> B["規劃（結構化輸出,技能名 enum）"]
  B --> C{"技能齊全?"}
  C -- "缺技能" --> G["技能生成子流程（圖 3）"] --> D
  C -- "齊" --> D["權限 / 安全檢查"]
  D --> E["顯示計畫"]
  E --> F{"權限層級"}
  F -- "唯讀 fast-path" --> X["自動執行"]
  F -- "寫入 / 破壞性" --> Y{"使用者確認"}
  Y -- "同意" --> X
  Y -- "拒絕" --> Z["取消（計畫留存,不執行）"]
  X --> W["記錄：workflow run＋activity log"]
```

> 說明：確認節點分兩路：唯讀自動執行（fast-path）、寫入/破壞性走人工確認。
> 對應 [`doc/` 目錄](https://github.com/billwu101/CloudDrive/tree/main/doc)的 `detailed-design/08-assistant-engine.md`（Workflow 管線）。

關鍵設計:
- **結構化輸出（DEC-032）**：規劃階段用 json_schema 文法約束輸出格式,技能名以 enum 枚舉 →
  幻覺技能在約束解碼下即無法生成。
- **權限分層（DEC-019）**：唯讀自動 / 破壞性需確認 / 生成碼需核可+沙箱+稽核。
- **如實回報失敗（DEC-029）**：執行失敗如實回報 StepResult,不偽裝成功;僅在唯讀時做一次
  有限度重規劃,**不採 agentic loop**（弱模型下無界迴圈風險 > 收益）。
- **串行執行（DEC-030）**：並行（DAG,有向無環圖）延後,因所有技能共用同一個資料庫連線
  （AsyncSession）需先解決。

## 4. 模型路由與送外部模型前的隱私檢查

【圖 4：模型路由 + 送外部模型前的隱私檢查決策流程圖】
> 說明：此圖為決策流程。使用者在每則訊息自行選擇要用哪個模型（本機模型,或自己設定的
> 外部模型）;送出前先做隱私判斷,若內容敏感又無法去識別化,就不送到外部模型;選定的模型
> 是這次唯一的執行者,不會自動改用別的模型;若失敗,回報可區分的錯誤原因（連不到模型、
> 憑證被拒、用量額度耗盡）。對應 [`doc/` 目錄](https://github.com/billwu101/CloudDrive/tree/main/doc)的 `detailed-design/08-assistant-engine.md`（模型路由）。

- **使用者每則訊息自選模型**：可選本機模型（gemma4:26b）,或使用者自己設定、加密儲存的
  外部模型連線（每位使用者各自一份）。選定後,這次就只用該模型執行,不會再換別的模型。
- **送外部模型前的隱私檢查一律執行（DEC-023）**：即使使用者手動選擇外部模型,一旦偵測到敏感內容仍不會送出。
- 舊版的「本地模型失敗就自動改用外部模型」已改為由使用者手動選擇,僅在 `target=None` 的相容路徑保留。

## 5. 對話記憶子系統（7 月新增）

助理原本每一輪只把當前這則訊息交給規劃器,先前的對話雖然已存進資料庫,但不會再讀出來使用;
因此使用者在後續訊息中提到前面講過的內容時,助理接不上。記憶 v1 讓規劃時能讀取先前的對話。

【圖 5：對話記憶資料流圖】
> 說明：此圖為資料流程。左路 `/chat`：載入最近數則歷史對話 → 併入規劃器的輸入 → 執行 →
> 把結果摘要接回 assistant 訊息、存進資料庫;右路 `/confirm`：使用者手動確認後執行 →
> 結果摘要一樣寫回該 session。圖中標出歷史則數上限,以及「工具執行結果以 assistant 文字
> 形式帶入」。對應 [`doc/` 目錄](https://github.com/billwu101/CloudDrive/tree/main/doc)的 `detailed-design/08-assistant-engine.md` §8.14 對話記憶（原 `proposal-assistant-memory.md` 已於 2026-07-13 併入此處）。

- **讀取歷史**：載入最近 `assistant_history_max_messages`（預設 12）則對話,依
  `[system, *歷史, 當前訊息]` 的順序送入模型。
- **工具結果以 assistant 文字帶入（而非 tool 角色訊息）**：真實模型對照實驗顯示,gemma4
  無法解讀單獨的 tool 角色訊息（tool 角色 0/4、assistant 文字 4/4）→ 此為決策依據,且不需資料庫 migration。
- **確認後寫回**：`/confirm` 端點原本不寫入歷史 → 已修正為執行後把結果摘要寫回 session。
- **目前限制**：只讀取最近約 6 輪對話,每則摘要截斷至 200 字,僅限單一 session,且以時間先後
  （而非語意相關性）挑選訊息。這些上限是為控制送入模型的 token 量與延遲,避免長脈絡拖慢回應
  並超出 context window。

## 6. 技能生成與治理（自我撰寫技能）

缺技能時,助理可**現場生成**一個新技能,但受嚴格治理:

【圖 3：技能生成子流程圖】
> 說明：流程圖 codegen(結構化輸出產 skill) → codeguard(靜態安全檢查) → sandbox(隔離執行) →
> approve(使用者核可) → execute → ingest(安裝回 registry)。標示「生成也是 workflow 化的前置子流程」。
> 對應 [`doc/` 目錄](https://github.com/billwu101/CloudDrive/tree/main/doc)的 `detailed-design/08-assistant-engine.md`（技能生成子流程）、`appendix-a-decisions.md` DEC-019。

- codegen 以結構化輸出保證產出 `{name, description, version, code, ui}` 這組欄位（envelope）;其中 handler 與 version 由程式注入。
- 生成碼經 codeguard 靜態檢查 + sandbox 隔離執行 + 使用者核可 + 稽核,才安裝。

## 7. 設計原則沉澱（決策鏈）

本專案的可靠性來自一連串**資料驅動的決策**,每個都有根因與驗證（[Harness 評測](Harness評測.md) 詳述數據）:

| 決策 | 原則 |
|---|---|
| DEC-017 | 助理一律經 service 層,不直接碰 DB/FS |
| DEC-019 | 生成技能：核可 → 沙箱 → 稽核 |
| DEC-020 | session/技能持久化到 DB |
| DEC-029 | 失敗回覆由程式依執行結果組合（如實回報）+ 有限度重規劃,不採 agentic loop |
| DEC-031 | 結構化解碼防止重複生成（重複生成迴圈:模型重複生成同一段內容且無法停止,repetition loop;詳見 [Harness 評測](Harness評測.md) 第二部;做法為 num_predict〔輸出 token 上限〕 + 非零 temperature） |
| DEC-032 | schema enum：技能名以文法約束枚舉;約束解碼會在每個生成步驟把不在枚舉內的技能名遮罩掉、機率歸零,幻覺技能因此無法生成 |
| DEC-033 | planner 預設關閉 thinking（從根本消除重複生成迴圈、速度快約 10 倍） |

**共同原則**：把「請求模型遵守」的保證,提升為「模型無法違反」。

## 8. 限制與未來方向

**架構層限制**
- sub-agent 僅 codegen 一種實例,非通用多代理。
- 執行串行,DAG 並行未做（DEC-030 前置未解）。
- 斷線後無法取消：使用者放棄的請求仍佔用 Ollama 唯一的並行處理名額（實際使用中發現）。

**模型層限制**
- 只有單一本地模型（gemma4:26b）;規劃能力有上限（寫入意圖的規劃,非機制所能補足）——當前通過率見[目標總覽的技術債](../00-目標與進度/目標總覽.md)。

**未來可行方向**
- 工程可達成：斷線取消、記憶摘要格式修復、codegen 系統化跑分。
- 模型能力前沿（只能持續改善、沒有一勞永逸的解）：對 planner 的寫入意圖做 prompt 工程,用多輪評測迭代。
- 記憶 v2：摘要壓縮 → 語意檢索（復用搜尋的 pgvector）→ 跨 session。

## 9. 文獻定位

本 harness 對應 agent 系統六核心元件的通用框架;本專案的特色在於
**「本地小模型 + 以機制確保可靠性」**——用文法約束、num_predict、thinking 配置等機制,
把一個 26B 參數的本地模型調校到可用的可靠度,而不依賴更大的雲端模型。

**相關研究對照——StraTA（arXiv 2605.06642）**：一種 Agentic 強化學習框架,先從任務初始狀態
生成「精簡策略」,再以其為條件指導行動,分層聯訓策略生成與動作執行,提升長軌跡任務的探索與
credit assignment（ALFWorld 93.1%／WebShop 84.2%／SciWorld 63.5%）。

- **與本專案的關係**：解同一類問題（長程規劃可靠度,即 M3/M5 弱區）;「先抽象策略再執行」
  與 Workflow 管線、「自我評判」與 validator＋repair loop 在設計原則上相似。
- **為何不採用**：StraTA 是訓練期強化學習方法,需微調模型（獎勵訊號＋訓練基礎設施＋大量 GPU）;
  本專案前提為凍結的本地 gemma4:26b（DEC-018）,26B 微調超出範圍。
- **由此定位本專案**：模型端解法（訓練期）存在但成本不可行,故採 harness 層推理期補強
  （約束解碼、驗證重試、確認閘）——這正是 harness 作為「LLM 與外部世界間可靠性基礎設施」的價值。

---

## 附：圖表清單

| 編號 | 圖名 | 類型 | 資料/出處 |
|---|---|---|---|
| 圖 1 | Harness Runtime 六元件架構 | 分層方塊圖 | detailed-design/03-architecture.md |
| 圖 2 | Workflow 執行管線 | 流程圖 | detailed-design/08-assistant-engine.md |
| 圖 3 | 技能生成子流程 | 流程圖 | detailed-design/08-assistant-engine.md |
| 圖 4 | 模型路由 + 送外部模型前的隱私檢查 | 決策樹 | detailed-design/08-assistant-engine.md |
| 圖 5 | 對話記憶資料流 | 資料流/序列圖 | detailed-design/08-assistant-engine.md §8.14 |
