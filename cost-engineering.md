---
title: Cost Engineering（agent 成本方程式與可以動的槓桿）
tags: [agent, cost, token, prompt-caching, mcp, model-routing, harness, ai-engineering]
created: 2026-09-12
last_reviewed: 2026-09-12
type: reference
status: living-document
sources:
  - Uber Engineering（@UberEng，作者 @udaykiran）— "Running a Software Factory Efficiently at Uber Scale"（2026-08-29）— https://x.com/UberEng/status/2093444169037762840
  - Anthropic — prompt caching 定價與 TTL 規則（cache write 1.25×／2×、cache read 0.1×、Fable 5.1 0.025×）；2026-09-12 對 claude-api skill 隨附文件核對
  - OpenAI — prompt caching 文件（in-memory 5–10 分鐘、extended retention 24 小時）；2026-09-12 WebSearch 核對
  - 本 repo：harness-engineering.md（§七「Token 成本」—— 本篇是它的深潛；§四 面向 1 offloading／compaction；§五 5.5 Statusline）
  - 本 repo：context-engineering.md（§2.3 漸進揭露／deferred tool loading —— 本篇 §四 4.3 給它價格）
  - 本 repo：eval-engineering.md（§四 從 trace 收割 —— 本篇 §三 選模型的 benchmark 從那裡來）
  - 本 repo：graph-engineering.md、codez-graph-engineering.md（subagent 繼承 session model 的第一方契約 —— 本篇 §三 3.2 的前提）
---

# Cost Engineering：agent 成本方程式，與六個可以動的槓桿

