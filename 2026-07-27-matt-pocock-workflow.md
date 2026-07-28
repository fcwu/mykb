---
title: "Matt Pocock Skills Workflow：OpenSpec 主從整合"
date: 2026-07-28
category: ai-agent
description: "比較 Matt Pocock Skills 與 OpenSpec，說明如何以 OpenSpec 管理變更生命週期、以 Matt 強化工程實作。"
---

# Matt Pocock Skills Workflow：定位與 OpenSpec 主從整合

> **來源**：[Matt Pocock · skills](https://github.com/mattpocock/skills)（擷取於 2026-07-28）

> 研究日期：2026-07-27。來源以 [mattpocock/skills](https://github.com/mattpocock/skills) 的現行文件為準。

## 結論

Matt 的 skills **有 workflow**，但它刻意不是一個強制、包辦整個開發生命週期的框架。它的主線是：

```text
grill → spec → tickets → implement → review
```

這條線由「使用者主動呼叫」的 orchestration skills 串起；領域建模、原型、TDD、除錯與 code review 等則是可被流程呼叫的工程紀律。官方將它定位為小型、可調整、可組合的 skills，而不是接管流程的框架。[README](https://github.com/mattpocock/skills/blob/main/README.md)

一個實務上較穩定的分工是：**OpenSpec 作為唯一的規格與變更生命週期主體；Matt 作為釐清、設計與實作品質的輔助工具。** 不要同時讓 OpenSpec proposal 和 Matt `to-spec` 各自成為同一變更的權威規格。

## Matt 的現行主流程

| 階段 | Skill / 產物 | 用意 |
| --- | --- | --- |
| 一次性設定 | `setup-matt-pocock-skills` | 設定 issue tracker、triage labels、`CONTEXT.md` 與 ADR 的位置。 |
| 釐清 | `grill-with-docs` | 深問需求，同步建立共通詞彙與重要 ADR；可帶動 `grilling`、`domain-modeling`。 |
| 規格 | `to-spec` | 將既有對話與 codebase 理解整理成 PRD/spec，發佈至設定好的 issue tracker。 |
| 拆票 | `to-tickets`（可選） | 拆成可獨立驗證的 tracer-bullet 垂直切片，並標記 blockers。 |
| 實作 | `implement` | 依 spec/tickets 實作，在預先同意的 seam 使用 TDD，持續 typecheck 與測試。 |
| 收尾 | `code-review` → commit | 實作 skill 會要求先 review，再提交目前 branch。 |

`ask-matt` 是路由器：不知道該選哪一條線時才呼叫它，本身不做規格、訪談或實作。它也把主流程明確描述為「idea → ship：grill → spec → tickets → implement → review」。[ask-matt 文件](https://github.com/mattpocock/skills/blob/main/docs/engineering/ask-matt.md)

兩個容易忽略的分支：

- `triage` 是外來 bug／需求的入口，而不是主流程必經步驟。
- `wayfinder` 用於超過一次 agent session 的大型、不確定工作，以 tracker 上的調查 tickets 逐步找路。

## 四個特別值得保留的 standalone skills

這三個不是每個 change 的固定步驟，卻各自解決了 workflow 很難取代的問題。

### `writing-great-skills`：讓 skill 行為可預期

它不是「如何寫一篇好說明文件」，而是如何讓 agent skill 在多次執行時走出一致的 process。重點包括：model-invoked 與 user-invoked skills 的 context load / cognitive load 取捨、以 completion criterion 防止過早結束、將 reference 做 progressive disclosure，以及用單一真相、pruning、leading words 減少重複與冗詞。

適用於新增、修改或 review reusable skills 時。它可與既有的 skill 測試紀律並用：前者改善 skill 的資訊架構與可預期性，後者驗證 skill 在壓力情境下是否真的改變 agent 行為。[writing-great-skills](https://github.com/mattpocock/skills/blob/main/skills/productivity/writing-great-skills/SKILL.md)

### `teach`：把一次問答變成可持續的學習空間

`teach` 把目前目錄當成一個有狀態的學習 workspace：用 `MISSION.md` 記錄學習動機、`RESOURCES.md` 管理可信來源、learning records 記錄已理解的洞見，並以短小、可回訪的 lesson 和 reference documents 累積進度。它的重點不是「解釋一次」，而是讓後續內容能依使用者的能力邊界與長期保留需求調整。

適合明確要跨多次 session 學習一個主題時。因為它會建立多種 workspace artifacts，應放在專屬學習目錄，而非隨意在一般 repo 根目錄啟動。[teach](https://github.com/mattpocock/skills/blob/main/skills/productivity/teach/SKILL.md)

### `wayfinder`：在迷霧中先找路，不急著交付

`wayfinder` 適合一個 session 無法容納、但目前也還不能寫成可靠 spec 的巨大工作。它建立的是 issue tracker 上的「決策地圖」：每張 ticket 要解決一個 decision，而不是交付一片 implementation；已知但尚無法精確表述的部分則留在 fog of war，隨著決策逐步長成新的 tickets。

它最重要的界線是 **plan, don't do**。地圖完成時代表路徑清楚，而不是功能已實作。因此在 OpenSpec 主導的架構中，`wayfinder` 是大型探索的入口；當決策收斂後，將結果壓縮為一個或多個 OpenSpec changes，再由 proposal / tasks / apply 接手實作。[wayfinder](https://github.com/mattpocock/skills/blob/main/skills/engineering/wayfinder/SKILL.md)

### `improve-codebase-architecture`：先看摩擦，再決定要不要重構

這個 skill 的目標不是列出一堆「可以重構」的清單，而是找出讓未來修改變容易的 deepening opportunities：介面相對太複雜的 shallow modules、理解一個概念時必須在許多小模組間跳轉、難以在正確 seam 測試的區域，或耦合外洩的邊界。它會優先從近期 commit history 找 hot spots，並用 `codebase-design` 的 module、interface、depth、seam、locality 等詞彙來描述問題。

輸出是一份暫存的 HTML report，每個候選項都有問題、方案、效益、before/after 圖與推薦強度；先讓人選擇值得探索的項目，再用 grilling 和 domain modeling 釐清設計。因此它適合定期的 codebase health review，或 bug 修好後發現「根本沒有可鎖住問題的 seam」時使用。它不直接授權廣泛重構：候選方案獲得認可後，仍應由 OpenSpec proposal 承接範圍、設計與工作清單。[improve-codebase-architecture](https://github.com/mattpocock/skills/blob/main/skills/engineering/improve-codebase-architecture/SKILL.md)

## 為何看起來不像 OpenSpec

| 面向 | Matt Pocock Skills | OpenSpec |
| --- | --- | --- |
| 設計哲學 | 小而可組合；使用者選擇、串接需要的 skills。 | 有明確變更生命週期與 artifacts。 |
| 規格真相 | `to-spec` 發佈到設定的 issue tracker。 | repo 內的 proposal、design、tasks、delta specs。 |
| 任務管理 | `to-tickets` 的垂直切片與 blocker 關係；可發佈 tracker 或 local files。 | `tasks.md` 隨 change 管理，完成後 sync / archive。 |
| 實作入口 | `implement`，內含 TDD、測試、review 與 commit。 | `apply-change`，再由 `verify`、`sync-specs`、`archive` 收尾。 |
| 適用重點 | 讓 agent 的需求理解、設計與回饋迴圈更像資深工程師。 | 讓 repo 內的規格演進可追溯、可審查、可封存。 |

所以 Matt 並非「沒有 workflow」；它是**由 skills 組成的可選流程**。OpenSpec 則是**由檔案 artifacts 與狀態轉換組成的流程**。Matt 的 `to-spec` 甚至明確要求把 spec 發佈到 issue tracker；`to-tickets` 也會發佈 ticket 或寫入 local files。[to-spec](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-spec/SKILL.md) [to-tickets](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-tickets/SKILL.md)

## 建議的主從流程：OpenSpec 主、Matt 輔

```text
需求／想法
  →（需要釐清時）Matt grill-with-docs、domain-modeling、prototype
  → OpenSpec explore / propose
      └─ proposal、design、specs、tasks 是唯一權威
  →（需要跨人協作時）將已核准的 OpenSpec tasks 單向映射為 Jira / GitHub tickets
  → OpenSpec apply-change
      └─ 使用 Matt 的 tdd、codebase-design、diagnosing-bugs、code-review
  → OpenSpec verify → sync-specs → archive
```

### 在這個架構中要用什麼

- 在 `openspec explore` 或 `propose` 前／中：使用 `grill-with-docs`、`grilling`、`domain-modeling`、`prototype`、`research`。它們幫你把問題講清楚、建立詞彙、驗證高風險設計。
- OpenSpec proposal 建立後：由 OpenSpec 擁有需求、設計、驗收規格與工作清單。
- 若需要 Jira / GitHub 排程、多人並行：可把**已核准**的 OpenSpec tasks 映射成 tickets；必須註明 OpenSpec change 路徑，且變更仍回寫 OpenSpec，不在 ticket 內另寫一份規格。
- 在 `openspec apply-change`：使用 Matt 的 `tdd`、`codebase-design`、`diagnosing-bugs`、`code-review` 等葉節點 skills，強化每個實作切片的工程回饋。
- 完成時：仍由 OpenSpec 的 `verify`、`sync-specs`、`archive` 做驗證、現況規格同步與封存。

### 預設不要納入主線的 Matt skills

在 OpenSpec 為主的前提下，以下不應同時主導同一變更：

- `to-spec`：會再產生並發佈一份 tracker spec，容易與 OpenSpec proposal/design 分叉。
- `to-tickets`：只有在 tracker 是協作必要介面時使用，而且只做 OpenSpec tasks 的下游映射；不要反向把 ticket 當規格真相。
- `implement`：它自帶「依 spec/tickets 實作 → review → commit」的收尾邏輯，會與 OpenSpec 的 apply/verify/sync/archive 生命週期重疊。
- `triage`：保留給 issue intake，不放入每個一般 change 的必經路徑；大型未知工作則依上節使用 `wayfinder`，收斂後回到 OpenSpec。

## 實際判斷規則

1. 單一 repo 變更：直接 OpenSpec；遇到理解或設計風險才插入 Matt 的葉節點 skills。
2. 需求仍模糊：先 `grill-with-docs`，把共通詞彙和決策帶入 OpenSpec `explore` / `propose`。
3. 需要跨 team 排程：proposal 核准後，將 OpenSpec tasks 映射到 tracker；ticket 是執行看板，不是另一份規格。
4. 大型未知工作：先用 `wayfinder` 解出決策地圖，而不是直接實作；收斂後可形成多個 OpenSpec changes，但每個可實作變更仍回到 OpenSpec 生命周期。
5. 實作或 review：在 OpenSpec `apply-change` 內使用 Matt 的 `tdd`、`code-review` 等技能，不改變 OpenSpec 的收尾責任。

## 參考來源

- [Matt Pocock Skills README](https://github.com/mattpocock/skills/blob/main/README.md)
- [ask-matt：router 與 main flow](https://github.com/mattpocock/skills/blob/main/docs/engineering/ask-matt.md)
- [setup-matt-pocock-skills](https://github.com/mattpocock/skills/blob/main/skills/engineering/setup-matt-pocock-skills/SKILL.md)
- [to-spec](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-spec/SKILL.md)
- [to-tickets](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-tickets/SKILL.md)
- [implement](https://github.com/mattpocock/skills/blob/main/skills/engineering/implement/SKILL.md)
- [writing-great-skills](https://github.com/mattpocock/skills/blob/main/skills/productivity/writing-great-skills/SKILL.md)
- [teach](https://github.com/mattpocock/skills/blob/main/skills/productivity/teach/SKILL.md)
- [wayfinder](https://github.com/mattpocock/skills/blob/main/skills/engineering/wayfinder/SKILL.md)
- [improve-codebase-architecture](https://github.com/mattpocock/skills/blob/main/skills/engineering/improve-codebase-architecture/SKILL.md)
