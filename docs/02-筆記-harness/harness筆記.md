# Harness 筆記：架構、研究歷程、答辯

> 2026-07-16 由三份筆記合併去重而成（harness架構-學習筆記、harness架構-六元件對照、harness架構-研究日誌），每個事實只出現一次：概念在第一部、實驗數據在第二部、答辯 Q&A 在第三部。正式報告見 [harness架構報告](../99-報告/harness架構報告.md) 與 [測試系統報告](../99-報告/測試系統報告.md)。
> 工程事實來源：cloud_drive `doc/detailed-design.md` §9.7／§11、`doc/decisions.md` DEC-016～033。

---

# 第一部：架構與心法

## 0. 核心心法：機制優先於模型自覺（mechanism over model self-awareness）

- **是什麼**：凡是能用「程式／規則／文法」保證的，就不要靠「模型記得要做對」。
- **為什麼**：本專案用的是本地小模型 gemma4:26b，不夠聰明、不穩定；與其求它自律，不如把保證外包給機制。
- **關鍵洞察**：把要求從「請模型遵守」升級成「使其不可違反」。
- **一句話記住**：模型的能力預算很稀缺，只花在真正需要智慧的地方（規劃判斷）；其餘全用機制兜底。

這條心法解釋了後面每一個設計決策——遇到問題先問：「這能用機制解嗎？能就別靠模型。」

## 1. 六元件框架與九實作模組

agent harness 文獻（survey）將 harness 定義為六元組 `H = (E, T, C, S, L, V)`：

| 代號 | 名稱 | 白話 | 對應九模組（主要檔案） |
|---|---|---|---|
| **E** | Execution Loop | 主迴圈：送訊息→解析→執行→回填；含模型介面 | 01 while loop、04 sub-agents、workflow 執行器（`service.py`、`workflow.py`、`subagent.py`、`llm/router.py`） |
| **T** | Tool/Skill Registry | 有哪些技能可用（內建／自建／現場生成） | 03 skills & tools、05 built-in skills（`skills/registry.py`、`manifest.py`、`authoring.py`、`builtin/`） |
| **C** | Context Manager | 塞什麼進 prompt（裁切、技能清單、對話歷史） | 02 context management、07 system prompt assembly（`context.py`、`planner.py`、`subagent.py`） |
| **S** | State Store | 存什麼（session／訊息／技能），依 user 隔離 | 06 session persistence（`repository.py`＋migrations） |
| **L** | Lifecycle Hooks | 治理層：權限、確認、沙箱、稽核、隱私閘 | 08 lifecycle hooks、09 permissions & safety（`hooks.py`、`permissions.py`、`codeguard.py`、`sandbox.py`） |
| **V** | Evaluation Interface | 離線評測工具（不在請求路徑上） | §11 驗證/評分 harness（`backend/eval/`） |

**為什麼六元件要拆成九個模組**（計數：E→3、T→2、C→2、S→1、L→2，V 在九模組之外）：

1. **粒度服務於可測試性**：九模組是程式碼層的切法，讓 test_loop／test_context／test_planner／test_sandbox／test_hooks 各自獨立、LLM 一律 mock。
2. **九模組不在同一抽象層級**：有主元件（01 while loop）、子元件（05 built-in skills 是 T 的技能來源之一）、橫切關注點（09 permissions & safety 貫穿所有攔截點）。
3. **一句話記住**：**程式碼九層（實作模組），報告六層（概念架構）**——同一系統的兩種視角。

對外（報告、簡報、答辯）採二層說法：**Workflow Pipeline** 描述「要做什麼」（DEC-021）、**Agent Harness Runtime** 描述「怎麼可靠地跑」（對齊六元件）。

### 答辯注意事項（呈現時的三個要點）

1. **V 不在請求路徑上**：eval harness 是離線的開發者工具，從外部打 `/assistant/chat` 評測系統；畫架構圖時 V 放 runtime 旁邊（箭頭指向系統），不可疊進 runtime 層疊——這也符合 survey 稱之為 evaluation *interface* 的原意。
2. **sub-agent 的現況要如實說明**：`subagent.py` 目前唯一實例是 CodegenSubAgent，不是通用多代理編排。建議說法：「主迴圈可派生 bounded sub-agent，目前實例化為 codegen 子代理」。
3. **L 的合併邏輯**：嚴格說 `sandbox.py`／`codeguard.py` 是執行基礎設施而非 hook；正確說法是「Lifecycle Hooks 是治理層，permissions／codeguard／sandbox 是它在各攔截點強制執行的機制」。

## 2. Workflow 執行管線（一句話怎麼變成動作）

