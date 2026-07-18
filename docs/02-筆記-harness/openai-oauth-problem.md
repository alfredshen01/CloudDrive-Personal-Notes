# Codex 訂閱制接入 — 問題分析與結論

> 狀態:分析文件(2026-06-25)。對應已實作功能見 cloud_drive 的 [tasks/external-model.md](https://github.com/billwu101/CloudDrive/blob/main/doc/tasks/external-model.md)、決策 DEC-026。
> 結論:**Codex 訂閱制(路徑 A)目前不適合作為一般使用者的主要管道**,僅保留為技術型 power user 的進階選配。測試階段可用,但上線面向大眾不建議主推。

---

## 1. 背景

專案 AI 助理預設使用本機 Ollama(免費、自架);使用者可為每則訊息自選模型來源——本機或自建的具名外部連線(撰文當時的設計是「本機連續失敗自動升級外部」,其後由手動選擇取代,演進見 [harness筆記](harness筆記.md) 第一部 §3)。外部連線走使用者自己的 GPT-5.5,有兩條路徑:

- **路徑 A:Codex 訂閱**(`provider=codex`, `auth_type=oauth_token`)— 用 ChatGPT 訂閱額度,憑證是 `codex login` 產生的 `auth.json`。對應 `app/external_model/codex_client.py`。
- **路徑 B:OpenAI API key**(`provider=openai`, `auth_type=api_key`)— 按 token 計費,憑證是 `sk-...`。對應 `app/assistant/llm/external.py`。

本文件聚焦**路徑 A 的問題**。

## 2. 核心問題:取得與使用 token 的門檻太高

`codex` 工具在架構裡有**兩個分離的角色**,跑在不同機器:

| 角色 | 動作 | 需求 | 執行位置 |
|---|---|---|---|
| 取得 token | `codex login`(OAuth 開瀏覽器登入 ChatGPT)→ 產生 `auth.json` | 要能開瀏覽器 | 使用者自己的電腦 |
| 使用 token | 後端 `codex exec` 帶 token 呼叫 GPT-5.5 | 主機要裝 `codex` CLI 二進位 | 後端主機(VM / 上線伺服器) |

token(`auth.json`)本身可跨機搬移(僅 OAuth token,未綁機;見 external-model.md §5),所以設計上是:使用者在有瀏覽器的機器登入拿 token → 貼進網頁 → 後端用自己的 CLI 執行。

### 一般使用者要做的事(致命 UX 問題)

1. 安裝 Node.js / Homebrew
2. 安裝 `@openai/codex` CLI
3. 跑 `codex login` 做 OAuth
4. 找到 `~/.codex/auth.json`
5. 複製內容、貼進網站設定頁

這是**工程師等級**的操作流程,非技術使用者幾乎不可能完成。因此路徑 A 實質上只適合技術型 power user。

## 3. 為什麼不能做成「網頁內點按授權」(Sign in with ChatGPT)

理想 UX 是:使用者在網頁按一個鈕 → 跳轉 ChatGPT → 同意授權 → app 自動拿到 token(像 Sign in with Google)。但 2026 現況做不到:

1. **「Sign in with ChatGPT」≠ 借用訂閱額度**:它是身分登入,不等於把訂閱開放給第三方 app 呼叫模型;且到 2026/4 仍只在 Codex 工具內運作。
2. **唯一能扣訂閱額度的 token 就是 Codex OAuth token**(`auth.json`),其 OAuth 是**本機 PKCE 流程**:需在使用者電腦起 `localhost:1455` callback server、且借用 OpenAI 自己的 Codex client_id(`app_EMoamEEZ73f0CkXaXp7hrann`)。
3. **與 hosted SaaS 架構衝突**:`localhost:1455` callback 必須與「開瀏覽器的那台機器」同機,雲端後端無法承接;且第三方網站借用 Codex client_id 走此流程屬 **ToS 灰色地帶**。

→ 結論:「網頁內點按授權訂閱」**只有在「本機自架版」(後端就跑在使用者電腦,瀏覽器與後端同在 localhost)才可能實作**,多人雲端 SaaS 不可行。

## 3.5 詳細流程:授權回呼為何在雲端「接不住」

### 關鍵前提:`localhost` 指「講這句話的那台機器自己」

`localhost`(= `127.0.0.1`)不是公開地址,而是相對代名詞,意思永遠是「正在執行的這台機器自己」,**不能跨機器指到別台**:

- 在使用者 Mac 上講 `localhost` → 指那台 Mac
- 在伺服器上講 `localhost` → 指那台伺服器

### 同機(本機 `codex login`)— 接得住 ✅

```
使用者的 Mac（同一台機器內）
┌─────────────────────────────────────────────┐
│ ① codex 在 localhost:1455 開一個小伺服器       │
│ ② 瀏覽器（也在這台 Mac）開 ChatGPT 登入頁      │
│        ↓ 使用者按「同意授權」                   │
│ ③ OpenAI 叫瀏覽器去 localhost:1455/callback     │
│    ?code=授權碼                                │
│        ↓                                       │
│ ④ 瀏覽器連 localhost:1455 → 大家都在這台 Mac，  │
│    剛好連到 ① 的小伺服器 → 接住授權碼 ✅          │
└─────────────────────────────────────────────┘
```

重點在 ④:瀏覽器與 codex 程式同在一台 Mac,雙方的 `localhost` 是同一處 → 對得上。

### 雲端(瀏覽器在使用者端、後端在伺服器)— 接不住 ❌

```
使用者電腦                          你的伺服器（另一台）
┌──────────────────────┐          ┌──────────────────────┐
│ ② 瀏覽器在這裡         │          │ 後端在這裡(想接授權碼) │
│ ③ OpenAI 叫瀏覽器去   │          │                       │
│   localhost:1455 ...  │          │                       │
│        ↓              │          │                       │
│ ④ 瀏覽器連 localhost  │          │                       │
│   = 使用者「自己這台」 │  ✗ 連不到 ─→（伺服器在別台）    │
│   但這台沒人在 1455監聽│          │                       │
│   → 連線失敗 ❌         │          │                       │
└──────────────────────┘          └──────────────────────┘
```

- OpenAI 叫瀏覽器去 `localhost:1455`;瀏覽器在使用者電腦,故 `localhost` = 使用者自己那台。
- 但使用者那台沒有程式在 1455 監聽(想接的後端在伺服器、另一台)→ 撲空。
- 也無法叫瀏覽器改連伺服器的 localhost —— 對瀏覽器而言 `localhost` 永遠是它自己那台。

### 為何不能「把回呼地址改成伺服器公開網址」

1. **redirect_uri 寫死且需事先註冊**:OAuth 規定回呼地址必須是 app 向 OpenAI 註冊好的;Codex 的 client_id 只註冊 `http://localhost:1455/auth/callback`,只允許這一個。
2. **沒有自己的 client_id**:要改成 `https://你的網站/callback` 需用自己註冊的 client_id,但 OpenAI 未開放第三方申請「借用 ChatGPT 訂閱」的 client_id,只能冒用 Codex 那個(→ ToS 風險,且它只認 localhost:1455)。

### 「本機收到後再送給雲端」可行嗎?——可行,這正是現行做法

問題不在「誰拿 token 去問 GPT」(伺服器代問完全 OK),而在「token 怎麼產生」。把「本機收 token → 交給雲端」這動作獨立出來看:

| 方式 | 可行? | 代價 |
|---|---|---|
| 本機收到 → **手動**貼給雲端 | ✅ 現行做法 | 使用者複製貼上一次 |
| 本機收到 → **自動**傳給雲端 | ✅ 技術可行 | 需在使用者電腦**安裝並執行一支本機程式**(等於回到裝 CLI 的門檻) |
| 純網頁內按一下自動完成 | ❌ | 瀏覽器安全限制(不准網頁開本機伺服器/讀本機檔)+ localhost 同機限制 |

原因補充:「在本機開 localhost:1455 接授權」一定要有一支**程式跑在使用者電腦上**(`codex` CLI 就是這支);瀏覽器裡的 JavaScript 基於安全,不能開本機伺服器、也不能讀本機檔案,所以網頁無法自動完成。業界工具(如 OpenClaw)能做到「點按授權」,是因為它們**本身就是跑在使用者電腦上的本機程式**,沒有雲端那個跨機器問題。

**小結**:「本機收到再送雲端」方向正確,且就是現行方案;「送」這步要嘛使用者手動貼(現在),要嘛使用者裝本機程式自動傳(不更省事)。真正做不到的只有「完全在網頁、什麼都不用裝」。

## 4. 上線方案比較

| 方案 | 使用者要做什麼 | 誰付費 | 方便度 | 適合對象 |
|---|---|---|---|---|
| A. App 統一提供模型 | 不用做任何事 | App 方(一把 key,設用量上限/付費方案) | 高 | 一般大眾(ChatGPT、Notion AI 模式) |
| B. 自帶 OpenAI API key | 申請並貼上 `sk-...` | 使用者(按量) | 中 | 半技術/企業使用者 |
| C. 自帶 Codex 訂閱(現況路徑 A) | 裝 CLI + `codex login` + 貼 token | 使用者(吃訂閱) | 低 | 技術型 power user |

## 5. 結論與建議

1. **架構本身合理**:預設免費本機 Ollama,外部 GPT-5.5 僅在本機失敗 + 使用者自備憑證時 opt-in 啟用。「自帶憑證不方便」在 opt-in 進階選項的定位下可接受。
2. **路徑 A(Codex 訂閱)定位為進階選配**,不主推給一般使用者;保留給技術型使用者與「本機自架版」。
3. **面向大眾上線時**,真正方便的是 **方案 A(App 統一提供模型 + 用量控制/付費方案)**;路徑 B 作為「自帶憑證」的次選。
4. **「網頁內點按授權」** 僅在本機自架版列為未來選配,需走完整 proposal → design 流程,且須評估 ToS。

## 6. 待確認 / 後續

- [ ] 產品方向:上線主推方案 A 還是維持 B/C 自帶憑證?(影響是否要建「App 統一 key + 用量計量/限額」模組)
- [ ] 測試替代方案:用 Google Cloud(Gemini API,OpenAI 相容端點 + API key)走路徑 B 來驗證外部升級接線,不需 OpenAI/Codex(見另行紀錄)。
- [ ] 若採方案 A,需設計用量計量、限額、成本控制與濫用防護。
