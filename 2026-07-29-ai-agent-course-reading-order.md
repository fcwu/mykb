---
title: 五支 AI Agent 工程影片，先看哪一支
date: 2026-07-29
category: ai-agent
description: Google Vibe Coding / Agentic Engineering 課程摘要、兩支延伸影片與一支 Stanford 系統設計課的閱讀順序建議，以及它們合起來真正在講的那件事
---

# 五支 AI Agent 工程影片，先看哪一支

> **來源**：本篇是對 YouTube 頻道 Gary Chen 五支影片的綜合整理，各支的完整摘要與原始連結見文末。

Google 那套 Vibe Coding / Agentic Engineering 課程的中文摘要，七月一個月內連出了四支。大家的反應大多是先收藏，然後停在第一支的一半——四支加起來快七十分鐘，彼此重複的段落又不少，很容易看到後面覺得「這不是剛剛講過了嗎」。

我按發布時間全部看完，結論是：**重複不是問題，順序才是問題。** 這四支的依賴關係跟發布時間剛好是反的。照時間順序看，你會在還沒拿到骨架之前先撞上最進階的那一支，然後把後面三支讀成廢話。

重排一下，四支就從四篇獨立摘要變成一條有梯度的路。

還有第五支。同一個頻道五月發過一支 Stanford「Beyond LLM」課程的濃縮，27 分鐘，跟前面四支重疊很多——MCP、eval、context、multi-agent 全都講。但它不屬於同一條軸：那四支問的是「我怎麼用 AI 把 code 寫好」，這支問的是「我怎麼設計一個 AI 系統給別人用」。所以它不是第五步，是另一條路的第一步。

## 1. 先講結論

| 順序 | 影片 | 一句定位 | 發布 |
|------|------|---------|------|
| 1 | [Day 1：Vibe Coding 光譜、Context Engineering、Agent = Model + Harness](2026-07-28-yt-google-vibe-coding-agentic-engineering-day1/summary.md) | 開發者這條軸唯一的理論骨架 | 7/10 |
| 2 | [AI 改 code 一直改 A 壞 B？你缺的是這五個步驟](2026-07-28-yt-ai-code-fix-a-break-b/summary.md) | 明天就能用的那一支 | 7/18 |
| 3 | [Day 2+3：MCP、A2A、Skills 解析](2026-07-28-yt-google-ai-course-day2-3-mcp-a2a-skills/summary.md) | 工具層與 skill 的工程紀律 | 7/26 |
| 4 | [Loop Engineering 解析，大神都不寫 prompt 了？](2026-07-28-yt-loop-engineering/summary.md) | 方向，不是 to-do | 7/01 |
| — | [一部影片看完 Stanford AI 系統課程，從 LLM 到 Agentic Workflow](2026-07-29-yt-stanford-ai-systems-course/summary.md) | 另一條軸的全景地圖 | 5/04 |

這四支裡發布最早的排最後，理由在第 6 節。至於發布時間比它們都早的 Stanford 那支為什麼不編號，理由在下一節。

## 2. 先分軸，再排序：Stanford 那支放哪

[Stanford AI 系統課程](2026-07-29-yt-stanford-ai-systems-course/summary.md) 是整組裡發布最早的（5/04），也是唯一一支從「系統怎麼設計」而不是「開發者怎麼工作」切入的。

它給的是一張五層地圖：

| 層 | 技術 | 教授的態度 |
|---|------|-----------|
| Lv1 | Prompt Engineering | 最低成本，重點在 chaining 不在修辭 |
| Lv2 | Fine-Tuning | 能不做就不做 |
| Lv3 | RAG | AI 產品標配 |
| Lv4 | Agentic Workflow | 心態要從 deterministic 換成 fuzzy |
| Lv5 | Multi-Agent | 一個 agent 能解決就別硬上 |

把 Google 那四支放進這張地圖會發現，它們**全部住在 Lv4 的一個角落**——講的都是 agentic workflow，而且都是「agent 幫我改 codebase」這個特定應用。Lv1 到 Lv3 那三層幾乎沒碰。

所以順序其實是個岔路，看你要做什麼：

