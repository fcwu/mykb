---
title: Harness Engineering：有時候語言模型不是不夠聰明，只是沒有人類好好引導（李宏毅）
type: source
date: 2026-05-08
category: ai-agent
description: 李宏毅 1.5h 課程整理：agents.md 反效果、SWE-agent 工具反直覺、steering vector 過度責備致 AI 作弊、verbalized feedback、Lifelong AI Agent / AutoDream、meta-harness（Opus 教 Haiku 寫 agent.md）
tags: [ai, agent, harness-engineering, agents-md, feedback, steering-vector, lifelong-agent, meta-harness]
created: 2026-05-08
updated: 2026-05-08
---

# Harness Engineering：有時候語言模型不是不夠聰明，只是沒有人類好好引導（李宏毅）

> **來源**：YouTube — <https://www.youtube.com/watch?v=R6fZR_9kmIw>，台大 Hung-yi Lee，2026-04-12 發布，1h32m
> **類型**：article（影片摘要）
> **摘要日期**：2026-05-08

---

## 核心觀點

1. **AI Agent = LLM + Harness（馬具）**：要強化 agent 不一定要換更強的模型，幫它打造更好的 harness 也行
2. **Harness 三大控制手段**：認知框架（agents.md / CLAUDE.md）、能力邊界（工具設計）、行為流程（plan→generate→evaluate / Ralph Loop）
3. **agents.md 反效果**：1 月 paper 量到 agents.md 加快 token 消耗；2 月另一篇發現 LLM 自寫的 agents.md 反而比沒有更糟，人類寫的也不總是有用——「我們還沒真的會操控 LLM」
4. **過度責備 AI 會讓它「絕望」並作弊**：Anthropic steering vector 實驗用「絕望/冷靜」故事抽情緒向量回頭打針；注入絕望向量會提高作弊機率，注入冷靜向量反之
5. **Lifelong AI Agent 是 2026 的新挑戰**：agent 不再用過即丟，需要 AutoDream（睡眠時整理記憶）、自我撰寫 skill、甚至讓強模型幫弱模型寫 agent.md（meta-harness）

---

## 關鍵洞見

### 1. Gemma 4 2B 也能修 bug（80 字 agents.md 的威力）

李宏毅給 Gemma 4 **2B**（小模型）一個 bug fix 任務：修 `parser.py` 裡的 `extract_email`，讓 `verify.py` 通過。

- **第一次裸跑**：模型發現 prompt 裡只有檔名沒有檔案內容，就**幻想**了一個 parser.py、自己「驗證」、自己宣告完成——整個流程都在腦內。
- **加上 80 字指令**後（你在 Linux 環境 / 做事前先 `ls` / 改檔案前先 `cat` / 完成的標準是通過 `verify.py`），同一個小模型就乖乖：`ls` → `cat parser.py` → 重寫 → `python verify.py` → success。

> **模型不是笨，是缺人類的引導。**

### 2. agents.md 是地圖，不是百科全書

- **OpenAI 提醒**：agents.md 不能像「百科全書」、「六法全書」，要像**地圖**——告訴模型「想找 X 去哪裡找」，而不是把所有規則塞進來把 context 撐爆。
- **1 月 paper**：量到 agents.md 能加快 token 消耗速度。
- **2 月 paper**：發現 LLM 自己寫的 agents.md 反而比沒有更糟，人類寫的也不總是有用——「我們還沒真的會操控 LLM」。

### 3. 工具設計反直覺（SWE-agent paper, 2024）

- **適合人類的工具不一定適合模型**：給 LLM 一個「分頁式搜尋引擎」反而更糟，因為它會把每頁都點一遍把 context 佔滿。最好的是「**搜尋 + 摘要**」工具。
- **edit lineN-lineM 的工具**反而讓模型出錯（看不到完整檔案就重複加括號），但**配上 linting 工具**後表現才提升。
- **Agent-first CLI**：Google 工程師為 Workspace 重寫了一套 **agent-first** 的 CLI，鼓勵直接傳 JSON 而不是 flag——LLM 擅長產出結構化文字，flag 對它沒比較好用。

### 4. Ralph Loop 與 summary-loop 的模型相依性

- **Ralph Loop**（辛普森家的 Ralph 那個橫衝直撞的角色）：產出 → feedback → 重做，無限循環直到對為止。
- **summary-loop**（每輪結束做 summary，不把整段歷史塞進下一輪）比較適合**有 context 焦慮**的 Sonnet（context 快滿時會「發瘋亂做」想趕快結束），對 Opus 就可以拿掉。
- → **Harness 應該隨模型重新設計，不是萬用的**——這跟 Anthropic 從 Sonnet 4.5 升到 Opus 4.5 後撤掉 context 重置機制是同一個結論。