- **是什麼**：需求解析 → 規劃（結構化輸出）→ 技能檢查 → 權限／安全 → 顯示計畫 → 確認？→ 執行 → 記錄。
- **為什麼要「確認」這關**：寫入／破壞性操作執行前必須使用者核可；唯讀的才自動跑（fast-path）。
- **關鍵洞察**：規劃 ≠ 執行。模型先產出「計畫（步驟清單）」，系統檢查、使用者確認後才動手。執行失敗時的回覆由程式從 StepResult 組合，不沿用模型規劃時寫的話（honest reporting，DEC-029）。
- **一句話記住**：模型只負責「想計畫」，真正動手前有權限閘＋使用者確認把關。

## 3. 模型路由＋隱私閘（本專案賣點，勿因架構收斂而省略）

survey 六元組把 LLM 當 harness 之外的黑盒，但本專案的模型策略是明確賣點，歸入 E 的 model interface 子元件、受 L 的隱私閘治理：

- **現行為**（2026-06-25 起）：使用者每則訊息自選模型來源——本機 Ollama（gemma4:26b）或任一筆自建的具名外部連線（OpenAI 相容端點／Codex 訂閱，per-user 加密憑證）。選定即該次唯一執行器，無自動 fallback；失敗回可區分的明確錯誤（連不到／憑證被拒／額度耗盡）且快速失敗（本機 connect 逾時 5s）。
- **隱私閘永遠在**（DEC-023 第 2 條不變）：即使使用者手動選外部，敏感且無法去識別化的內容仍拒送。
- **歷史演進**：原「本地預設、連續失敗自動升級外部」已被手動選擇取代；自動升級僅保留於 `target=None` 相容路徑，仍有 eval `model-escalation` 案例覆蓋。
- **一句話記住**：本地優先是隱私主軸；外送前一定過隱私閘。

## 4. 技能生成與治理（自我撰寫技能）

- **是什麼**：缺技能時模型現場生成一個，但受嚴格治理：codegen → codeguard（靜態檢查）→ sandbox（隔離執行）→ 核可 → 執行 → 安裝。
- **一句話記住**：允許模型寫程式，但「核可＋沙箱＋稽核」層層把關，不讓生成碼裸奔。

## 5. 報告用架構圖（V 在旁側）

```
使用者自然語言需求
        ↓
┌─ Workflow Pipeline ────────────────────────────┐
│ 需求解析 → 規劃 → 技能檢查 → 權限/安全 → 確認 → 執行 → 記錄 │
└────────────────────────────────────────────────┘
        ↓
┌─ Agent Harness Runtime（六元件） ───────────────┐      ┌─ V Evaluation Interface ─┐
│ E Execution Loop                                │      │  （離線開發者工具）        │
│   while loop / workflow executor /              │ ◄─── │  deterministic assertion  │
│   bounded sub-agent(codegen) /                  │ 評測  │  API / browser / exec     │
│   model interface（使用者自選：本機/具名連線）    │ 請求  │  LLM judge / baseline     │
│ T Tool / Skill Registry                         │      └───────────────────────────┘
│   built-in / 自建 / 現場生成技能 + manifest       │
│ C Context Manager                               │
│   裁切 / prompt 組裝 / 技能清單注入               │
│ S State Store                                   │
│   sessions / messages / skills / workflows      │
│ L Lifecycle Hooks（治理層）                      │
│   permission gate / approval / codeguard /      │
│   sandbox / audit —— 含外送隱私閘                │
└─────────────────────────────────────────────────┘
        ↓
CloudDrive Service Layer（DEC-017：不直接碰 DB/FS）
        ↓
PostgreSQL + Storage Provider
```

## 6. 相關文獻（答辯用）

- **StraTA（arXiv 2605.06642）**：Agentic RL 框架——先從任務初始狀態生成「精簡策略」，再以其為條件指導行動，分層聯訓策略生成與動作執行，提升長軌跡任務的探索與 credit assignment（ALFWorld 93.1%／WebShop 84.2%／SciWorld 63.5%）。
  - **與本專案的關係**：解同一類問題（長程規劃可靠度，即 M3/M5 弱區）；「先抽象策略再執行」與 Workflow 管線、「自我評判」與 validator＋repair loop 在設計哲學上同構。
  - **為何不採用**：StraTA 是訓練期 RL 方法，需微調模型（獎勵訊號＋訓練基礎設施＋大量 GPU）；本專案前提為凍結的本地 gemma4:26b（DEC-018），26B 微調超出範圍。
  - **答辯用法**：被問「為何不從模型端解決規劃可靠度」時引用——訓練期方法存在但成本不可行，故採 harness 層推理期補強（約束解碼、驗證重試、確認閘），這正是 harness 作為「LLM 與外部世界間可靠性基礎設施」的價值主張。