- **想讓 AI 把 code 寫得更好** → 照第 1 節那四支的順序看，Stanford 這支可以先跳過。裡面的 chunking、vector database、embedding 對你當下沒用。
- **想蓋一個 AI 產品或 agent 系統** → 先看 Stanford 拿全景，再回頭進 Day 1。反過來看的話，你會誤以為 harness 六大件就是全部。

兩邊都要的話，最省時間的接法是 Stanford → Day 1 → 改 A 壞 B → Day 2+3 → Loop Engineering。

這支相對其他四支最有增量的部分是 **eval**。Day 2+3 給的是四道防線（上線前該做什麼），Stanford 給的是一個三維度交叉的框架（設計時該想什麼）：

- **End-to-End vs Component-based** — 只看整體，你知道哪裡壞但不知道為什麼壞；只看元件，你可能修了細節但整體體驗還是差
- **Objective vs Subjective** — 前者寫 script 就能自動對答案；後者靠人工評分或 LLM-as-Judge
- **Quantitative vs Qualitative** — 後者要人工一筆一筆看，沒有捷徑

加上一條操作原則，我認為是這支最實用的一句：**先人工掃出問題，再設計自動化 eval**。從一千個對話抽二十個真的讀完，把讀到的問題翻成 rubric，而不是坐在位子上想像失敗場景。再加一條：**模型跟 prompt 一次只動一個變因**，否則你得到的任何比較數字都不可信。

## 3. 第一支：先拿骨架

[Day 1](2026-07-28-yt-google-vibe-coding-agentic-engineering-day1/summary.md) 是 Google 那四支裡唯一提供框架的一支，另外三支都在引用它——`Agent = Model + Harness`、六種 context、factory model，後面三支開口就用這些詞。先看這支，其他三支才有掛鉤點。

你會拿到三樣東西：

- **一條光譜，不是一個開關。** vibe coding → structured AI-assisted coding → agentic engineering，判斷你在哪一段的標準不是「你用不用 AI」，而是 AI 輸出周圍有多少結構、驗證與人類判斷。
- **harness 六大件**（rule files、tools、sandbox、orchestration、hooks、observability）。這六格拿來盤點自己的環境剛好，缺哪一格會很明顯。
- **token 經濟學**：vibe coding 是低 CapEx、高 OpEx，那三項營運成本（token 燃燒率、維護稅、資安補救）會複利成長。

順帶一句：這支裡最好用的不是概念，是那兩個實證——某團隊在 Terminal Bench 2.0 完全不換 model 只改 harness，從 30 名外進前 5；LangChain 只調 system prompt、tools 跟 middleware 加了 13.7 分。要說服人「別再等下一顆模型」，這兩個數字比任何論述都快。

## 4. 第二支：可操作性最高的那一支

[改 A 壞 B](2026-07-28-yt-ai-code-fix-a-break-b/summary.md) 放第二，因為它是 Day 1 那句「context engineering 就是幫新員工做入職簡報」在真實場景裡的具體長相。

它的核心診斷一句話講完：網路上教的全是 Greenfield（空地蓋新房），你每天面對的是 Brownfield（在別人住的房子裡拆牆改管）。而且——

> Greenfield 不會永遠是 Greenfield。vibe coding 狂寫一個月而沒有意識地維持架構，那個專案已經是 Brownfield 了，因為裡面大部分的 code 你根本沒讀過。

這支最直接可複用的產出是一張規格模板，把前半場探勘到的東西全部變成 agent 的邊界：

```xml
<task>       這次要做什麼
<read-first> 動手前必須先讀哪幾個檔案
<requirements> 沿用既有 pattern、重用現成元件
<forbidden>  絕對不准動的東西
```

`<read-first>` 是 static context 的最小可行版本，`<forbidden>` 就是 Day 1 harness 六大件裡的 guardrails。換句話說，這支示範了怎麼**臨時**組一份 harness，不需要先建好一整套 rule file 才能開工。

還有一句反直覺但我認為是整組素材裡最實用的一句：**在 Brownfield 裡，Clean Code 不是你覺得漂亮的寫法，而是跟前人一致的寫法。**

## 5. 第三支：把 skill 當軟體

[Day 2+3](2026-07-28-yt-google-ai-course-day2-3-mcp-a2a-skills/summary.md) 放第三，不是因為它比較難，是因為它的價值集中在後半段，而後半段需要前兩支當動機。沒有前面的鋪陳，那些防線會被讀成一張沒有理由的 checklist。