> repo 裡談 agent 成本的地方只有 [harness §七「Token 成本」](./harness-engineering.md#token-成本)三條 bullet。Uber 這篇是第一方、有數字、有方法論的成本報告：用量 7 倍、支出持平，而且他們把「怎麼做到」拆成一條可以逐項量測的方程式。本篇問的是：**agent 的錢花在哪一項、哪幾項是你能動的、每一項有哪些已被驗證的槓桿。**

## 目錄

- [前言：定位、缺口與來源分級](#前言定位缺口與來源分級)
- [一、成本方程式：六項相乘](#一成本方程式六項相乘)
  - [1.1 公式與三組分工](#11-公式與三組分工)
  - [1.2 五層量測](#12-五層量測)
  - [1.3 固定模型才量得到自己的功勞](#13-固定模型才量得到自己的功勞)
- [二、四層場景：越往上越能控](#二四層場景越往上越能控)
- [三、Price／Token：選模型](#三pricetoken選模型)
  - [3.1 benchmark 驅動的四步](#31-benchmark-驅動的四步)
  - [3.2 兩個預設，subagent 那個最大](#32-兩個預設subagent-那個最大)
- [四、Tokens／Request：每一輪重送的東西](#四tokensrequest每一輪重送的東西)
  - [4.1 兩個 fleet 預設：400K 壓縮、Medium effort](#41-兩個-fleet-預設400k-壓縮medium-effort)
  - [4.2 Prompt cache TTL 是一道算式](#42-prompt-cache-ttl-是一道算式)
  - [4.3 MCP schema 稅與三條路](#43-mcp-schema-稅與三條路)
  - [4.4 Code-mode：省的是 overhead，不是 payload](#44-code-mode省的是-overhead不是-payload)
  - [4.5 SaaS MCP：比內部的更胖](#45-saas-mcp比內部的更胖)
- [五、Requests／Turn：讓 agent 少找](#五requeststurn讓-agent-少找)
- [六、可見度：讓人和 agent 都看得到帳單](#六可見度讓人和-agent-都看得到帳單)
- [七、個人與小團隊可以抄什麼](#七個人與小團隊可以抄什麼)
- [八、與本 repo 其他筆記的對照](#八與本-repo-其他筆記的對照)
- [九、來源與更新](#九來源與更新)

### 閱讀路徑建議

- **只想拿可以馬上用的**：§七 的「直接抄」清單 → §4.2 TTL 決策表 → §3.2 subagent 模型
- **想要一個分析框架**：§一 方程式 → §1.2 量測表 → §二 四層。這三節合起來是一套「先量再動」的方法
- **在管團隊的 AI 支出**：§二（往上層推是戰略）→ §六（tier 與 nudge 而不是硬上限）→ §1.3（怎麼向上報告自己的功勞）
- **已經讀過 [context-engineering.md](./context-engineering.md)**：直接跳 §4.3–4.5。那篇說「別全部前載」，這裡給出前載的價格與三種替代路徑的實測

---

## 前言：定位、缺口與來源分級

### 在 repo 裡的位置

本篇是 [harness §七「Token 成本」](./harness-engineering.md#token-成本)的深潛。那節列了三個手段（KV-cache 前綴穩定、工具精簡、tool-call offloading），沒給任何數字，也沒說這三個手段各自打在成本的哪一項。本篇補一條方程式，把那三條和另外十來條槓桿各自放進一個項裡。

它和 [context-engineering.md](./context-engineering.md) 的分工：那篇回答「這代模型該給它看什麼」，本篇回答「給了之後，每一輪要付多少錢，怎麼少付」。兩篇在 §4.3（工具 schema）交會，那篇說 deferred tool loading 是新 primitive，本篇給它一個價格：100 個工具、每輪 50–70K token。

### 這篇補的缺口

repo 現有筆記碰到成本時都是一筆帶過：

- harness §七 三條 bullet，無數字
- [graph](./graph-engineering.md)、[codez](./codez-graph-engineering.md) 提到「subagent 繼承 session model，所以大 run 整場按 session tier 計價」，只講了風險，沒講該設成什麼
- [boris-cherny-tips](./boris-cherny-tips.md) 記了 Fable 5.1 cache read 降價，是廠商那一側的事

**真正的空白是分解框架。** 沒有方程式，成本優化就是一堆互不相關的技巧；有了方程式，每個技巧都能回答「我在動哪一項、預期壓多少」。§一 補的是這個。§三到§六 是逐項的槓桿，§七 是把 Uber 規模翻譯成個人規模。

### 來源該怎麼看

原文是 Uber 工程部落格在 X 上的長文（2026-08-29），第一方、掛名、附圖表。它和 repo 其他篇的來源性質不同：

| 級別 | 內容 | 處理方式 |
| --- | --- | --- |
| **一級（可對外部一手核對）** | prompt cache 的寫入／讀取倍率、TTL 選項 | 已核對（§九），直接採用並補上 Fable 5.1 的新價 |
| **二級（Uber 自家量測，數字真、環境專屬）** | 7×／9.4×、34%／52%、50–70K schema、code-mode 五筆查詢、38 秒 vs 20 分鐘 | 採用，但**只當量級參考，不當基準**。原文自己也說 "your mileage may vary" |
| **三級（Uber 的判斷與提法）** | 四層分類、「subagent 預設弱模型是最大槓桿」、400K 提前壓縮、Medium effort | 當提法採用。這些是他們在自己 workload 上的最佳解，換 workload 要重量 |

還有一項要標清楚：**原文沒有揭露絕對金額**。所有成本數字都是相對值（較高峰降 X%、較六月降 Y%）。這是合理的商業選擇，但代表你沒辦法拿它算自己的 ROI。

### 適用邊界

- 方程式（§一）與槓桿分類不分規模，個人也適用。
- §二 的四層與 §六 的 tier 制度是**組織**工具。個人只有底下兩層（raw session、session with skills），也不需要 tier。
- §五 的 Context Graph（2,400 萬節點）是 Uber 規模的解。它背後的原則（grounding 減少搜尋 turn）可搬，實作不可搬。§七 有個人版翻譯。
- **本篇的數字有保鮮期**，以模型世代與廠商定價為單位。cache read 倍率在本篇寫作時已經因 Fable 5.1 變了一次（§4.2）。方程式本身不過期。

---

## 一、成本方程式：六項相乘

### 1.1 公式與三組分工

Uber 把一個 agent session 的總支出拆成六項相乘：

```
總支出 = Users
       × Sessions / User
       × Turns / Session
       × Requests / Turn
       × Tokens / Request
       × Price / Token
```

六項分三組，方向不同：

| 組 | 項 | 方向 | 意義 |
| --- | --- | --- | --- |
| 採用與參與 | Users、Sessions/User | **要長** | 這兩項變大是好事。不管是人在互動，還是 agent 替人跑 |
| agent 軌跡 | Turns/Session、Requests/Turn、Tokens/Request | **要縮** | agent 為了完成你那一個請求，自己額外做的事：規劃、搜尋、重試、每輪重送的 context |
| 單價 | Price/Token | **要縮** | 廠商定價你動不了，但你決定哪個 workload 跑哪個模型 |

**中間三項是重點。** 這三項是「agent 替自己花的錢」，跟你要它做的事無關。原文說絕大部分優化力氣都花在這裡：幫 agent 更快規劃、減少不必要的 turn 或錯誤、縮小每輪輸入。

一個容易漏掉的讀法：**Turns 和 Requests 是兩個不同的項。** 一個 turn 是一次人（或上游 agent）的請求；一個 request 是一次模型呼叫。一個 turn 裡模型呼叫十次工具就是十個 request，每個 request 都重送整份 context。所以 §五 的「讓 agent 少找」和 §四 的「每輪少送」是兩個獨立槓桿，可以各自量。

### 1.2 五層量測

原文給了一張他們每週／每月追蹤的指標表。值得整張留下，因為它回答的不是「花了多少」，而是「為什麼變了」：

| 層 | 指標 | 回答什麼 |
| --- | --- | --- |
| Portfolio | 總歸因成本、去重使用者數、每個工具／agent 的成本、使用者與支出占比 | 錢流去哪裡、哪個工具在動 |
| Unit economics（每工具） | 每人成本、每人請求數、每千請求成本、每請求的 input／output／總 token、每百萬 token 成本、每千 session 成本、每活躍 session 小時成本、**prompt cache 命中率** | 工具是真的變便宜，還是用量在挪 |
| Model economics | 每個模型的成本與占比、請求數與占比、每千請求成本、每百萬 token 成本 | 哪一次模型發布真的改變了帳單 |
| Driver decomposition | 把成本變動**依序**分解成：採用（使用者）→ 參與（每人請求）→ 輸入工作量（每請求 input token）→ 輸出工作量（每請求 output token） | 數字為什麼動，逐項說清楚，不留無法解釋的殘差 |
| Managed agent outcomes | 每個 managed agent 的**結果計價成本**（每個合併 PR、每次 review、每則告警、每次清理）、品質訊號（revert 率、F1、MTTR）、產量（diff 數、review 數、告警處理數） | 每個 agent 交付每單位價值是不是越來越便宜、換模型後品質有沒有守住 |

兩個讀法：

- **Driver decomposition 是方程式的執行版。** 它把 §1.1 的六項變成一個有順序的歸因流程，強迫你把總變動填滿，不准剩一塊「其他」。
- **最底下那層才是目標。** 前四層都在量 token，最後一層量的是 PR、review、告警。原文的立場是把 workload 往上推到 managed agent，正是因為只有在那一層，成本才能用「每單位價值」計價（§二）。

### 1.3 固定模型才量得到自己的功勞

原文有一段方法論值得單獨記：adoption、workload 組成、模型升級三個都在同時變，**你的優化成果會被淹沒在裡面**。他們的做法是固定一個模型，只在那個模型上量二月到七月：每千請求成本較高峰降 34%，每 session 成本較六月高峰降 52%。

這條跟 [eval-engineering §5.1「釘住判官」](./eval-engineering.md#51-釘住判官要記什麼什麼時候重跑)是同一個動作：**要量自己做了什麼，先把不是你動的變數釘住。** 對個人來說翻譯成：換模型的那一週，別同時改 CLAUDE.md 或工具設定，否則你不知道帳單是誰動的。

---

## 二、四層場景：越往上越能控

原文把 AI 使用分成四層，從最通用到最專用。層越高，他們對成本、品質、模型選擇的控制越多：

| 層 | 誰起頭、跑在哪 | 例子 | 成本單位 | skill 有沒有持續優化 | 跑在 Uber 的機器上 |
| --- | --- | --- | --- | --- | --- |
| **Raw sessions** | 互動式、你的筆電。你自己給 context | Claude Code、Codex、OpenCode | cost/session | 否 | 否 |
| **Sessions with skills** | 互動式、你的筆電。共用 skill，本機執行 | 3,600+ 個 golden skill | cost/session | 是 | 否 |
| **General agents** | 互動式、Uber 雲端。共用 skill、代管 | Cortana：零設定、可用任何 skill 的通用 agent | cost/query | 是 | 是 |
| **Specialized agents** | agent 驅動＋人在迴路、Uber 雲端 | Minion（intent → PR）、uReview（PR → code review）、Agentic XP（XP 完成 → 讀數）、Conan AI（告警 → RCA）、Fawkes（cron → 維護） | cost/merged PR、cost/review、cost/alert、cost/cleanup | 是 | 是 |

最上層的說明只有一句，但它是整篇的論點：**任務越窄 → 有真 benchmark → 可以跑 Pareto 最優模型 → 每一塊錢都對應到一個工作單位。**

這解釋了原文結論那個「戰略轉向」：從互動式開發流程轉到 managed agent，不是因為 managed agent 比較炫，是因為只有在那一層，你才同時握有模型路由、執行 harness、支出三個控制權。優化幾十個有 benchmark 的專用 agent，比優化幾千個工程師的終端 session 便宜得多。

> 跟 repo 的接點：這四層是 [harness §六「互動式 harness vs 自主 pipeline harness」](./harness-engineering.md#另一條正交軸互動式-harness-vs-自主-pipeline-harness)那條軸的**成本讀法**。那節講兩種 harness 的結構差異，這裡補一句：結構差異直接決定你能不能用「每單位價值」計價。

---

## 三、Price／Token：選模型

廠商定單價，你選哪個 workload 跑哪個模型。Uber 對「選對」的定義是 Pareto 效率，三個維度：**每完成任務的成本、輸出品質、模型可靠度。** 注意第一個維度是 cost per *completed task*，不是 cost per request。便宜的模型多跑三輪才做完，不算便宜。

### 3.1 benchmark 驅動的四步

每個 managed agent 都走同一套：

1. **用 agent 的真實工作建 benchmark。** uReview 的 benchmark 是真實 PR 加已知 bug，分 easy／medium／hard。
2. **用一個能接任何模型的 harness 跑。** frontier 或 open-weight 都走同一個介面。
3. **對 benchmark 量：** precision、recall、F1，加上每次 review 的成本、延遲、timeout、噪音。
4. **移到 Pareto 最優點，然後持續移。** 原文的說法是 frontier 每幾週就會移動。

uReview 的結果：換模型後 F1 上升、每 PR 成本大幅下降。原文的圖是所有測過的配置散在 cost／F1 平面上，虛線是 Pareto 前緣，前緣左下方的每一點都被某個更便宜或更好的配置打敗。

他們另外有一個 Uber SWE Benchmark，用大型 monorepo 裡幾千個真實 PR，跨任務型態跑 frontier 與 open-weight 模型，供所有 SDLC agent 選模型時參考。

> **這一步的 benchmark 從哪來，repo 已經寫過。** [eval §四「從 trace 收割」](./eval-engineering.md#四案例哪裡來從-trace-收割)講的就是「拿真實 run 建題目」。那篇的目的是放行閘門，這裡的目的是選模型，**同一份題目可以兩用**。eval §4.4 的保鮮期警告在這裡同樣成立：模型追上來、題目全滿分，Pareto 前緣就畫不出來了。

### 3.2 兩個預設，subagent 那個最大

互動式介面裡 token 單價是固定的，但你可以決定 token 分配到哪個模型。兩個預設值管這件事：**session 起始模型**與 **subagent 模型**。

原文說 subagent 那個是**最有影響力的槓桿，而且影響還在變大**：會開 subagent 的 session 比例持續上升，因為新模型更會做多 agent 編排。他們的做法：

- subagent 做的是輸入明確、任務清楚的活，多半不需要 frontier 級推理 → **預設給較弱、較便宜的模型**，保留手動覆寫
- 主模型負責拆任務和評結果，subagent 負責執行

repo 對這件事的前置知識在 [graph](./graph-engineering.md) 與 [codez](./codez-graph-engineering.md)：subagent 省略 `model` 時繼承 session model，這是第一方契約。那兩篇推出的後果是「大 run 整場按 session tier 計價」。Uber 這裡給了對策：**別讓它繼承，給它一個預設。**

一個要小心的地方：這條和 [eval §1.4](./eval-engineering.md#14-三條規則與它們在-repo-裡的接點)「判官要跨家族、要夠強」不衝突。Uber 說的弱模型 subagent 是**執行者**，不是**判官**。graph §5.3 那句「裁判是動 model 的唯一例外」在這裡要反過來讀：執行用弱的、裁判用強的，兩個方向都是刻意的。

---

## 四、Tokens／Request：每一輪重送的東西

每一輪都重送完整對話歷史、專案 context、工具結果。任何縮小單次 payload 的手段，效果都會在 session 裡累乘。這一章的五個槓桿，前兩個是設定，後三個是架構。

### 4.1 兩個 fleet 預設：400K 壓縮、Medium effort

Uber 所有互動式 harness 都包在一個統一 wrapper 裡（安裝、設定、認證、成本可見度）。wrapper 帶兩個預設：

- **自動壓縮在 400K token 觸發，即使模型有 1M context。** 理由是在模型表現、cache 爆量、重複輸入成本之間取平衡。他們量到 fleet 級的每請求 input token 有意義地下降。
- **Reasoning effort 預設 Medium。** 輸出 token（含內部推理 token）的單價是輸入的數倍，這個設定直接砍最貴的那一類。原文的判斷是對一大類任務，Medium 是成本與品質的好平衡。

> **對照第一方預設。** Claude Code 的 effort 預設是 `xhigh`（Anthropic claude-api skill 文件，2026-09-12 查）。Uber 選 Medium 是刻意往下調兩級。哪個對取決於 workload：coding 與長程 agentic 任務對 effort 敏感，分類、改寫、chat 型任務不敏感。這是三級來源（Uber 的判斷），換 workload 要自己量。

這兩個預設的本質是 [harness §四 面向 1「compaction vs reset」](./harness-engineering.md#面向-1context-脈絡)那條光譜的**成本讀法**：那節說壓縮時機是模型相依的；這裡補一句，它也是**帳單相依**的。1M 視窗不代表你該用滿它。

### 4.2 Prompt cache TTL 是一道算式

每輪重送整份歷史，所以把前綴 cache 起來，後續讀取只付標準輸入價的 0.1×。但寫入有溢價，而且兩種 TTL 溢價不同：

| 動作 | 倍率（相對標準輸入價） | 備註 |
| --- | --- | --- |
| Cache read | 0.1× | **Fable 5.1 已降到 0.025×**（$0.25/MTok，見 [boris-cherny-tips](./boris-cherny-tips.md)），原文寫作時尚未發布 |
| Cache write，5 分鐘 TTL | 1.25× | 預設 |
| Cache write，1 小時 TTL | 2× | 要顯式指定 |

**該選哪個 TTL，取決於兩輪之間的空檔。** Uber 的觀察是工程師常把互動式 session 晾超過 5 分鐘，cache 過期，下一輪就要全價重建。所以：

- **互動式 session → 1 小時 TTL。** 空檔常在 5–60 分鐘，這是 2× 寫入唯一划算的區間。
- **subagent → 維持 5 分鐘。** 它只做一件短任務，讀取密集、不會晾著。

原文提到 Anthropic 提供 5 分鐘與 1 小時，OpenAI 提供 30 分鐘。**OpenAI 那半是簡化**（見 §九）：實際是 in-memory 5–10 分鐘、extended retention 最長 24 小時。決策邏輯不受影響。

> **Fable 5.1 改了這道算式。** cache read 降到 0.025× 之後，Anthropic 自家文件的建議變成：與其付 2× 寫 1 小時 cache，不如用便宜的 keep-alive 讀取把 5 分鐘 cache 續命。Uber 的結論（互動式用長 TTL）是在 0.1× 的世界推出來的，**換模型要重算**。這正是 §前言 說的保鮮期。

repo 的接點：[harness §七](./harness-engineering.md#token-成本)第一條「KV-cache 優化：穩定前綴、只追加」講的是**讓 cache 命中**；本節講的是**命中之後 TTL 怎麼選**。前者不對，後者沒意義。

### 4.3 MCP schema 稅與三條路

Uber 所有 MCP 呼叫走一個統一 gateway，後面掛 1,000 多個 MCP server（內部加第三方 SaaS）。問題在標準 MCP 的行為：**所有工具 schema 一開 session 就全部載入 context，不管你這場會不會用到。** 100 多個工具就是 50–70K token 的 schema，塞在 initial prompt 裡，每一輪重送。

原文的 Figure 7 把三條路並排：

| 路徑 | session 開始時 context 裡有什麼 | 規模上限 |
| --- | --- | --- |
| **直接安裝 MCP server** | 每個工具的 schema，用不用都在。100+ 工具 ≈ 50–70K token | 加工具就加稅 |
| **CLI 解析** | 近乎零。模型執行一個 shell 指令，CLI 在呼叫當下向 gateway 解析並呼叫工具。1K+ 個 MCP 工具全部投影成 CLI 指令 | gateway 有多少就多少 |
| **Tool search** | 近乎零。模型先搜工具目錄，只載入需要的那幾個 | 幾千個工具，選擇準確度不隨數量退化 |

兩條路互補：CLI 解析把 MCP 從 context 裡拿掉；tool search 讓你在 context 之外還能找到它們。

> 這一節是 [context-engineering §2.3](./context-engineering.md#23-全部前載--漸進揭露)那條「deferred tool loading」的**價格標籤**。那篇說這是 Claude Code 的新 primitive，好處是工具可以往上加而不吃 context。這裡的 50–70K 是「不這樣做」的代價。另外 tool search 不只是 Claude Code 功能，Anthropic API 也有 `tool_search_tool_regex`／`bm25` 加 `defer_loading` 的第一方版本（claude-api skill 文件）。

### 4.4 Code-mode：省的是 overhead，不是 payload

工具變成 shell 指令之後，模型可以把多個動作寫進**一個腳本**。這對「囉嗦」的工具協定特別有用。以資料倉儲查詢為例：

- **標準 MCP 流程**：送查詢 → 輪詢狀態 2–5 次 → 取結果。每一步都是一個模型 turn，每個回應都落進 context。3–7 個 model turn。
- **Code-mode**：一個 Python 迴圈跑在子行程裡，輪詢留在子行程，只有摘要回到 context。1 個 model turn，約 400 token，其中約 300 來自打包好的 skill。

Uber 在同一個 Claude Code session 裡用五個相同的 SQL 查詢量了兩條路：

| 查詢 | LLM tool-use（token） | Code-mode（token） | 節省 |
| --- | --- | --- | --- |
| `SELECT 1`（1 列） | 903 | 402 | 55% |
| `COUNT(*)`（1 列） | 954 | 403 | 58% |
| `GROUP BY LIMIT 20`（20 列） | 1,600 | 457 | 71% |
| `SHOW COLUMNS`（175 列） | 2,200 | 900 | 59% |
| `SELECT *` 寬表（50 列） | 1,431,594 | 900 | ≈100% |

**前三列才是重點。** 結果集小到遠低於回應大小上限，code-mode 仍然省一半以上。省下來的不是資料 payload，是 overhead：schema 初始化、多輪輪詢、每一步的逐步推理。最後一列那個 143 萬 token 是另一個問題（大 payload 灌進 context），code-mode 順便也解了。

批次工作會把效果放大：原本 N 個 model turn 的迴圈變成一個腳本，節省超過 90%。Uber 為最常用的 MCP server 做了 25 個以上的 code-mode skill，讓標準流程預設走最便宜的路。

> 這是 [harness §四 面向 1「tool-call offloading」](./harness-engineering.md#面向-1context-脈絡)的實測版。那節說「大型工具輸出寫檔案，context 只留指針」；這裡多了一層：**輪詢與中間步驟也可以 offload**，而且小輸出時省的比大輸出時還「純」。[graph](./graph-engineering.md) 把這件事在 workflow 層開到極致，見 [context §本 repo 內部連結](./context-engineering.md#本-repo-內部連結)的分界說明。

### 4.5 SaaS MCP：比內部的更胖

第三方 SaaS 的 MCP server 比內部的難管。廠商不知道你要用哪幾個功能，所以把整個產品都塞進去：一個 workspace 套件 49 個工具、約 22K token 的 schema；即時通訊和專案追蹤的廠商分別是 34 和 46 個工具。載兩三個 SaaS server，agent 還沒看到你的 prompt，背的 schema 就比要改的檔案還大。

Uber 的處理跟內部 MCP 一樣：SaaS MCP 也走 gateway、也投影成 CLI、也為每個 server 在 code-mode plugin 裡寫專用 skill 封裝常見流程。

這一節對個人的意義最直接：**你裝的每一個 MCP server 都在收這筆稅**，而且是廠商的產品範圍決定稅率，不是你的使用量。

---

## 五、Requests／Turn：讓 agent 少找

原文這一章的開場句值得留：**沒有 grounding 的 agent 是慢慢失敗，不是便宜地失敗。** 它會一次又一次把越來越大的 context 送出去，再多搜一個地方。

Uber 的解是 AI Context Graph：2,400 萬個節點、8,000 萬條邊，86 種節點型別、117 種邊型別，整合 30 多個內部系統（服務、團隊、事故紀錄、PR、設計文件、部署、資料集、歷史查詢），任何 agent 都能用自然語言查。

對照案例是同一個 prompt、同一個模型：

| | 有 graph | 沒有 graph |
| --- | --- | --- |
| 做法 | 查歷史用量，找到 50 多個分析師在用的那張表 | 翻服務程式碼 |
| 結果 | 38 秒，答對 | 20 分鐘、開了 2 個 subagent、撞了 3 個錯誤，**答錯**（結論該資料集不可查） |

用方程式讀這個案例：沒 graph 那邊不只 Requests/Turn 爆了，Turns、Tokens/Request 也一起爆（subagent、錯誤重試、越積越大的 context）。**Requests/Turn 是六項裡槓桿最長的一項，因為它拉高時會帶著另外兩項一起走。**

原文說在 Uber 的 codebase 規模下，agent 大部分的 turn 花在**找資訊**而不是寫程式。「提前給更多資訊」是壓這一項最有力的槓桿。

> 個人版的 grounding 不是 graph，是 [context-engineering §三](./context-engineering.md#三分層配置哪一層該放什麼)講的那套：CLAUDE.md 只放 Claude 自己推不出來的 gotcha，細節做成可按需載入的 skill。目的相同：**讓 agent 第一次就查對地方**。那篇從品質角度說這件事，這裡補上它的價格。

---

## 六、可見度：讓人和 agent 都看得到帳單

Uber 沒有設硬上限，用的是可見度加回饋迴圈。

**Status line 即時計數器。** harness 的狀態列顯示這場 session 的即時花費，也顯示跨所有 harness 的個人總額。

**Spend tier 與 nudge：**

- 一個共用 tier 涵蓋所有互動式 harness（不是每個工具各一份預算）；managed agent 另外分 tier
- Slack 在預期支出的 50／80／100% 提醒，讓工程師有時間規劃
- tier 升級走主管簽核，快速生效
- 一個 cost check skill 隨時可看成本拆解，status line 上有即時教練提示

原文的立場是讓工程師**自己評估任務 ROI**，同時擋住失控支出。

**Session 分析 dashboard。** status line 只給總額，不說錢花在哪、該改什麼。這個 dashboard 內建在 runtime 裡，零設定，一個 skill 跑下去就掃使用者在本機與雲端 sandbox、跨所有 harness 的 session trace。它不給一個總分，而是標記 **16 種反模式**，每種附上金額影響與具體修法。原文列了四種，我把它們對回方程式：

| 反模式 | 動到方程式哪一項 |
| --- | --- |
| 模型路由不當：簡單的多輪 session 跑 Opus，Sonnet 就夠 | Price/Token（§三） |
| context 膨脹：40KB 的 MCP 回應留在 context 裡，之後每輪重複計費 | Tokens/Request（§4.4） |
| cache 過期：長時間中斷後恢復 session，過期的 cache 逼你全價重建前綴 | Tokens/Request（§4.2） |
| 初始化開銷：使用者還沒開口，就先載了 10 萬 token 的系統指令與工具定義 | Tokens/Request（§4.3、4.5） |

> repo 的接點是 [harness §五 5.5](./harness-engineering.md#55-verification-驗證--hooks--session-jsonl--statusline)的 observability 半邊：Statusline 顯示 cost、Session JSONL 留軌跡。那節說「小專案 Statusline 加偶爾翻 JSONL 就夠」。Uber 這個 dashboard 就是「翻 JSONL」的自動化版：**把反模式寫成規則，對著 trace 跑。** 這也是 [eval §四](./eval-engineering.md#四案例哪裡來從-trace-收割)「從 trace 收割」的成本版，收的不是失敗案例，是浪費模式。

---

## 七、個人與小團隊可以抄什麼

> **這一節是本篇自己加的翻譯，原文沒有。** 分類依據是「需不需要 Uber 那種基礎設施」。

### 直接抄（不需要任何基礎設施）

- **subagent 用便宜模型。** 主模型拆任務、評結果；subagent 執行。判官例外，判官要強、要跨家族（§3.2）。
- **互動式 session 的 cache TTL 對照你的空檔。** 常晾超過 5 分鐘就用 1 小時；Fable 5.1 上先看 keep-alive 是不是更便宜（§4.2）。
- **少裝 MCP server，尤其是 SaaS 的。** 每個都在收 schema 稅。能用 CLI 或 deferred loading 就別直接掛（§4.3、4.5）。
- **囉嗦的工具流程寫成腳本。** 輪詢、批次、多步查詢包進一個 Bash 或 Python，只把摘要交回 context（§4.4）。
- **換模型的那一週別動別的。** 否則量不出是誰改變了帳單（§1.3）。
- **effort 依 workload 調，不要一個值走天下。** coding 與長程任務用高的，改寫、分類、問答用低的（§4.1）。

### 需要自己建一點東西

- **一份小 benchmark 選模型。** 不用幾千個 PR，[eval §4.4](./eval-engineering.md#44-套件規模與保鮮期)說的 100–200 題起步就夠畫出 Pareto 前緣的形狀（§3.1）。
- **一個看 session 反模式的腳本。** 對著 Session JSONL 掃 §六 那四種就有用：哪幾場 session 用了 Opus 卻只有三輪、哪幾場 context 裡有 40KB 的工具回應（§六）。
- **把 gotcha 寫進 CLAUDE.md、細節做成 skill。** 這是個人版 grounding（§五）。

### 不適用或不必

- 四層場景與 spend tier（§二、§六）：組織工具。
- Context Graph（§五）：規模解。原則可搬，實作不可搬。
- 400K 提前壓縮（§4.1）：個人 session 很少真的碰到；碰到時，先問為什麼一場 session 會這麼長，再問要不要提前壓縮。

---

## 八、與本 repo 其他筆記的對照

| 筆記與段落 | 關係 |
| --- | --- |
| [harness §七 Token 成本](./harness-engineering.md#token-成本) | **母節點**，本篇是其深潛。那節三條手段各自對應：KV-cache → §4.2 的前提、工具精簡 → §4.3／4.5、offloading → §4.4 |
| [harness §四 面向 1](./harness-engineering.md#面向-1context-脈絡) | compaction vs reset 光譜的成本讀法（§4.1）；tool-call offloading 的實測（§4.4） |
| [harness §五 5.5](./harness-engineering.md#55-verification-驗證--hooks--session-jsonl--statusline) | Statusline／JSONL 是 §六 dashboard 的個人版原料 |
| [harness §六 互動式 vs 自主 pipeline](./harness-engineering.md#另一條正交軸互動式-harness-vs-自主-pipeline-harness) | §二 四層是這條軸的成本讀法：只有自主層能用每單位價值計價 |
| [context §2.3 漸進揭露](./context-engineering.md#23-全部前載--漸進揭露) | **最直接的接縫**。那篇說 deferred tool loading 是新 primitive，§4.3 給它價格（50–70K）並補上 CLI 解析這條第三路 |
| [context §三 分層配置](./context-engineering.md#三分層配置哪一層該放什麼) | §五 的個人版 grounding。那篇講品質，本篇講價格 |
| [eval §四 從 trace 收割](./eval-engineering.md#四案例哪裡來從-trace-收割) | §3.1 選模型的 benchmark 與那篇的放行題目**是同一份**；§六 的反模式 dashboard 是同一手法收浪費而不是收失敗 |
| [eval §1.4 三條規則](./eval-engineering.md#14-三條規則與它們在-repo-裡的接點) | **方向相反但不衝突**：§3.2 執行者用弱模型，判官仍要強、要跨家族 |
| [eval §5.1 釘住判官](./eval-engineering.md#51-釘住判官要記什麼什麼時候重跑) | §1.3「固定模型才量得到自己的功勞」是同一個動作 |
| [graph](./graph-engineering.md)、[codez](./codez-graph-engineering.md) | subagent 繼承 session model 的契約是 §3.2 的前提；那兩篇講風險，本篇給對策 |
| [boris-cherny-tips](./boris-cherny-tips.md) Fable 5.1 | cache read 降到 $0.25/MTok 是 §4.2 算式改變的原因 |
| [ihower §八 harness 會過期](./ihower-harness-engineering.md#八收尾model-harness-fit-與會過期的-harness第-8-篇) | 本篇保鮮期依據：方程式不過期，倍率與門檻以模型世代和廠商定價為單位過期 |

一句話定位：harness 回答「模型在什麼環境做事」，context 回答「該給它看什麼」，**本篇回答「這些選擇每一輪要付多少錢，哪幾項是你能動的」**。

---

## 九、來源與更新

### 主要來源

- **Uber Engineering**（@UberEng，post author @udaykiran）— _"Running a Software Factory Efficiently at Uber Scale"_（2026-08-29）— [x.com/UberEng/status/2093444169037762840](https://x.com/UberEng/status/2093444169037762840)
  - 第一方、掛名、附圖表。**所有成本數字都是相對值，無絕對金額。** 原文明說定價與廠商數據來自公開資料，成本降幅是 Uber 環境專屬
  - 文中提到的 AI Engineer 2026 conference 演講與 Uber SWE Benchmark 未另外查證
  - 2026-09-12 以 claude-in-chrome 讀取；圖表（四層、方程式、量測表、槓桿表、code-mode 對照表、Figure 7）以截圖判讀轉錄，非原文文字

### 一手查證紀錄（2026-09-12）

| 原文宣稱 | 一手來源 | 結果 |
| --- | --- | --- |
| cache read 0.1×、5 分鐘寫入 1.25×、1 小時寫入 2× | Anthropic claude-api skill 隨附 `shared/prompt-caching.md` | ✅ 相符。**補充：Fable 5.1 的 read 已降到 0.025×**（$0.25/MTok），且同一份文件據此建議 Fable 5.1 上 keep-alive 常比 1 小時 TTL 便宜。原文寫作時 Fable 5.1 尚未發布 |
| Anthropic TTL 選項 5 分鐘與 1 小時 | 同上 | ✅ 相符 |
| OpenAI 提供 30 分鐘 TTL | OpenAI prompt caching 文件（WebSearch） | ⚠️ **簡化**。實際為 in-memory 5–10 分鐘（最長 1 小時）、extended retention 最長 24 小時；GPT-5.5 起 24h 為唯一模式。「30 分鐘」接近 extended retention 的典型值。決策邏輯不受影響 |
| Claude Code effort 預設（本篇 §4.1 對照用，非原文宣稱） | claude-api skill 主文件 | ✅ 預設 `xhigh`。Uber 的 Medium 是往下調 |
| API 層 tool search 存在（本篇 §4.3 補充，非原文宣稱） | claude-api skill 主文件 | ✅ `tool_search_tool_regex_20251119`／`bm25` 加 `defer_loading` |
| 7×／9.4×／34%／52%、50–70K、code-mode 五筆、38 秒 vs 20 分鐘 | — | 🟡 Uber 自家量測，**不可重跑**。當量級參考 |
| 四層分類、subagent 弱模型是最大槓桿、400K 壓縮、Medium effort、16 種反模式 | — | 🟡 Uber 的判斷與提法。三級來源 |

> **方法。** 定價倍率直接對 skill 隨附的 Anthropic 文件逐字核對；OpenAI 部分只用 WebSearch 摘要，未讀原始頁面，所以只標到「簡化」而非「錯誤」。Uber 內部數字沒有外部核對途徑，全部標 🟡。

### 本 repo 內部連結

- [harness-engineering.md](./harness-engineering.md) — 本篇是其 §七 Token 成本的深潛；§四 面向 1、§五 5.5、§六 皆有接點（見 §八）
- [context-engineering.md](./context-engineering.md) — §2.3 deferred tool loading 的價格；§三 分層配置是 §五 的個人版
- [eval-engineering.md](./eval-engineering.md) — 選模型的 benchmark 與放行題目同源；判官強、執行者弱的分工
- [graph-engineering.md](./graph-engineering.md)、[codez-graph-engineering.md](./codez-graph-engineering.md) — subagent 繼承 model 的契約
- [boris-cherny-tips.md](./boris-cherny-tips.md) — Fable 5.1 cache read 降價
- [ihower-harness-engineering.md](./ihower-harness-engineering.md) — 保鮮期分界

### 校對紀錄

- **2026-09-12**：初版。依 Uber 2026-08-29 長文整理，重編為「方程式 → 四層 → 逐項槓桿 → 可見度 → 個人翻譯」骨架；定位為 harness §七 Token 成本的深潛。原文沒有、本篇自行加入並已就地標示的內容：§1.1 Turns 與 Requests 是兩個項的讀法、§3.2 與 eval 判官規則的方向對照、§4.1 與 Claude Code effort 預設的對照、§4.2 Fable 5.1 改變算式的註記、§六 反模式對回方程式的表、§七 整節

下次 review 觸發點：Anthropic 或 OpenAI 改 cache 定價或 TTL、Claude Code 改 compaction 或 effort 預設、Uber 發後續文（原文 What's Next 列了動態模型路由與即時 session 分析）、harness §七 重寫、有人在個人規模量出 code-mode 或 TTL 的實測數字。