## 7. 報告建議措辭

> CloudDrive 的 In-App AI Assistant 採二層設計：Workflow Pipeline 描述「要做什麼」，Agent Harness Runtime 描述「怎麼可靠地跑」。Runtime 對齊 agent harness 文獻的六核心元件（Execution Loop、Tool Registry、Context Manager、State Store、Lifecycle Hooks、Evaluation Interface）；工程實作上，六元件再細分為九個模組以利獨立開發與測試。九模組中的 sub-agents、built-in skills、system prompt assembly、permissions & safety 分別是六元件下的子模組或橫切治理機制，而非獨立最高層。模型來源由使用者逐訊息自選（本機優先、外送必經隱私閘），評測則由獨立的離線 eval harness 以確定性斷言＋LLM judge 持續把關。

---

# 第二部：研究歷程與數據

- **研究對象**：把本地小模型 gemma4:26b 包裝成可靠代理的 Agent Harness。
- **核心研究問題**：一個不夠聰明、不穩定的本地小模型，能不能用工程機制馴服到「可用的可靠度」？
- **一句話結論**：能——但要分清「機制救得了的」與「只能靠模型智慧的」；前者用機制根治，後者用資料量測、以 prompt 有限度改善。

## 研究方法論

> **量化 → 定位根因 → 修正 → 再驗證**

1. **真模型量測（temp_sweep）**：代表案例 × N 次 × 多個 temperature，量 pass rate／跳針率／延遲；機率性系統不看單次。
2. **A/B 對照**：疑似退化時起 baseline 版對打，拿基準再定位，不猜。
3. **單元測試不足以驗 LLM**：單元裡 LLM 是 mock 的，全部通過只證明管線接得對，不證明真模型做得對。
4. **全量 log 落檔再過濾**：診斷先完整記錄，不在管線裡先 tail／grep。

## 時間線總覽

| 期間 | 階段 | 關鍵產出 |
|---|---|---|
| 6/17–6/20 | 測試基礎設施（eval harness E1–E6） | 400 案例套件、三執行模式、LLM 考官、分數導向 |
| 7/05 | 執行策略決策 | DEC-030 串行執行（DAG 並行延後） |
| 7/06 | 防跳針循環一、二 | DEC-031（結構化解碼）、DEC-032（enum）、temp_sweep＋E7 量測 |
| 7/07 | 防跳針循環三＋記憶起步 | codegen 保護、DEC-033（think:false A/B）、記憶 v1、多輪 eval 起步 |
| 7/08 | 記憶收尾＋eval 嚴謹化 | confirm 修復、多輪 eval（recall 0/5） |
| 7/08–09 | recall 診斷閉環＋temperature 再量測 | 依假設誤修 → 正確診斷雙缺陷 → 5/5；temperature 假象釐清 |

## 6/17–6/20：建立測試基礎設施（eval harness）

先有「尺」才能量：生成 M2–M5 案例套件（每級 100、共 400）；三種執行模式（api／browser／exec）；LLM judge 可換 provider；分數導向範式（考官給分數＋優缺點，不只 pass/fail）；`--tag`／`--verbose` CLI。**沒有可重複的量測，就沒有資料驅動的迭代**——這套 harness 是後面所有「量化」的前提。

## 7/05：DEC-030 執行維持串行

workflow 執行維持串行，DAG 並行延後——所有技能共用同一 AsyncSession，並行需先解此前置。把「暫不做、為什麼」寫成正式決策，不假裝完整。

## 7/06：防跳針循環一、二（DEC-031／DEC-032）＋E7 量測

- **現象**：某些請求（如「查剩餘容量」）讓模型陷入跳針（repetition loop，重複生成卡到 timeout）。
- **定位**：結構化請求把 temperature 釘 0，gemma4 在 greedy decoding 下掉入決定性的 repetition loop，同 prompt 100% 重現。
- **循環一 DEC-031**：`num_predict` 生成上限（跳針變有界失敗）＋結構化請求用低而非零的 temperature（微量隨機打破迴圈，格式仍由 grammar 保證）。驗證：原本必逾時失敗的整合測試 40/40 通過。
- **循環二 DEC-032**：模型會規劃出不存在的技能名。把計畫 schema 的技能名做成 enum（依當下 registry 動態組）→ 幻覺技能在取樣層即不可生成。副發現：`validate_plan` 漏查 `depends_on` 非法相依，一併補上。這是「機制優先於模型自覺」的教科書例——不是求模型別亂編，是讓亂編文法上不可能。
- **E7 首次量測**（4 案例 × 5 × 4 個 temperature）：首次量化確認**破壞性規劃是模型重災區**（0–40% 可靠度，跳針與幻覺技能並存）。過程中修了兩個量測工具 bug（token 過期假 401、後端提早退出未偵測）——量測工具本身也要驗。