前半段講 MCP 與 A2A，其中最值得記的是**為什麼 MCP 不能代替 A2A**：MCP 像計算機（單向、無狀態、做完就沒了），A2A 像同事（有記憶、多回合、會反問你「這邊有異常值，要刪還是留」）。

後半段是重點：Skill 進 production 的四種失敗（trigger / token budget / execution / regression）對上四道防線（Evals as Unit Tests、Golden Dataset、Red Team、Shadow Mode + Canary）。裡面有一個要求，四支裡只有這支講：**測 Skill 不只看最終產出，還要攤開它呼叫了什麼工具、按什麼順序**——驗 trajectory，不只驗結果。

`skill.md` 超過 5000 字就該拆、觸發準確率至少 90%，這兩個門檻可以當粗略的體檢線。

## 6. 第四支：方向，不是待辦

[Loop Engineering](2026-07-28-yt-loop-engineering/summary.md) 發布最早卻排最後，因為它站在前三支的肩膀上——而且它自己就說了，**大多數人現在不需要 loop**。

它的骨架很乾淨：一個 loop 只由兩個問題定義，**Trigger**（什麼時候開始）與 **Verifiable Goal**（什麼時候停止）。而 Verifiable Goal 正是 Day 1 那個「evals」的具體化：主觀任務用 rubric 分級或二元檢查清單轉成可驗證條件，作者說**二元清單比打分數穩定**。

失控三問題三規則也值得抄：停不下來 → hard stop（最多幾輪、多久、多少 token、幾輪沒進展就回報）；亂改邊界 → 明確寫出不能刪測試、不能改公開 API、不能動 schema；Verifier 失效 → 驗收標準盡量機器可檢查。外加一條，我認為是這支最重要的一句：

> 不是有分數就代表客觀。如果你叫 AI 做任務，然後 AI 自己評分，那就是球員兼裁判。

## 7. 五支其實只在講同一件事

看完再回頭，會發現重複的段落不是懶，是同一個原則在不同層級反覆現形。五支裡有四支各自寫了一段勸退，內容不一樣，門檻完全一樣：

- Loop Engineering：大多數人現在不需要 loop，作者自己仍偏好 human in the loop——更準、更有效率、更便宜。
- Day 2+3：公司評估系統沒建好前，強烈建議不要碰 Meta-Skill，否則 AI 只是瞎子摸象亂改。
- 改 A 壞 B：production 環境下，除非你很熟這套 codebase 或它已有很好的 harness，重要決策先不要外包給 agent。
- Stanford：一個 agent 就能解決的任務硬上 multi-agent 只會增加複雜度，工程設計能簡單就簡單，不要 over design。

四句話疊起來就是一條成熟度階梯：

```text
能驗證  ──▶  才能放手  ──▶  才能自動迭代
(evals)      (agent 動手)      (loop)
```

跳級的代價都很具體。沒有 verifier 的 loop 是 token 黑洞；沒有 evals 的 Meta-Skill 是亂改；沒有探勘就動手的 agent 就是改 A 壞 B。

同一條軸上還有兩個到處出現的推論。第一，**可外包的是勞力活（讀、找、生成），不可外包的是拍板**——「起草跟拍板是兩回事，改壞了被 QA 找麻煩的是你」。第二，**產出與驗證必須是兩個角色**，這條在五支裡以四種形式出現：reviewer agent、Red Team、人工檢查點，以及 Stanford 那支的 LLM-as-Judge。

至於 context，處理方式其實是同一個機制的四種尺度：Day 1 的 static / dynamic 二分、Day 2+3 的 `skill.md` 骨架 + `references/`、改 A 壞 B 的 `<read-first>`、Stanford 的 working / archival memory 二分——全都是 progressive disclosure。Day 1 把它推到財務層那句話說得最好：**你不能決定模型的費用，但可以用比較少的 token 完成一樣的任務。**

## 8. 沒時間全看的話