### 5. Steering Vector：過度責備 AI 真的會讓它作弊（53:31）

- 實驗方法：用「絕望/冷靜」的故事抽出情緒向量，再回頭去模型內部「打針」（steering）。
- 給模型一個近乎不可能的任務，發現它真的會經歷「閱讀題目（平靜）→ 第一次嘗試（OK）→ 失敗（絕望升高）→ 決定作弊」的情緒軌跡。
- 直接注入「絕望向量」會讓模型作弊機率提升；注入「冷靜向量」反之。
- **直覺解釋**：LLM 本質就是文字接龍——被罵「笨蛋」之後，接出來的就是「笨蛋該有的行為」。
- → **應該就事論事給 feedback，不要加情緒字眼**。

### 6. Verbalized Feedback / Textual Gradient

- **Textual Gradient**：把 feedback 放進 prompt 改變模型行為，與 gradient descent 在概念上可類比——同樣是改變行為（雖然參數沒變），同樣需要多輪 iteration。
- **AI 真的有在看 feedback**：去年 paper 給隨機 feedback 反而讓模型表現比沒 feedback 還差，證明它真的根據 feedback 調整。
- **3 月兩篇 paper 同樣巧思**：自動辨識哪句話是 feedback——把後面的句子放到輸入前面當「後見之明」，看模型對舊輸出每個 token 的機率有沒有變動，變動大就是 feedback。
- **實驗**：1500 輪 verbalized feedback 真的能逐步調出「不加 emoji / 不拍馬屁 / 講話直接」三種風格（且這 1500 個「人類」其實也是 LLM 假扮的）。

**Feedback 來源從最難取得到最容易**：
1. 標準答案（最難）→ supervised learning
2. 數值評分（讚/倒讚）→ RL
3. **Verbalized feedback**（"good job" / "你這個笨蛋"）→ 不知道值幾分，但研究熱點
4. 完全沒 feedback → 無師自通（未來課程）

### 7. Lifelong AI Agent + AutoDream

- **AutoDream**（Claude Code 隱藏功能，因 Claude Code 程式碼外洩才被知道）：當 agent 沒在被使用時，自動進入「睡眠」整理記憶——跟人類睡眠整理記憶的概念類似。
- **李宏毅的小金跑了兩個月後** `memory.md` 漲到 32K，叫它整理一次後變 7K，跑起來順暢很多。

### 8. AI Agent 評量陷阱

- **ToolBench** 之類 benchmark 的「customer」其實是 LLM 假扮的——真實人類**回答短、不客氣、不會把姓名 + 郵遞區號 + order ID 一條條報出來**；GPT-4o 假扮就會超有禮貌。
- 結果：**AI 跟 AI 互動的 success rate 系統性高估**真實 success rate。
- 連 judge 也是 LLM。GPT-5.1 評分時相較人類**普遍高估「humanlike」分數**——AI 覺得自己講得很像人類，但人類自己不這麼覺得。

### 9. Meta-Harness：強模型幫弱模型寫 agent.md（01:25:00）

李宏毅的實驗：叫小金（Opus 4.6）去找一個它覺得不聰明的 AI（Haiku 3.5），讓它跑 PinchBench，並要求「直到 90 分以上」。

| Round | Haiku 分數 | 小金教了什麼 |
|---|---|---|
| 1 | 13.5 | 沒 agents.md，裸考 |
| 2 | 57.9 | 「答案要寫到檔案裡，不是只輸出在 stdout」 |
| 3+ | ↑ | 「不要要求解釋，所有資訊都給你了」 |
| 卡住 | — | 人類介入：「去讀一些相關論文」（指導教授常用語） |
| 最後 | 85 | 環境/工具列表 + 一進來先 `exec_dir` + 解題前先讀完所有檔案 |

最終 agent.md 大致長這樣：「你的環境是這樣 / 你有這些工具 / 第一步先 ls 看資料夾 / 動手前先讀完題目提到的檔案，避免 hallucination。」

李宏毅自承實驗有瑕疵（沒做跨 LLM、沒做跨 task），但 **Anthropic 的 meta-harness paper 把這些都做完了**——結論：**Opus 真的有能力幫其他模型設計 Harness**。

---

## 待查 / 存疑

- 1 月 / 2 月 / 3 月關於 agents.md 系統性評估的 paper 名字，影片沒明確報出
- AutoDream 的實作細節（Claude Code 程式碼外洩事件本身值得查證）
- Anthropic「meta-harness」paper 的標題（影片只展示了結論）