## 7/07：防跳針循環三（DEC-033）＋codegen 保護

**(a) codegen 結構化保護**：codegen 是最後一個未受 grammar 保護的結構化呼叫點，補上 `response_format`。真模型 A/B 發現：codegen 被拖進結構化低溫 0.2 後產碼品質 0/4（JSON 信封完好、碼壞）；per-call temperature 覆寫 0.8 恢復。**原則：temperature 依任務類型而非是否結構化。**

**(b) 循環三 DEC-033 think:false**：DEC-031/032 後殘留跳針的唯一棲息地是模型「思考」段——grammar 只約束最後答案。A/B（60 樣本、只改思考開／關一個變數）：

| | thinking 開 | think:false |
|---|---|---|
| 通過率 | 60% | **100%** |
| 跳針 | 3/10 | **0/10** |
| 平均延遲 | 92.4s | **8.6s（快約 10 倍）** |

結論：思考對這顆小模型的規劃品質無可量測貢獻，卻是跳針與延遲的全部來源 → planner 預設 `think:false`，codegen 不連動。

**裸露出的天花板**：雜訊清掉後，困難集 M3/M5（多查詢＋一個寫入）仍約 **47%**（think:false，30 樣本；分級數字樣本小、不逐級宣稱）；失敗典型是**合法但語意錯——漏掉使用者要的寫入步驟（規劃成唯讀）**，歸類為模型能力限制。

## 7/08：對話記憶 v1＋多輪 eval

- **根因（查證非推測）**：對話有存進 DB、但規劃器只收當前訊息——歷史從沒回讀，修法是接線。
- **關鍵 A/B（工具結果怎麼承載）**：`tool` 角色訊息 0/4（幻覺假檔名，chat template 不消化孤立 tool 訊息）vs assistant 文字 4/4 → 以 assistant 文字承載，順帶零 migration。
- **工程 bug（真實使用抓到）**：按下確認、執行成功後追問，助理答「還在等確認」。根因：記憶只在 `/chat` 寫歷史，`/confirm` 從沒寫回 → 執行後把結果摘要寫回 session。
- **多輪 eval**：單輪 harness 擴成多輪（播種狀態＋串 session）。結果：純對話指涉 5/5、工具結果指涉（代理）5/5、嚴謹回想 **0/5**——初判「摘要截斷」，是憑讀碼的假設、未診斷（後被推翻）。

## 7/08–7/09：recall 診斷閉環（方法論最完整的一次示範）

- **現象**：嚴謹回想案例 0/5；代理（改名）案例卻 5/5。
- **錯誤的第一步（如實記錄）**：憑閱讀 `summarise_results` 推論「輸出含 UUID → 超 200 字截斷 → 丟名」，並預設列檔已成功；依此假設改了摘要格式 → 重跑仍 0/5。
- **正確診斷**：verbose 印出實際回覆與記憶內容 → 列檔實際失敗、尚無輸出可供截斷。真因是兩個疊加缺陷：①主因——模型列根目錄傳 `parent_id="root"`（而非 null），`_optional_uuid` 解析崩潰，list_items 失敗、記憶中無檔名；②次因——即使列檔成功，摘要序列化原始 dict（UUID 佔滿 200 字預算）也會截斷丟名（先前誤修的那項，被主因遮蔽）。
- **修復並驗證**：兩者皆修 → 端到端真模型驗證 → recall 0/5 → 5/5、三案例綜合 15/15。
- **方法論收穫**：**憑讀碼向前推論、預設上游成功，是典型認知陷阱；印出 runtime 的真實診斷才是可信判準。**

**同期：temperature 再量測（釐清假象）**——乾淨重跑（think:false，4 個 temperature × 10）：四個 temperature 全 100%、0 跳針；對照 thinking 開（4 個 temperature × 5）85–90% 且有跳針。先前看似的「temperature 效應」其實是跳針傾向（低 temperature 接近 greedy decoding，更易進入 repetition loop）；`structured_temperature=0.2` 的定位是「打破 repetition loop 的最小非零 temperature」以求確定性，並非因高溫會失敗。

## 未解問題與未來方向

- **斷線取消**（工程，可解）：放棄的請求仍佔 Ollama 單併發槽（真實使用發現）。
- **planner 寫入意圖**（模型前沿）：用多輪 eval 迭代 prompt 工程，目標把 47% 往上推。
- **記憶 v2**：摘要壓縮 → 語意檢索（復用 pgvector）→ 跨 session。

