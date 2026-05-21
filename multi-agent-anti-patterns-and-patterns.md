---
title: Multi-Agent 架構再探 — 三省六部反模式與業界收斂共識
date: 2026-05-19
category: ai-agent
description: 2026 業界對 Multi-Agent 的收斂共識：四大反模式（更多 Agent 迷思／三省六部幻覺／固定 workflow ／生產隱藏成本）、Anthropic 三場景五模式、LangChain 四模式、Cognition 寫入單線程、動態子代理生成。
tags: [ai, agent, multi-agent, claude, anthropic, langchain, cognition, anti-pattern]
---

# Multi-Agent 架構再探：三省六部反模式與業界收斂共識

> **來源**：Blog — [aihao.tw · Multi-Agent 架構再探：三省六部反模式和業界收斂共識](https://blog.aihao.tw/2026/05/19/multi-agent-anti-patterns-and-patterns/)（2026-05-19 發布，延續自 [ihower《Multi-Agent 或 Single-Agent》](https://ihower.tw/blog/12776-multi-agent-or-single-agent) 2025-06）

**一句話**：去年「預設 Single-agent，特定場景才上 Multi-agent」的結論沒過時；2026 反而被更多論文與生產經驗驗證，業界對「特定場景」與「怎麼做」有了清晰共識。

## 文章骨架（一次看完）

1. **反模式（4 個）** → 拆穿「更多 Agent 更好」、「三省六部式公司」、「預設固定 workflow」、「生產隱藏成本」
2. **適用場景** → Anthropic 三場景（Context 隔離 / 並行 / 專業化）
3. **協調模式** → Anthropic 五種、LangChain 四種、Cognition 寫入單線程
4. **Best Practices 6 條**
5. **未來方向** → 動態子代理生成（Dynamic Subagent Spawning）
6. **結論** → Multi-Agent 的價值是並行覆蓋，不是分工

## 反模式一：「更多 Agent 就更好」

### Google Research 2026 年初實驗

[原推](https://x.com/GoogleResearch/status/2016621362480382213)：測了 **180 種配置**，發現 multi-agent 效果完全取決於任務類型。

| 任務類型 | Multi-Agent 對比單一 Agent |
|---|---|
| 可並行的金融任務 | **+81%** |
| 需要順序推理的規劃任務 | **−70%** |

**結論**：架構與任務的匹配度，比 agent 數量重要得多。

### Stanford 2026/4 論文：資訊論硬證明

論文：[Single-Agent LLMs Outperform Multi-Agent Systems on Multi-Hop Reasoning Under Equal Thinking Token Budgets](https://arxiv.org/abs/2604.02460)

- 用資訊理論的 **數據處理不等式（Data Processing Inequality）** 論證：固定 reasoning token 預算下，單一 agent 資訊效率更高
- 跨三個模型家族驗證：**Qwen3、DeepSeek-R1、Gemini 2.5**，結果一致
- **控制 token 用量後，single-agent 持平或優於 multi-agent**
- 含義：很多 multi-agent 聲稱的效能提升，其實只是「花了更多 token」，不是「架構更好」

### Arize AI 創辦人 Aparna Dhinakaran 補充

[原推](https://x.com/aparnadhinak/status/2017013647156138191)：
- Multi-agent 架構產生 **通訊開銷（communication overhead）**
- 在不易並行化的任務上不如單一順序 agent 有效
- **Evals 不只要評估單一 agent，也要評估 multi-agent 架構本身**

## 反模式二：「三省六部幻覺」— 把 Agent 當公司部門

2026 年討論度最高的 anti-pattern 之一。源頭：[@sujingshen 的長文《三省六部幻覺：為什麼「虛擬公司」式多 Agent 架構在工程上不成立》](https://x.com/sujingshen/status/2043898494818410731)（**1600+ 讚 / 2400+ 收藏**）。

### 反模式長相

很多團隊把 Agent 命名為「產品經理」、「架構師」、「測試工程師」，讓它們像公司部門一樣傳遞文檔、協作完成任務。直覺，但有根本性缺陷。

### 三個結構性問題

🔹 **人類分工是因為注意力與專業壁壘有限；LLM 沒這些限制**
- 同一個模型既能寫 PRD 又能寫程式碼
- 貼「測試工程師」標籤不會讓它更專業，**只會讓它拒絕越界**
- 最有價值的推理往往發生在邊界上 — 角色扮演在系統層面封死了這個可能性

🔹 **資訊在流轉中死亡**
- Agent A 傳給 Agent B 的是 **結論不是推理過程**
- 每次傳遞累積誤差，工作流越長，輸出越「局部正確但整體漂移」
- 像傳話遊戲

🔹 **三家頭部廠商都不用這個模式**
- **Anthropic、OpenAI、Google** 自己建 Agent 系統時，沒有一家採用角色扮演式流水線
- 他們用的是 **orchestrator-worker 架構 + 外部狀態文件**

### Dify 工程師實戰血淚

Yeuoly [親身踩坑](https://x.com/Yeuoly1/status/2054922914458464754)：

- 小團隊嘗試給不同 Agent 分配 iOS 上架、UI 設計、工程開發等職責
- 結果：注意力爆炸、Agent 互相干擾
- 心得：「**好像去咸魚找個人會更快更好**」

## 反模式三：預設固定 Workflow 的脆弱性

### @aneeshpappu — Multi-Agent Teams Hold Experts Back

[原推](https://x.com/aneeshpappu/status/2019447577825976332)：
- 大多數 multi-agent 系統使用 **預先指定的工作流程、固定角色、聚合規則**
- 任務複雜到無法提前規劃時 → 系統就崩潰

### OneManCompany 論文（@dair_ai 介紹）

[原推](https://x.com/dair_ai/status/2048909068409147460)：
- 比喻：把 agent 當成 **「可動態招募的人才市場」**（dynamic talent marketplace）
- 而不是 **「固定的組織架構圖」**

## 反模式四：生產環境的隱藏成本

進入 production 後 multi-agent 問題集體浮現。

### 1) Token 成本非線性爆炸

- Anthropic：[multi-agent 通常用 **3–10 倍** token](https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them)
- Anthropic Research 系統：**約 15 倍**
- [Augment Code 分析](https://www.augmentcode.com/guides/multi-agent-cost-compounding)：「為什麼 3 個 Agent 要花 10 倍成本」
  - 原因：**context 複製 × 協調訊息 × 驗證層 × 重試迴圈 = 乘法效應**

### 2) 錯誤放大效應

- 直覺算：每個 agent 90% 準確率，串三個變 **72.9%**
- Google/MIT [agent 系統擴展研究](https://x.com/GoogleResearch/status/2016621362480382213)：
  - 獨立 agent 錯誤放大倍數：**17.2 倍**
  - 加上中央協調：**4.4 倍**

### 3) MAST 失敗分類學

論文：[MAST: Multi-Agent System Failure Taxonomy](https://arxiv.org/pdf/2503.13657)
- 分析 **1600+ 失敗案例**
- 7 個 SOTA 開源 multi-agent 系統的失敗率：**41% – 86.7%**
- 關鍵發現：**很多失敗來自系統設計問題而非 LLM 能力不足** — 改 prompt 不夠，需要結構性重新設計

### 4) Antigravity Lab — 生產級 12 個坑

[原文](https://antigravitylab.net/en/articles/agents/antigravity-multi-agent-production-12-pitfalls-deep-dive)：

- 重試迴圈
- Token 失控
- Context 污染
- 級聯故障
- 可觀察性缺失
- ……（共 12 個 prototype 階段看不到、production 一定遇到的陷阱）

## 業界收斂的核心原則

### 跨廠商共識：從簡單開始，必要時才加複雜度

- **Anthropic**：「我們見過團隊花了好幾個月建構精緻的 multi-agent 架構，結果發現改進單一 agent 的 prompt 就能達到等效結果。」
- **LangChain**：「先從單一 agent 和好的 prompt engineering 開始。**先加工具，再加 agent。**」
- **OpenAI**：*Building Effective Agents* 開頭也是類似建議

## Anthropic 三個適用場景（2026/1）

來源：[Building Multi-Agent Systems](https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them)

### 1️⃣ Context 隔離

- 子任務產生大量但 **無關** 的 context（例：查訂單歷史 2000+ token）
- 直接放進主 agent 會污染推理品質
- 做法：用 subagent 查完，只回傳 **50–100 token 的摘要** 給主 agent
- 保持主 agent 的 context 乾淨

### 2️⃣ 並行化

- 需要同時探索 **多個獨立方向**（如 Research 功能）
- 重點是 **「覆蓋面更廣」** 而非「速度更快」
- 反直覺：**平行 agent 常常比 single-agent 花更久**，但搜尋空間大得多

### 3️⃣ 專業化

- 工具數量 **超過 15–20 個**
- 或不同任務需要 **衝突的 system prompt**（例：客服要有同理心、code review 要嚴格挑錯）
- 拆成專門 agent 提升可靠性

### 最關鍵的設計原則

> **以 context 為中心拆分，不是以問題類型拆分**

- 寫功能的 agent 也應該寫它的測試 — 因為它已經擁有完整的 context
- 只有在 context **真的可以完全隔離** 時才拆開

## Anthropic 五種協調模式（2026/4）

來源：[Multi-Agent Coordination Patterns](https://claude.com/blog/multi-agent-coordination-patterns)（[宝玉繁中翻譯](https://x.com/dotey/status/2043240706156728322)）

### 1️⃣ Generator-Verifier（生成–驗證）

- **運作**：一個 agent 產出結果 → 另一個專門挑錯 → 不通過就打回去重做，直到通過或達上限
- **適用**：程式碼生成（一個寫、一個跑測試）、事實核查、合規檢查
- **關鍵設計**：**驗證標準要具體明確**，否則驗證者只是橡皮圖章
- **反直覺發現**（Cognition/Devin 實驗）：驗證 agent **不共享產出 agent 的 context** 反而效果更好 — 乾淨 context 能更深入推理，不被前面累積的雜訊干擾

### 2️⃣ Orchestrator-Subagent（調度–子代理）

- **運作**：主 agent 規劃與分派、子代理處理子任務後回報
- **代表案例**：**Claude Code** — 主代理自己寫程式，需要搜尋大型 codebase 時才派子代理出去
- **建議**：**大多數情況從這個模式開始** — 能處理最廣泛的問題、協調開銷最小

### 3️⃣ Agent Teams（代理團隊）

- **vs orchestrator-subagent 的差別**：worker **持續存活、累積 context**，不是做完一個任務就消失
- **適用**：大型 codebase 遷移這類長時間、多步驟獨立工作
- **前提**：子任務之間 **真的互相獨立**，否則容易衝突

### 4️⃣ Message Bus（訊息匯流排）

- **運作**：Agent 透過 **發布/訂閱（pub/sub）** 事件溝通，無中央調度
- **適用**：事件驅動的流水線（如安全警報處理）、生態會持續擴展的場景
- **代價**：**除錯困難** — 事件在五個 agent 間串聯時要追蹤發生了什麼，需要很仔細的 logging

### 5️⃣ Shared State（共享狀態）

- **運作**：**無中央協調者**，所有 agent 透過讀寫同一個共享儲存（DB、檔案）協作
- **適用**：研究綜合類任務 — 一個 agent 找到的線索其他 agent 即時看到並跟進
- **好處**：無單點故障
- **壞處**：容易出現 **重複工作** 和 **反應式迴圈**（A 寫了發現 → B 看到回應 → A 又回應 B…）
- **必要**：設好終止條件

### 小編三個觀察

🔹 **大多數團隊從 Orchestrator-Subagent 開始就對了** — 處理最廣泛、協調開銷最小。觀察哪裡撞牆，再演化到其他模式。

🔹 **生產系統通常是混合模式** — 整體 orchestrator-subagent，某個高度協作的子任務內部 shared state。**這些模式是積木，不是互斥選擇**。

🔹 **Generator-Verifier 是 ROI 最高的起手式** — 只加一個 verifier 就能顯著提升品質，不需大改架構。很多團隊第一步是 single-agent + verifier。

## LangChain 觀察：讀取 vs 寫入

來源：[LangChain 2025/6 How and When to Build Multi-Agent Systems](https://www.langchain.com/blog/how-and-when-to-build-multi-agent-systems)

- **讀取型任務的 multi-agent 比寫入型好管理**
- **讀取型**（搜尋、研究）：天生適合並行化，多個 agent 讀同樣的東西不會衝突
- **寫入型**（寫程式碼、寫文件）：平行化容易出現衝突決策、難以合併的輸出
- **解釋了**：
  - Anthropic Research（讀取型）→ 成功用 multi-agent
  - Cognition/Devin 去年（寫入型）→ 反對 multi-agent
  - 不過 Cognition 今年立場有演化（見下節）

## LangChain 四種架構模式（2026/1）

來源：[Choosing the Right Multi-Agent Architecture](https://www.langchain.com/blog/choosing-the-right-multi-agent-architecture)

| 你的需求 | 推薦模式 |
|---|---|
| 多個不同領域（日曆、Email、CRM），需要平行執行 | **Subagents（子代理）** |
| 單一 agent 有多種可能的專長，輕量組合 | **Skills（技能）** |
| 有順序的工作流，需要狀態轉換，agent 全程與用戶對話 | **Handoffs（交接）** |
| 不同垂直領域，平行查詢多個來源再綜合 | **Router（路由）** |

### 實測觀察

🔹 **Subagents 和 Router 在多領域任務上效率最高** — 能平行 + context 隔離
- 處理三個語言的比較文件：**Subagents 的 token 用量比 Skills 少 67%**

🔹 **Skills 和 Handoffs 是有狀態的模式** — 重複請求時能省 **40–50% API 呼叫**（記住之前對話 context）

🔹 **Handoffs 無法平行化** — 只能順序執行，不適合需要同時查多個領域

### LangChain 結論

> Context engineering 才是讓 agentic 系統可靠運作的核心。

LangGraph 作為底層框架 **刻意不藏任何隱藏的 prompt 或強制的認知架構** — 為了讓開發者完全掌控 context 的組成。

## Cognition/Devin 實戰驗證

來源：[Walden Yan, "Multi-Agents: What's Actually Working"](https://cognition.ai/blog/multi-agents-working)（2026/4，去年[Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents) 的續集）

### 核心立場

**平行寫入還是行不通** — 多個 agent 同時寫程式碼會產生 **隱性的風格和決策衝突**。

**新發現的有效形態**：**寫入動作維持單線程，其他 agent 只負責提供判斷，不負責動手做事**。

### 實驗 1：程式碼審查迴圈（有效）

- 設定：Devin 寫完程式碼 → 另一個 Devin Review 專門挑錯
- 結果：
  - 平均每個 PR 抓到 **2 個 bug**
  - **58% 是嚴重問題**（邏輯錯誤、邊界情況遺漏、安全漏洞）
- **最反直覺發現**：**審查 agent 不共享寫程式 agent 的 context，效果反而更好**
- 原因：
  - 寫程式 agent 工作幾小時後累積大量 context → 注意力品質隨長度下降（他們稱為 **context rot**）
  - 審查 agent 從乾淨 context 出發、只看差異、需要什麼資訊自己去查 → 能更深入推理
- 呼應 Anthropic「context 隔離」但提供了更具體的機制解釋

### 實驗 2：聰明朋友模式（目前算失敗）

- 構想：用 **便宜快速的小模型** 當主力，遇到困難時呼叫大模型當「顧問」 → 兼顧成本和品質
- 結果：**還沒成功**
  - **SWE 1.5 太弱**，根本判斷不了「什麼時候該求助」 — 而這恰恰是整個架構最關鍵的決策
  - 後續 **SWE 1.6** 好一些但仍不到位
- 他們的判斷：**這本質上是個訓練問題** — 未來模型需要在「來回協作」環境中被訓練
- **唯一成功的變體**：兩個前沿模型互相搭配（例：**Claude + GPT**） — 利用不同模型擅長不同子任務的特性互補
  - 但這就不是省錢了，意義不同

### 編按：Anthropic Advisor Strategy 同方向

引用自 [Claude 官方實作 Advisor Strategy](https://blog.aihao.tw/2026/05/19/claude-managed-agents-architecture/)：
- Sonnet 自己完成工作 → 卡住時才向 **Opus 請求建議**
- 官方數據：**Sonnet + Opus 顧問 比單獨 Sonnet 高 2.7 個百分點，成本反而低 12%**
- 但同樣有 **「執行者過度自信而不求助」** 的問題
- 可見：這個模式的核心難題是共通的

### 更高層級委派的踩坑

- 由 **管理者 Devin** 拆分大任務、派子 Devin 執行、透過 MCP 協調進度
- 常見坑：
  - 管理者缺乏 codebase context 時會 **過度指揮**
  - Agent 誤以為跟子 agent **共享狀態**（其實沒有）
  - **跨 agent 的溝通不會自動發生** — 需要明確設計
- **無結構 swarm 架構** → 直接說是 **繞遠路**
- 實務有效形態：**拆分–執行–彙整（map-reduce-manage）**

### Walden Yan 的總結

> 所有 multi-agent 的待解問題，本質上都是 **溝通問題**：
> - 弱模型怎麼知道該向強模型求助？
> - 子 agent 發現了什麼重要資訊該怎麼傳給兄弟 agent？
> - 怎麼傳遞 context 又不淹沒接收方？

## 業界收斂的 Best Practices（六條）

### 1️⃣ 先證明 single-agent 不夠用，再考慮 multi-agent

- 量化你的瓶頸：**是 context 不夠？工具太多？需要並行探索？**
- 如果改進 prompt 或用 **context compaction** 就能解決 → 別加 agent

### 2️⃣ 從 2–3 個 agent 開始，不要一口氣蓋大系統

- 每多一個 agent 都會 **增加除錯面積**

### 3️⃣ Day 1 就建 observability

- 沒有 tracing 的 multi-agent 系統是 **不可能除錯** 的
- 必備工具：**LangSmith、OpenTelemetry**

### 4️⃣ 設定成本上限和熔斷機制

- 每個 task 設 token / 金額上限
- 凌晨三點的 runaway agent 應該 **自動停掉** 而不是燒光 API 預算

### 5️⃣ 驗證 agent 只負責挑錯，不負責接手執行

- 它的職責是 **「專門找前一個 agent 的問題」** — 跑測試、查事實、驗格式
- 發現問題 → 打回去讓原本的 agent 重做
- **不會自己動手修**、**不會接棒繼續往下產出**
- 這就是 Generator-Verifier 模式 — 兩個 agent 之間是 **對抗關係**（一個產出、一個否決），不是流水線（一個做完交給下一個繼續做）

### 6️⃣ 保持架構的可演化性

- Anthropic 自己也發現：**為 Sonnet 4.5 設計的 context reset workaround，在 Opus 4.5 上就變成了無用代碼**
- 模型能力在快速提升，**今天的 workaround 六個月後可能就是死重量**

## 未來方向：動態子代理生成（Dynamic Subagent Spawning）

### 核心思想

> **不要在設計時預先固定 agent 的分工和流程，而是讓模型在執行時根據任務需求自行決定怎麼拆解、要生成幾個子代理、各自做什麼。**

這是個 **光譜**：
- 一端：固定工作流（拖拉式建構器）
- 另一端：完全動態（模型自己決定一切）
- 大多數系統在中間，但 **趨勢往動態端移動**

### Bitter Lesson 觀點

引用自[小編另一篇《為什麼多數 Agent 框架都沒有內化 Bitter Lesson?》](https://blog.aihao.tw/2026/02/17/bitter-lesson-agent-harness/)，**Minh Pham 的觀點**：

> 現在多數 agent 架構都在做一件事 — 「模型不夠可靠，所以把可靠性寫進外層框架裡。」
> 這在產品層面合理，但本質上是把複雜度從 **可規模化的部分（模型）** 搬到 **不可規模化的部分（手工搭建的鷹架程式碼）**。

### 已經走在這方向上的實作

- **Anthropic 的 Research 系統**：主代理透過 extended thinking 動態決定任務分解，子代理是 **即時建立的新 Claude 實例**
- **Claude Code**：主代理可以把任務委派給子代理在 **獨立 context** 裡執行
- **LangChain Deep Agents**：提供 **任務工具** 讓主代理呼叫生成子代理

### 檢驗標準（很關鍵）

> **如果模型能力明年翻倍，你的系統會不會在不需要大幅重構的情況下變得更好？**

- 動態分工 → 模型進步時委派策略「免費」變好
- 硬編碼分工 → 你得手動重寫組織圖

### 短期 vs 長期

- **短期**：固定工作流在可預測性、可稽核性、成本控制上仍有優勢
- **長期**：最後勝出的 agent 架構會越來越像 **「算力分配引擎」**，而不是一張 **「手工打造的組織圖」**

## 結論

一年前的結論 — 「**預設 Single-agent，特定場景才用 Multi-agent**」 — 不但沒過時，反而被 2026 年更多論文和生產經驗驗證。但業界進步在：

| 維度 | 一年前 | 2026 |
|---|---|---|
| 「特定場景」是什麼？ | 模糊 | **Context 隔離、並行搜尋、工具專業化** |
| 「怎麼做」？ | 各自摸索 | **五種協調模式 + 四種選擇框架** |
| 「未來往哪走」？ | 未定 | **動態子代理生成** |

### 一句話帶走

> **Multi-Agent 的價值是並行覆蓋，不是分工。**
> 最好的 multi-agent 系統不像公司，更像一個 **思考者的多次草稿** — 同一個大腦在不同維度展開推理，最終合併成一個連貫的結論。

## 引用清單（可進一步追下去）

### 廠商官方
- Anthropic [Building Multi-Agent Systems](https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them)（2026/1）
- Anthropic [Multi-Agent Coordination Patterns](https://claude.com/blog/multi-agent-coordination-patterns)（2026/4）
- LangChain [Choosing the Right Multi-Agent Architecture](https://www.langchain.com/blog/choosing-the-right-multi-agent-architecture)（2026/1）
- LangChain [How and When to Build Multi-Agent Systems](https://www.langchain.com/blog/how-and-when-to-build-multi-agent-systems)（2025/6）
- OpenAI *Building Effective Agents*

### 論文
- Stanford [Single-Agent LLMs Outperform Multi-Agent Systems on Multi-Hop Reasoning Under Equal Thinking Token Budgets](https://arxiv.org/abs/2604.02460)（2026/4，Qwen3 / DeepSeek-R1 / Gemini 2.5）
- [MAST: Multi-Agent System Failure Taxonomy](https://arxiv.org/pdf/2503.13657)（1600+ cases，7 SOTA，41–86.7% 失敗率）
- Google Research [180 種配置實驗](https://x.com/GoogleResearch/status/2016621362480382213)（金融 +81% / 規劃 −70%）
- OneManCompany 論文（[@dair_ai 介紹](https://x.com/dair_ai/status/2048909068409147460)）

### 實戰文章
- Cognition [Multi-Agents: What's Actually Working](https://cognition.ai/blog/multi-agents-working)（2026/4）
- Cognition [Don't Build Multi-Agents](https://cognition.ai/blog/dont-build-multi-agents)（2025）
- Augment Code [Multi-Agent Cost Compounding](https://www.augmentcode.com/guides/multi-agent-cost-compounding)
- Antigravity Lab [Multi-Agent Production 12 Pitfalls](https://antigravitylab.net/en/articles/agents/antigravity-multi-agent-production-12-pitfalls-deep-dive)
- ihower [AI Agent 架構比較：Multi-Agent 或 Single-Agent](https://ihower.tw/blog/12776-multi-agent-or-single-agent)（2025/6 前作）

### X / Twitter 重要討論
- [@sujingshen 三省六部幻覺](https://x.com/sujingshen/status/2043898494818410731)（1600+ 讚 / 2400+ 收藏）
- [@aparnadhinak（Arize AI）通訊開銷與 evals](https://x.com/aparnadhinak/status/2017013647156138191)
- [@aneeshpappu Multi-Agent Teams Hold Experts Back](https://x.com/aneeshpappu/status/2019447577825976332)
- [@Yeuoly1（Dify）實戰血淚](https://x.com/Yeuoly1/status/2054922914458464754)
- [@dotey 宝玉中文翻譯（Coordination Patterns）](https://x.com/dotey/status/2043240706156728322)

### 同站延伸
- [Claude Managed Agents Architecture（Advisor Strategy）](https://blog.aihao.tw/2026/05/19/claude-managed-agents-architecture/)
- [為什麼多數 Agent 框架都沒有內化 Bitter Lesson?](https://blog.aihao.tw/2026/02/17/bitter-lesson-agent-harness/)（Minh Pham 觀點來源）
