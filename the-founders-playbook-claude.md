---
title: The Founder's Playbook — 用 Claude 打造 AI-Native 新創
date: 2026-05-14
category: ai-agent
description: Anthropic 給 founder 的 AI-native 創業手冊摘要：四階段（Idea / MVP / Launch / Scale）的失敗模式、Claude 三介面的分工、可執行 exercise 與案例公司數據。
tags: [ai, agent, claude, claude-code, founder, startup, anthropic]
---

# The Founder's Playbook: Building an AI-Native Startup

> **來源**：Anthropic — [The Founder's Playbook](https://claude.com/blog/the-founders-playbook)（[PDF, 36 頁](https://cdn.prod.website-files.com/6889473510b50328dbb70ae6/69fe2a55b93bb0732b1fe33c_The-Founders-Playbook-05062026_v3%20(1).pdf)，2026-05-14 發布）

## 一句話定位

Anthropic 給 founder 的 AI-native 創業手冊。核心觀點：「沒寫過一行程式碼的 founder 都在 ship production apps，10 人獨角獸（lean unicorn）從爆紅故事變成可執行計畫。」AI 把過去 quarters 壓縮成 weeks，瓶頸從「你能 build 什麼」變成「你選擇 build 什麼」。

舊路徑：`validate → raise → hire → build → raise → grow → hire → repeat`
新路徑：每個階段不再需要更大的團隊、新技能組合、新一輪募資。

---

## Founder 角色的根本轉變

Founder 從**個別貢獻者（IC）** 變成 **agent 編排者（orchestrator）**。AI 抹除了「會 build 的人」與「有 idea 的人」之間的牆 — 非技術 founder 用 AI 出 production software，技術 founder 用 AI 出 GTM 策略、財務模型、pitch deck。

### 三種 AI 能力，撐起 lean startup

| 能力 | 比喻 | 用途 |
|---|---|---|
| **對話智能與研究**（Chat / Cowork） | on-call 各領域專家 | 競爭分析、TAM/SAM/SOM、財務模型、pitch deck、PRD、devil's advocate、pre-mortem、roadmap |
| **Agentic coding**（Claude Code） | 永遠不會被卡住的工程師 | 用自然語言描述 → AI 生成、測試、debug、refactor production-grade codebase |
| **Workflow 自動化**（Claude Cowork） | 隨叫隨到的 ops 團隊 | CRM 更新、週報、文件同步、合規追蹤、scheduling，整合既有工具 |

關鍵：時機與編排（timing and orchestration）才是真本事。「The intelligence in the system is yours.」

---

## Claude 三種介面 — 選對才有效

| 任務 | 用哪個 | 為什麼 |
|---|---|---|
| 快問快答、改寫、quick brainstorm | **Claude (Chat)** | 快、對話式、無 setup |
| 研究、分析、產出最終文件 | **Claude Cowork** | 資料夾存取、connectors、skills、排程 |
| 寫 code、測試、出貨軟體 | **Claude Code** | Codebase 存取、diff、git、dev environment |

底層都是同一個 Claude；差別在**workspace 環境**。

---

# 四階段框架

## Stage 1 — Idea

**目標**：研究導向的 problem-solution fit 驗證。在動手 build 之前蒐齊「真問題、真目標族群、真有人會付錢」的證據。

**Exit 條件**（三個都要 yes）：
1. 問題是否真實且具體？能說出**誰**遇到、**多常**遇到、**多嚴重**、現在**怎麼解**
2. 你的解法是否真的對應 validation 揭露的問題（不是你原本以為的問題）？
3. 是否有足夠 signal 支撐進入 build 階段？

### 三大失敗模式

1. **把 build 誤認為 validating**：42% 的新創死於做出沒人要的東西。Agentic coding 讓 prototype 變得太容易，founder 以為「有 prototype = 假設成立」，但 prototype 只是訪談道具，**對話本身才是證據**
2. **過早 scaling**：build 太容易導致沒驗證就規模化，「sense-making 必須跑在 building 前面」
3. **客觀性流失（confirmation bias 加強版）**：你叫 AI 證明你對，它就找證據證明你對。解法是反過來叫 Claude **找反證、扮 devil's advocate**

### 具體 exercise（從 PDF 摘錄）

| Exercise | 做什麼 |
|---|---|
| 鍛尖假設 | 把「contract review takes too long」改成「In-house legal teams at mid-market companies spend 3+ days per contract review cycle because redlines are managed across email threads rather than a single version-controlled document」這種**可測試**敘述 |
| 競爭地圖 | 請 Claude 分四層（直接、間接、潛在收購者、鄰近玩家）做競爭分析，**並為每一層為什麼會威脅你寫一個強論證** |
| 客戶訪談範本 | 不要問「would you use something like this?」（generic、未來式），要問「tell me about the last time you dealt with this problem」（具體、過去式） |
| 訪談分析 | 每 5 場訪談後，叫 Cowork 同時產出**支持假設的證據**和**挑戰假設的證據**兩份清單，看不對稱性 |
| Outreach 自動化 | 給 Cowork 目標 profile → 自動抓 contacts → 個人化邀訪信 → 串 Gmail / Calendar via MCP 安排會議 → D7 follow-up |
| Solution 壓力測試 | 把你的解法概念給 Claude，請它找出最依賴的 3 個假設，每個假設「what would have to be true」與「if not, what's the consequence」 |
| 最小 prototype | 定義 single core interaction，叫 Claude Code 只做那個，找 5 位 target profile 受訪者，**那 5 場對話決定要繼續還是回頭** |

---

## Stage 2 — MVP

**目標**：把驗證過的問題翻譯成真正可用的產品，蒐集 **product-market fit (PMF)** 證據。同時要快 **且** 不累積會反噬的 technical debt。

**Exit 條件**：有具體可辨識的用戶群展現 retention（會回來）、revenue（會付錢）、referral（會推薦）其中至少一項。

### 四大失敗模式

1. **Agentic technical debt**：AI 拆掉所有「節制 production 的瓶頸」。沒有 spec 與 architectural constraint 寫在 AI 讀得到的地方（如 `CLAUDE.md`），每次 session 都從零推導架構決策 → drift → 最後沒有 coherent mental model
2. **誤判 PMF（false PMF）**：launch 高峰來自一次性力量（朋友圈、HN 頭條、投資人 portfolio），不等於第 6、12 週還有人留下
3. **零摩擦的 scope creep**：每個追加功能都「合理」，因為加一個 feature 只要一個下午。解法是**事先寫好 scope doc**，定義「what the product does, what it deliberately does not do, what evidence justifies adding more」
4. **經驗不足的不安全感（insecure by inexperience）**：agentic code 是「會動的 code」，不等於「安全的 code」。漏洞是隱形的，產品上線到真實用戶就是真實風險

### 關鍵實作建議

- **先寫 `CLAUDE.md`，再寫 production code**：架構文件是第一個 artifact，每次 session 開始先 revisit scope doc + 載 `CLAUDE.md`，session 結束更新 log。「五分鐘 doc 換 architectural drift 的保險」
- **Sean Ellis Test**：問活躍用戶「How would you feel if you could no longer use this product?」，>40% 回「very disappointed」= 有意義的 PMF 訊號
- **Effort Test**：PMF 前留住用戶要 founder 拼命推；PMF 後產品自己拉用戶，「things begin pulling instead of pushing」是最清楚的 shift
- **Security review**：上線前用 Claude 跑 auth & session handling、API data exposure、injection、known CVE dependencies。**Claude Code Security**（目前 limited beta）能做進一步掃描
- **PMF false positive 清單**：要事先定義 false positive 長什麼樣（如：signup 沒 activation、revenue 沒 retention、初期熱度沒 repeat usage），數據進來後叫 Claude **扮懷疑論者**論證「為什麼這些不算數」
- **Pivot 診斷**：3 個 iteration cycle 沒進展，叫 Claude 分析三題 — (a) 數據裡是不是有 segment 反應不同？(b) 缺口是定位問題還是產品問題？(c) 現有產品要達 PMF 需要什麼條件，現實嗎？

---

## Stage 3 — Launch

**目標**：把早期 traction 轉成可重複的成長引擎；同時讓 founder 從每件事中抽身。

**Exit 條件**（三個都要）：
1. 成長可重複，有 channel；CAC、LTV、payback period 都有數字
2. 產品 production-grade，infra、security、compliance 都到位
3. Ops 沒有 founder 瓶頸（你不再親自跑 support、triage、sprint planning、reporting）

### 四大失敗模式

1. **Technical debt 到期**：MVP 為了速度欠的債開始收利息
2. **Founder 變瓶頸**：MVP 時 founder 在每個 loop 是優勢，Launch 時就是限制。徵兆 — 「1 小時的決策要等一週、support 堆積、ops 要 founder 親自記得」
3. **Security & compliance 不能再延**：beta 用戶時的「理論風險」，production 之後變實際暴露面
4. **太早擴張**：新市場 ≠ 新成長機會，新合規/支付/期待會打破你能看懂數據的能力

### 關鍵實作建議

- **Technical debt remediation**：用 Claude Code 跑 architectural audit → 把報告餵 Claude **排序**（哪些下個 release 前一定要解、哪些可緩、哪些是可接受 debt）→ 順便把 MVP 時只活在你腦中的架構決策寫進 `CLAUDE.md`
- **建「不需要 founder 觸發」的 PM 系統**：sprint 節奏、最低 spec template、bug triage 決策樹、weekly metrics brief；用 Cowork 跑 scheduling、routing、report compilation
- **Compliance as workstream**：不是 one-time project，要持續維護。針對 SOC 2 / GDPR / HIPAA 之類目標市場要求的框架做 code-level review
- **三個工具互為輸入輸出**：Code build 產品、Cowork build 公司、Chat operationalize 知識 → 「small team runs like a company nx its size」

---

## Stage 4 — Scale

**目標**：從千級用戶到百萬級、從一個市場到多個；建立可審計的成長與成熟的組織。Founder 重心從 builder 轉向公眾面 executive（analyst briefing、IPO roadshow），但仍維持 lean 結構。

**Exit 條件**：公司在 founder 越來越不參與 day-to-day 的情況下仍可持續。三種典型出場 — 不再需外部資金的持續盈利、IPO-ready、或被併購。回答得了「If a well-funded incumbent copied your product today, would your users stay?」

### 四大失敗模式

1. **委派 ops layer 的心理障礙**：scale 時系統要能自己跑，不能 babysit；hands-on founder 在心理與結構上都難交棒
2. **技術 ops 擴展**：客戶評估的不只是產品，而是組織能否當「可靠的基礎設施合作夥伴」— SLA、文件、可靠性保證
3. **組織職能擴展**：HR、payroll、accounting、legal ops 不管團隊多小都要有
4. **GTM 沒準備好**：organic growth 有上限，founder-led selling 通常在 scale 階段撞牆

### 關鍵實作建議：**護城河 = 深度**

> 「accumulated depth — expertise built into your product, integration depth with users' other tools, proprietary system data and workflows」

四種建立深度的方法：

1. **Bottleneck Map**：請 Claude 列出每個 workflow / decision / approval 經過你，然後模擬「你連續一週不在」，**會卡住的就是你還太 hands-on 的地方**
2. **Enterprise readiness gap analysis**：選三個你最想簽的 enterprise prospect，叫 Claude 列出他們採購團隊會要求的 documentation / SLA / support infra，對照你目前缺什麼 → 用 Code + Cowork 排序工程與文件工作
3. **Domain knowledge → AI context**：透過 extended conversation / projects / memory 把行業知識（jargon、regulatory gotchas、edge cases、why obvious answers don't work）灌進 Claude；用 Skills 把 recurring workflow 編成 reusable routine。**範例**：「How I audit a commercial lease」、「How I triage a patient intake form」。隨時間累積 = 任何 generalist AI 無法匹敵的 proprietary knowledge substrate
4. **Data flywheel + workflow lock-in**：每個用戶互動產生 behavioral signal → 改進產品 → 更多使用 → 更多 feedback。為前 10 大客戶做 workflow integration audit，列出他們在你產品上建的自動化、依賴的整合、團隊 workflow、預估 switching cost，找出「什麼類型 integration 在你產品上 lock-in 最深」

**Edge case 護城河 exercise**：找一個 generic 競爭者一定做錯的 vertical edge case（如「medical billing 對 340B drug program 的處理邏輯」），用 Claude Code 寫成真實場景測試（不是 unit test），每次出現類似 case 就加進去。**你的測試套件變成你護城河的地圖。**

---

## 案例公司（含具體數據）

| 公司 | 用 Claude 做什麼 | 數據／重點 |
|---|---|---|
| **Carta Healthcare** | Clinical abstraction platform | 每年處理 22,000 例手術案例，data abstraction 時間 **減少 66%** |
| **Anything** | 用 Agent SDK 幫人無 code build 軟體 | 服務 **1.5M 用戶**，含一位非技術 founder 已在賣完整 recruiting platform |
| **Cogent** | Applied AI lab，自動化企業 security 任務 | Claude 當 reasoning layer 跑 investigation / prioritization / remediation |
| **Airtree** | Ops infrastructure | 用 Cowork 統一原本散在十幾個工具的資料；個人建的 skill 全組織可用 |
| **Duvo** | 採購 / 供應鏈 / category 管理 agents | 跨 ERP、supplier portal、spreadsheet、email、phone call；Agent SDK 編排 |
| **Zingage** | 居家照護 24/7 自動化平台 | Claude structured tool calling 串 EMR 與多溝通管道 |
| **Kindora** | NPO matching | 非工程出身執行長用 Claude Sonnet 從零做出來；MCP connector 讓 NPO 直接在 Claude 內用 |
| **Wordsmith** | 法務 AI | 律師轉 CTO 創辦；Claude 做 reasoning，自家工程團隊用 Claude Code 開發 |
| **HumanLayer / Ambral / Vulcan Technologies** | YC 三家用 Claude Code 從 prototype 到 scale | 詳見 Anthropic blog "How three YC startups built their companies with Claude Code" |
| **GC AI** | In-house legal team 平台 | Company-specific playbook、跨職能 stakeholder、variable risk tolerance |

---

## 一句話總結

> 「The bottlenecks are no longer what you can build, but what you choose to build.」

Founder 工作沒變（找真問題 → 解 → scale 成公司），變的是路徑：validation cycle 從月變天，prototype 不需 co-founder，launch readiness 變持續工作流，scale 階段的 ops 重量交給 AI 處理。

---

## 相關資源（PDF Resources 章節）

**Building with Claude**
- Building AI Agents for Startups
- Claude Code docs（"How Claude Code works" 是好的起點）
- Claude Code best practices（context、permissions、planning、verification）
- Using `CLAUDE.md` files
- Claude Code power user tips（parallel session、verification loop）
- Get started with Claude Cowork
- claude.com/resources/tutorials

**Founder stories**
- HumanLayer (F24) + Ambral (W25) + Vulcan Technologies (S25) — YC 三家用 Claude Code
- GC AI、Carta Healthcare、Anything、Cogent、Airtree、Duvo、Zingage、Kindora、Wordsmith

**Anthropic Startups Program**：與 Anthropic VC 合作的新創可拿免費 API credits、最高 rate limit、founder event 邀請