## 研究反思

> 最大的認知風險不是「不會做」，而是「以為驗證過了」。兩週內至少三次「All pass」但真模型實測或真實使用發現有問題。對策不是寫更多同源測試（它們有一樣的盲點），而是引入不同源的視角：真模型 A/B、機械式窮舉資料表面。

---

# 第三部：答辯 Q&A 與一頁速記

## 答辯用專案敘事

> 功能面已完工,專案的價值在「一個本地小模型如何被工程機制馴服到可用的可靠度」這條完整證據鏈——三個「量化 → 修正 → 驗證」循環（DEC-031/032/033）是最強的工程敘事。流程層的隱形資產同樣值得講：文件先行＋DEC 決策鏈＋真模型驗證紀律＋秒級 sweep 的評測基礎設施。

## Q&A

**Q：「規劃成唯讀」是模型自己把改名做完了嗎？**
> 不是，剛好相反——它把改名整個漏掉、什麼都沒改。計畫裡只剩查詢步驟，「自動執行」跑的是那些無害的查詢。病灶是「該動手的默默漏做」。

**Q：M3/M5 測的是什麼？**
> 給模型一句「先查幾樣、然後做一個寫入動作」的複合請求，看它能不能組出含那個寫入步驟的完整計畫；跑一百種、算通過率。失敗典型＝漏掉寫入那步。

**Q：為什麼「漏掉寫入」是模型限制、不是工程缺口？**
> 它產出的是合法但語意錯的計畫（文法對、技能名真，只是漏了 rename）。grammar／enum／think 這些機制只能約束輸出空間或解碼，不能替模型做「選對技能」的判斷；且 think 開關對它沒影響（證明不是解碼殘影）。選擇是推理能力，無法用機制保證——只能靠 prompt 工程或更強的模型推進（數據見第二部 7/07 節）。

**Q：think:false A/B 是什麼？**
> 只改「思考開／關」一個變數、其他全同的對照實驗（60 樣本）。關掉後跳針歸零、延遲快約 10 倍、通過率 60%→100%（表見第二部）。

**Q：eval 怎麼「自動判對錯」？**
> 不是叫模型打分，而是把模型回傳的結構化 JSON 做 `==`／`in` 比對（例：`status == "pending_approval"`、`"rename_item" in steps`）。同樣回應必得同樣結果。「規劃成唯讀」就是這樣量到的：期望 `pending_approval`、實得 `auto_executed`。

**Q：代理測試 vs 嚴謹測試？**
> 代理（proxy）＝間接驗證（改名案例只驗「計畫含 rename＋新名」）；嚴謹＝直接驗證（回想案例要求報出全部三名）。教訓：寬鬆的代理測試會遮蔽缺陷——改名 5/5 遮蔽了列檔失敗，是嚴謹回想把它暴露（完整閉環見第二部 7/08–09 節）。

**兩把判斷尺（最該內化）**
> ①設計時：機制 vs 模型自覺——能用文法／程式保證的，不靠模型自律。②排序時:模型限制 vs 工程缺口——工程缺口做一次永久解決（值得做）；模型限制無機制能保證，只能以量測數據為據、靠 prompt 有限度改善（有限投入）。

## 一頁速記

- **核心心法**：機制優先於模型自覺——凡能以文法／程式保證者，不依賴模型自律。
- **六元件**：E 執行迴圈／T 技能登錄／C 脈絡管理／S 狀態儲存／L 生命週期治理／V 評測介面（在旁側）；程式碼九層、報告六層。
- **執行管線**：規劃 ≠ 執行；寫入操作執行前有權限閘與使用者確認;失敗回覆由程式從 StepResult 組合（honest reporting）。
- **可靠性三循環**：DEC-031 防跳針 → DEC-032 防幻覺技能 → DEC-033 關 thinking（快約 10 倍）；方法為「量化 → 定位根因 → 修正 → 再驗證」。
- **對話記憶**：存了沒回讀 → 接回規劃器；工具結果以 assistant 文字承載（tool 角色 gemma4 不解讀）；為滑動視窗（約 6 輪、摘要 200 字、單 session）。
- **eval 判定**：以 `==`／`in` 機械比對結構化 JSON（非考官打分）。
- **診斷紀律**：憑讀碼即預設上游成功是典型認知陷阱；以實際 runtime 輸出為準。
- **如實界線**：sub-agent 僅 codegen；執行串行、未並行；記憶為滑動視窗；寫入意圖規劃約 47%，為模型能力上限。