- **只看一支** → 改 A 壞 B。單位時間可操作性最高，XML 模板可以直接複製去用。
- **你的目標是蓋 AI 產品而不是寫 code** → Stanford 那支。它是唯一一支給完整地圖的，而且 27 分鐘涵蓋 Lv1 到 Lv5。
- **要說服團隊投資 evals 與 harness** → Day 1 的 token 經濟學那節，加上第 3 節那兩個 benchmark 實證。
- **已經很熟 MCP** → Day 2+3 的前半段（挑選流程、三條安全紅線）可以跳過，直接從「Skill 進 production 的四種失敗」開始。

## 9. 我的看法

**框架可以用，數字先別引用。** 五支裡最有說服力的那些量化——METR 說資深工程師用 AI 反而慢 19%、Terminal Bench 從 30 名外進前 5、LangChain 加 13.7 分、觸發準確率 90%、`skill.md` 5000 字上限——沒有一個標了出處。Stanford 那支更徹底：影片只說「Stanford 有一堂課叫 Beyond LLM」，沒給課號、教授姓名或公開連結，整支是 2 小時課程的二手濃縮，所有「教授說」都無法回溯；連被引用最多次的 BCG 那個顧問實驗也只給了公司名。當成方向感很好，寫進簡報前得自己找原文。

**「有好的 harness 就不會有這些問題」這句講得太輕。** 這是改 A 壞 B 結尾回扣主線的說法，我不同意。就算 rule file 與 context 管理都完備，一個共用 hook 的 blast radius 仍然只能靠實際探勘才知道——那是 codebase 的事實，不是能預先寫進規則的偏好。這也是四支裡我唯一明確反對的一句。

**範例的重心明顯偏前端與個人開發者。** 「打開 `package.json` 看狀態管理」「用開發者工具檢查元素定位檔案」這類手法，換到後端、系統層或多 repo 的環境就要自己翻譯（用 build system 檔案、grep 呼叫圖代替）。五步流程的骨架是通用的，具體手法不是。

**一個橫跨兩條軸的分歧，值得單獨拿出來看。** 對 multi-agent 的態度，Google Day 2+3 講 A2A 與 Agent-as-a-Service 時明顯樂觀，而 Stanford 那支跟 Loop Engineering 都在勸退——「一個 agent 能解決就不要硬上」。同一批素材裡出現方向相反的判斷，本身就是有用的資訊：**Google 在推生態系，另外兩支在講工程成本。** 你要相信哪一邊，取決於你是在賣平台還是在維護它。

**最後一點：勸退段反而是整組素材裡最該先讀的部分。** 這類內容的常態是把新名詞包裝成你不用就落後了；五支裡有四支花篇幅講「你可能不需要這個」，其中一支還直接說「這些功能的需求來源大多是 OpenAI 跟 Anthropic 的工程師，他們用得到的我們不一定用得到」。就這點而言，它們比多數同題材內容誠實。

## 10. 原始連結

各支的完整重點摘要（含講義截圖）：

- [Day 1：Vibe Coding 光譜、Context Engineering、Agent = Model + Harness](2026-07-28-yt-google-vibe-coding-agentic-engineering-day1/summary.md) — 影片：<https://www.youtube.com/watch?v=GzHfE50N8x4>（00:15:57）
- [AI 改 code 一直改 A 壞 B？你缺的是這五個步驟](2026-07-28-yt-ai-code-fix-a-break-b/summary.md) — 影片：<https://www.youtube.com/watch?v=CMs8YMU6_RM>（00:15:54）
- [Day 2+3：MCP、A2A、Skills 解析](2026-07-28-yt-google-ai-course-day2-3-mcp-a2a-skills/summary.md) — 影片：<https://www.youtube.com/watch?v=XTCP1qoa3cc>（00:22:17）
- [Loop Engineering 解析，大神都不寫 prompt 了？](2026-07-28-yt-loop-engineering/summary.md) — 影片：<https://www.youtube.com/watch?v=kGYFSDd-ZVY>（00:13:06）
- [一部影片看完 Stanford AI 系統課程，從 LLM 到 Agentic Workflow](2026-07-29-yt-stanford-ai-systems-course/summary.md) — 影片：<https://www.youtube.com/watch?v=eKW9ITaltWw>（00:27:23）

Day 1 的講義本體為 Google《The New SDLC with Vibe Coding》，共 51 頁；Stanford 那支的原始教材是「Beyond LLM」課程，約 2 小時。
