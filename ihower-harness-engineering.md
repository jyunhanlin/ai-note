---
title: 駕馭工程（ihower 系列）
tags: [agent, harness, ai-engineering, feedback-loop, agent-development, ihower]
created: 2026-07-07
last_reviewed: 2026-07-07
type: reference
status: living-document
sources:
  - ihower，「給 Agent 開發者的駕馭工程」系列共 9 篇（2026-06-26，演講書面版，aihao 小編整理）— blog.aihao.tw
  - ihower 演講投影片《給 Agent 開發者的 Harness+Loop Engineering》— https://ihower.tw/presentation/harness.html
  - 本 repo：harness-engineering.md（使用者視角：單一 agent 運行環境的五面向）
  - 本 repo：loop-engineering.md（harness 上一層的自走 loop）
---

# 駕馭工程：給 Agent 開發者的 Harness Engineering（ihower 系列）

> 開發者視角的 harness 筆記。本 repo 的 [harness-engineering.md](./harness-engineering.md) 站在「Claude Code 使用者」這端（把現成 harness 調成自己的）；這個系列站在「自行開發 AI Agent 的工程師」這端（Text-to-SQL、RAG、訪談 agent……未必是 coding）——Claude Code 與 Codex 在系列裡是**被解剖的參考實作**：作者讀 Codex 開源碼、用 mitmproxy 攔 Claude Code 封包、引用兩家 system prompt，目的是抽出可移植的 pattern，回頭建自己的 agent。

## 目錄

- [前言：系列定位與一句話主軸](#前言系列定位與一句話主軸)
- [一、基礎：六項 Deep Agent 內建能力（第 1 篇）](#一基礎六項-deep-agent-內建能力第-1-篇)
- [二、核心：回饋迴路，不是完美提示（第 2 篇）](#二核心回饋迴路不是完美提示第-2-篇)
- [三、時機一：工具回傳值是寫給 agent 的回饋（第 3 篇）](#三時機一工具回傳值是寫給-agent-的回饋第-3-篇)
- [四、時機二：Mid-run Injection（第 4 篇）](#四時機二mid-run-injection第-4-篇)
- [五、時機三：單輪驗收，Goal 與 Outcomes（第 5 篇）](#五時機三單輪驗收goal-與-outcomes第-5-篇)
- [六、時機四：外層 Loop（第 6 篇）](#六時機四外層-loop第-6-篇)
- [七、進階：自我改進 Harness（第 7 篇）](#七進階自我改進-harness第-7-篇)
- [八、收尾：Model-Harness Fit 與會過期的 Harness（第 8 篇）](#八收尾model-harness-fit-與會過期的-harness第-8-篇)
- [九、實作：自建 Agent 的框架選型（第 9 篇）](#九實作自建-agent-的框架選型第-9-篇)
- [十、與本 repo 筆記的對照](#十與本-repo-筆記的對照)
- [十一、來源與更新](#十一來源與更新)

### 閱讀路徑建議

- **第一次讀**：前言 → §二（工作定義與四時機骨架）→ §五（全系列最深的一篇）
- **正在自建 agent**：§三 → §五 → §九（工具層 → 驗收 → 選型）
- **已讀過本 repo harness 筆記、想知道差在哪**：直接看 §十

---

## 前言：系列定位與一句話主軸

系列作者開宗明義劃了界線：網路上多數談 harness engineering 的內容是「怎麼把 Claude Code、Codex 用得更好」的使用技巧，**這個系列不是這個角度**——它站在「自行開發 AI Agent 的工程師」這邊，主張 harness engineering 是一套適用於任何場景 AI Agent 的廣義工程方法。

全系列反覆強調的一句話主軸：

> **Agent 需要的是回饋迴路，不是完美的提示（Agents need feedback loops, not perfect prompts）**——再怎麼改 prompt，沒有迴路它就收斂不了。

系列骨架（9 篇的結構）：

| 篇   | 主題                                 | 角色                              |
| ---- | ------------------------------------ | --------------------------------- |
| 1    | 六項 Deep Agent 內建能力             | 基礎：解決「能不能做」            |
| 2    | 什麼是 Harness Engineering           | 核心：定義 + 回饋四時機骨架       |
| 3–6  | 回饋時機①②③④                         | 主體：由內而外四個回饋注入點      |
| 7    | 自我改進 harness                     | 進階：讓 agent 自己改 harness     |
| 8    | Model-Harness Fit 與 harness 的過期  | 收尾：補上「時間」維度            |
| 9    | 框架選型                             | 實作：動手自建時的第一個決定      |

---

## 一、基礎：六項 Deep Agent 內建能力（第 1 篇）

所有 agent 的共同底層一句話：**LLM 在迴圈裡使用工具**——作者引 Anthropic 對自家 runtime 的形容「笨迴圈 (dumb loop)」：智能全在模型裡，迴圈只管每一輪要做什麼。過去較「淺」的 agent 做 5–15 步沒問題，碰到 500 步、跨多天的任務就有三種失控：工具輸出塞滿 context 把指令擠出視窗、在中間步驟雜訊裡忘了目標、走錯方向不會回頭只能將錯就錯。

Deep Agent（Claude Code、Codex 這類，有人稱 Agent 2.0）的解法是內建六項能力。技術上幾乎都是同一招：用 Function Calling 定義工具丟給模型呼叫。

| 能力                  | 解決什麼                       | 關鍵洞見                                                                                             |
| --------------------- | ------------------------------ | ---------------------------------------------------------------------------------------------------- |
| ① Plan & Todos        | 顯式待辦清單取代隱式 CoT 規劃  | Claude Code 的 Todo 工具其實是 **no-op**（引 LangChain）——純粹是 context engineering，防偏離用       |
| ② Filesystem & Bash   | 真的動手做事                   | filesystem 一次解鎖：工作區、context 卸載、跨 session 持久化、多 agent 共享、git 免費回溯；bash 是通用工具——模型能自己寫 code 補缺的能力 |
| ③ Sub-Agent           | context 隔離 + 平行            | 子代理耗數萬 token 搜尋試錯，只回一兩千 token 摘要；spawn_agent 從主迴圈看就是普通 tool call         |
| ④ Memory              | 跨 session 記得你和專案        | 運作在 context 層而非權重層；**記憶是提示 (hint) 不是事實**——行動前對照真實狀態再驗證                |
| ⑤ Skills              | 按需載入的技能包               | 漸進式揭露 (progressive disclosure)：平常只給精簡列表，用到才展開——對抗 context rot                  |
| ⑥ 更多工具            | 接上外部世界                   | MCP（標準化工具協議）、Browser、Computer Use；嚴格說不算 Deep Agent 必要條件                          |

兩個對自建者的關鍵提醒：

- **這是可選配的能力清單，不是必裝套餐**。六項是「通用 coding agent」才需要的完整清單；自建特定用途 agent 按任務取捨——任務單純就免 Plan & Todos，沒有檔案與跨 session 需求就免 Filesystem 與 sandbox。
- **能力只解決「能不能做」，沒回答「做得對不對、做完了沒」**。缺的不是更多工具，而是把能力接成可驗收、可修正、可持續運行的回饋系統——這套工程就是 harness。這條「能力面 vs 品質面」的分界貫穿全系列，第 9 篇還會用它收尾。

---

## 二、核心：回饋迴路，不是完美提示（第 2 篇）

### 工作定義

作者批評 LangChain 那句被大量引用的定義（「模型提供智能，harness 是讓那份智能變得有用的系統」／「如果你不是模型，那你就是 harness」）：它劃了邊界，卻沒告訴你 harness 內部該怎麼想、該往哪用力。他給出全系列的可操作定義：

> **Harness Engineering：讓 agent 根據目標，持續、正確地動作的工程。**
>
> 核心材料是「回饋訊號」——你得有東西能判斷「做得對不對、做完了沒」；沒有這個訊號，一切都是無回饋迴圈 (open loop)。

三層工程是疊加不是取代：Prompt Engineering 管「這一次怎麼回答得更好」、Context Engineering 管「該放什麼資訊進 context」、Harness Engineering 管「agent 在行動迴圈裡怎麼被約束、檢查、修正」。

### 病症與心法：premature completion 與 ratchet

長時間執行 agent 的失敗模式第一名（引 Anthropic）：**過早宣告完成 (premature completion)**——自信宣稱「已全部完成 ✅」但測試沒跑、需求漏一半，指出後道歉再犯。對策是 **ratchet（棘輪）**心法：每次出錯就做一個工程解法讓它再也不犯（不知道慣例→寫進 AGENTS.md；跑破壞性指令→hook 擋掉；長任務迷路→拆 planner/executor；交出跑不過的 code→強制 typecheck）。棘輪是**雙向**的：只在看到真實失敗時加約束，也只在模型強到讓約束多餘時拆掉——「一份好的 AGENTS.md，每一行都該追得回一個具體出過的包」。

> 這與 [harness-engineering.md](./harness-engineering.md) 心法 1 是同一條（同源 Hashimoto / HumanLayer）；「拆約束」那半邊在第 8 篇展開成 harness 的過期性。

### 兩軸座標：Böckeler 的 2×2

作者認為最好用的座標系（引 Thoughtworks Birgitta Böckeler）：**方向**（前饋 guides：行動前引導，提高一次做對的機率 vs 回饋 sensors：行動後感測，逼自我修正）×**執行型態**（運算式：確定性、毫秒級 vs 推論式：LLM 語意判斷，慢貴不穩）。同一個「要驗證」的願望落在不同格子強度天差地別：寫在 AGENTS.md 裡求模型照做是前饋、機率性的；用 hook 強制跑、不過不准收工是回饋、確定性的。**前饋是機率裝置，不是保證**——AGENTS.md 寫一百遍「一定要驗證」模型還是可能跳過。

作者的擴展主張：Böckeler 原文全站在 coding agent 場景，但框架本身跟 coding 無關——data agent、RAG 問答、訪談 agent 同樣可以問「我的前饋引導在哪、回饋感測在哪、哪些用程式判定就好、哪些非得用 LLM」。

基本策略只有一件事被各家反覆明講：**先 generate，再 verify**。模型有自我修正能力，但「沒有自發進入『寫完就驗證』這個迴圈的傾向」（引 LangChain）——你不逼它，它不會主動做。佐證 harness 本身就是表現關鍵：LangChain 固定 gpt-5.2-codex 不換模型、只調 harness，Terminal Bench 2.0 從 52.8% 拉到 66.5%。

控制論類比（引瓦特離心調速器）：1780 年代調速器發明前，工人站在蒸汽機旁手調汽門；發明後工作變成**設計調速器**。Harness engineering 同理——感測（測試、judge、review）＋致動（agent 自我修正、重試），工程師從盯著 agent 一步步做，變成設計環境與回饋迴圈。

### 系列骨架：回饋的四個時機

把 agent 的迴圈由內而外攤開，四個下手時機，**越往外越貴**（①③④由 harness 自動觸發，②是注入點）：

| 時機              | 觸發              | 成本／延遲    | 修正粒度       | 對應 hook 層          | 詳見 |
| ----------------- | ----------------- | ------------- | -------------- | --------------------- | ---- |
| ① 工具執行內      | 每次 tool call    | 毫秒級，最便宜 | 單一動作       | Pre/PostToolUse       | §三  |
| ② request 之間注入 | 人或程式想注入時  | 趨近零        | 當前這一輪方向 | 無專屬 hook           | §四  |
| ③ 單輪結束        | 每一輪            | 秒級          | 整輪產出       | Stop hook             | §五  |
| ④ 外層 Loop       | 每個 session      | 分鐘到小時    | 整個任務       | 排程／外迴圈          | §六  |

名詞界定：一輪 (Turn)＝「模型 request → tool call」反覆多次直到模型輸出答案；③沒過就用同一份 context 再跑一輪；④通過後 session 收掉、換全新 context 再跑一圈。

---

## 三、時機一：工具回傳值是寫給 agent 的回饋（第 3 篇）

最內層、最便宜、頻率最高的回饋點——失誤在 tool call 內部被擋住，就不會擴散成整輪甚至整個任務的大代價。核心觀念翻轉：

> **工具輸出不是寫程式的 function output，它是你寫給 agent 看的回饋。**回傳值除了資料，還要夾帶導向、後設資料、下一步指引——「tool response 本身就是一種 prompt engineering」（引 Jason Liu）。

### Tool call 三段式與四個設計原則

三段式：**執行前驗證**（確定性檢查擋危險呼叫，絕對防線）→ **執行後檢查**（結果品質把關、就地修復重試）→ **回傳值設計**（最容易被忽略的一層）。收斂成四原則：

1. **能確定就不用 LLM**——SQL 驗證用 SQLGlot parse AST（類型只允許 SELECT、表名白名單、LIMIT 補強、權限注入），純程式邏輯、零延遲、可重現，不用 judge 來猜
2. **語意判斷才用 Judge**——沒有 assert 可寫的細緻品質才交給 LLM judge；確定性 grader 只能附預先寫死的固定引導字串，judge 才能附當場生成的 reasoning
3. **回饋必須可行動**——爛錯誤訊息是機器級 error code（`ERROR: column "revenue_growth" does not exist`，agent 只能瞎猜），好錯誤訊息附選項與方向（「可用欄位: revenue, revenue_yoy…年增率請改用 revenue_yoy」）。「每個失敗都是你引導模型的免費機會，大多數人都浪費掉了」
4. **延遲預算內完成**——工具層回饋發生在 tool call 內，別讓把關拖慢主迴圈

**成功也要設計**：只回 `{"success": true}` 可能隱藏假完成——本應只改一位使用者、WHERE 寫鬆掃了全表，從旗標完全看不出來。要回**完整狀態**：影響筆數、動過哪些欄位／區塊、規模量化，agent 才有依據判斷合理性。成功但 0 筆結果也要提示可能原因（名稱對不上、時間範圍外），別讓 agent 把空結果當最終答案。

### 回傳值的另一半責任：context 控制

引 Arize 對四個 harness（Pi、OpenClaw、Claude Code、Letta）的逆向分析，四種趨同策略：**硬性上限**（Claude Code 讀檔前 stat 擋 256KB、讀後 token 把關）、**頭尾保留**（截斷捨中間——尾巴常有錯誤訊息與收尾關鍵字）、**卸載到磁碟**（context 只留 2KB 預覽＋路徑）、**續讀提示**（「顯示第 1–2000 行，用 offset=2001 繼續」）。大結果截斷時附整體統計並導向正確做法：「共 3,847 筆已截斷，涵蓋 412 家公司；要加總請直接用 SQL 聚合，不要逐筆讀回來自己算」——控 context、告知規模、導正做法一石三鳥。

RAG 場景的進階版是 **facets**（引 Jason Liu）：回傳不只 top-k，還夾帶資料集分佈統計（如 status 計數）＋ system-instruction——讓 agent 看見「top-3 全是 Done、但還有 5 筆 Open 沒進榜」，修正相似度搜尋偏誤；agent 本來就會多輪探索，不必一次完美 recall。

### 第二意見模型與 Advisor 困境

較貴但更強：工具內部呼叫**另一個訓練不同的模型**要第二意見（「一個模型審查自己的產出不算獨立檢查——就算換新 context，仍共享同一套訓練、先驗、失誤模式」，引 consult-llm）。實例：Amp（Sourcegraph）主 agent 跑 Sonnet、oracle 工具接 o3 專做審查；Claude Code 裝 Codex plugin 後的 `/codex:review`（Anthropic 模型呼叫 OpenAI 模型挑自己毛病）。

觸發時機的困境：想用便宜小模型當主力、卡關才呼叫貴模型當顧問（Advisor Strategy，目標「Opus 智能、Haiku 價格」），但**「知道自己搞不定」跟「有能力解掉」需要同一種能力**——越弱的模型越判斷不出自己卡住了，循環依賴。實務上先用「使用者觸發」或「寫死規則」（連續兩次測試紅、要動核心模組）定死觸發時機。

結構限制收尾：**單步正確，不等於整體完成**——十次 tool call 全驗過，整輪產出仍可能沒達標。這是時機③的事。

---

## 四、時機二：Mid-run Injection（第 4 篇）

Agent 還在一輪內反覆呼叫工具、尚未輸出最終答案時，唯一能把新訊息插進去的位置：**兩次 model request 之間**（tool calls 結果收齊 → 下一次 model request 之前）。

為什麼只有這裡：OpenAI 與 Anthropic API 的共同硬規則——**模型發出的每個 tool call 都必須先補上對應結果，中間不能放別的訊息**，否則下一個 request 直接 400。模型一次發 A、B 兩個 tool call，只補了 output A 就插 user message 會炸；補齊 output B 再插就合法。所以 harness 的做法是等工具結果全部收回、組成 API 合法狀態，才注入——「那是迴圈裡唯一一個對 API 合法、又還沒結束的位置」。

兩類來源、同一機制：

- **人中途 steer**：與 human-in-the-loop 相反——後者是 agent 跑到預先設計的等待點停下來問你，steering 是你在 agent 無意停下時主動介入。兩種強度：🧭 **steering**（不中斷：訊息排進佇列、下一次 model request 前注入，context 全保留）與 🛑 **interrupt**（中斷：取消執行中的工具，harness 替沒答完的 tool call 補一則寫死的 `aborted` tool_result 維持 API 合法，停下來等新指令）。Codex 於 2026-02 正式加入 mid-turn steering；Claude Code 也支援執行中輸入。
- **程式注入**：背景工具結果、外部事件（webhook、告警）、後續脈絡。「人 steer 只是『外部事件引導 agent 計畫』的一種，人就是那個外部事件。」背景工具用**兩步模式**：tool call 立即回一個佔位 tool_result（「工具已在背景執行，完成會自動送回，先做別的」）維持配對合法；背景完成後改用 **user 角色訊息**注入結果（不能再回 tool_result，已經回過了）。

框架支援現況：多數框架（OpenAI Agents SDK、Google ADK）**沒有**暴露這個注入點；少數例外是 Pydantic AI 的 `enqueue()`（v1.101.0+，`asap`／`when_idle` 兩模式），把人 steer 和程式注入當同一件事處理。

一個反思提醒（引 AgentPatterns）：一直需要 steer，常代表初始 prompt 不夠清楚——**手動修正是補救不是常態**，該補的往往是前饋。前饋（一次做對）跟回饋（出錯再修）要一起設計。

---

## 五、時機三：單輪驗收，Goal 與 Outcomes（第 5 篇）

全系列最深的一篇。時機③是 agent 想完、工具呼叫完、**準備收工那一刻**的攔截點（對應 Stop hook 層）：

> 不能因為「模型覺得自己大概做完了」就算數——完成與否，要對著一個**可驗證的停止條件**去驗證，沒過就繼續做。

### Goal：一份精簡契約

好的 Goal 不是把 prompt 寫長，而是講清楚三件事的契約：**終態**（做完長什麼樣）＋**證據**（用什麼驗證）＋**限制**（不能弄壞什麼）。OpenAI Cookbook 模板：`/goal <期望終態> verified by <證據> while preserving <要保住什麼>`。範例：「把 p95 latency 降到 120ms 以下，verified by 壓測報告，while preserving 正確性測試全綠」——降到 135ms 不算完成，破 120ms 但測試掛了也不算。

兩類任務不適合直接套：完成條件模糊的（「把這個變更好」）——套 Goal 只是把含糊換個地方放，「定義不出終態，代表問題不在工具，在你還沒想清楚要什麼」；驗證非二元的（重現論文）——先定義可信度分級（確定／近似／做不到），拿分層審計報告而非一句「成功」。

### 裁判獨立性光譜：三種產品化實作

Codex Goals、Claude Code /goal、Managed Agents Outcomes 是同一概念（durable objective + self-correction loop primitive）的三種實作，差別在**裁判有多獨立於做事的模型**：

|                  | Codex Goals（自我審計）                          | Claude Code /goal（transcript 裁判）                    | Managed Agents Outcome（獨立 grader）              |
| ---------------- | ------------------------------------------------ | ------------------------------------------------------- | -------------------------------------------------- |
| 誰來判定         | 主模型本人（呼叫 `update_goal` 就算數，無人覆核） | 獨立小模型（預設 Haiku）                                 | 全新 context 的 grader agent                       |
| 看什麼證據       | 自己 context 裡的一切（含完整推理）              | 刪減版 transcript（剝除 thinking；「證據沒印在 transcript 上，對裁判等於不存在」） | 只看 artifact——拿 rubric **實際操作**（Playwright 點 UI、打 API、查 DB） |
| 沒過時的回饋     | 沒有診斷，重播同一份 continuation.md             | Haiku 寫的針對性診斷（指出缺什麼）注入主 thread          | 逐條列缺失，最具體                                  |
| 防呆機制         | tool 合約寫滿 only when/do not；continuation.md 的 Completion audit（「must prove completion, not merely fail to find obvious remaining work」） | structured output `{ok, reason, impossible}`——impossible 欄防目標自相矛盾時無限重試 | max_iterations 預設 3、最多 20                      |
| 成本／延遲       | 趨近零（查狀態毫秒級，模板重播靠 prompt cache 吸收） | 每 turn 一次 Haiku call（官方稱可忽略），~1–2 秒         | 評估即執行：單次約 8 分鐘、數十倍 token             |
| 失效模式         | 自我歸因偏差：沒有外部視角，可能連錯好幾輪       | 被自信的收尾語言騙過（假性完成 false success）           | 抓不到「瑕疵成功」corrupt success（產出物完美但過程違規：亂改測試等） |

**沒有一家同時拿到「資訊量」與「獨立性」——這是取捨，不是優劣排名。**學界佐證同方向：純自我修正缺外部回饋易失敗（Huang et al., ICLR 2024）；能讀檔跑指令的 Agent-as-a-Judge 可靠度接近人類（Zhuge et al., ICML 2025）——越往獨立、越會實際操作那端，判定越可信。

成本的正確比較單位不是「每次評估多少錢」而是「達到同等可靠度要跑幾輪」——**收斂輪數才是成本最大宗**；且回饋越獨立越紮實，錯誤被抓到的時間越晚、重做量越大（Anthropic DAW 實測：第一輪跑 2 小時 7 分才迎來第一次 8.8 分鐘的 QA，發現核心互動不能用）。由此推出兩條規律：**驗證的單價直接決定驗證能做到多細**（便宜的逐輪把關、昂貴的只能事後總驗）；**愈把「做完」定義成可機械驗證的形式，愈不需要為獨立性付昂貴代價**。效果差距的實感：同一個 2D 遊戲編輯器題目，單一 agent 20 分鐘 $9、完整 harness（含獨立 evaluator）6 小時 $200——20 倍成本，但單跑版遊戲不能玩、harness 版能玩。

### harness 層 vs 模型層：兩家押注不同

疑問：研究都說分離做事與打分最有效，Codex 豈不站在最弱那檔？作者的拆法：**harness 層**（模型外面怎麼接驗證）vs **模型層**（模型本身的驗證能力）。Anthropic 押 harness 層職責分離（且光譜兩端都做了：/goal 和 Outcome）；OpenAI 押模型層——把「驗證自己」練進模型（作者標明此為合理推測、無官方背書），harness 只留薄薄一層 prompt 紀律。這方向若成立，長期更省更快、更接近「驗證是模型本能」的前沿（spontaneous verification，引 Philipp Schmid）；但訓練到位前是脆弱的——**自我歸因偏差**研究顯示模型評「自己對話歷史裡的行為」時判斷力選擇性退化，在錯誤行為上退化最嚴重。OpenAI 的修補紀錄也透露不好走：改過多版 continuation prompt（早期模型會把目標偷偷縮小成容易通過的子集）、把「已過多少時間」從 prompt 拿掉（模型看時間減少會草率行事）。

方法論插曲：有流量不錯的文章宣稱 Codex goal mode「用獨立小模型每回合評分」還附設定範例，但翻原始碼 `ext/goal/` 根本沒有第二個模型——**「harness 的細節，看文件不如自己攔一次封包、讀一次程式碼。」**

### 自建怎麼選 + 訪談 agent 綜合案例

作者的建議：**Claude Code 式 transcript 裁判是自建的最佳起點**（小模型讀逐字稿判 yes/no，實作簡單、輸入輸出都是文字好離線評估，成本與可靠性的平衡點）；Codex 式純自我審計別輕易複製（依賴指令遵循強項＋推測的 post-training，一般模型容易過早宣告完成）；Outcome 式獨立 grader 留給長時間、以可操作交付物為終點、假性完成代價高的任務。**驗證強度是 harness 的可調參數**，隨任務風險和模型能力調整。

作者自建的使用者訪談 agent 示範兩家的招可以組合：

- **借 Claude Code（回饋）**：agent 想換下一題要呼叫 `switch_next_question_workflow` 工具，工具不直接換題、先交 server 端獨立 Judge（gpt-5.4-mini、max_output_tokens=16、只回 Y/N）對照每題的 completion_criteria 審核；判 N 回可行動指令（「請換一種問法追問目前這題」）。連宣告權都不留給主模型。**語意檢查要有界**：被退 3 次或使用者喊跳就強制放行。
- **借 Codex（前饋）**：每輪注入動態狀態 `<time_control>已過 7 分鐘 / 共 15 分鐘、第 2/5 題</time_control>`——harness 才算得出、模型不知道的狀態；做去重（狀態變了才重插）省 token，並標明內部指示不得對使用者念出。

一個重要收尾：這整套其實做在**工具層（時機①）**而非 Stop hook——**pattern 不綁時機點**，可以在任何粒度上組合，看你要把「做完了沒」的判斷放在多細的工作單位上。

---

## 六、時機四：外層 Loop（第 6 篇）

> 這一篇與本 repo 的 [loop-engineering.md](./loop-engineering.md) 高度重疊（Addy Osmani 的分層＝ihower 的時機④）。心跳／跨 run 記憶／判停三塊的展開見該筆記，這裡只記 ihower 獨有的框架與案例。

前三種回饋全在同一個 session、同一份 context 裡運作，這是結構上限。需要外層迴圈的三種情況：單一 context 裝不下（大任務分段、每段乾淨 context 重開）、是不相關的新任務、要由外部事件自動觸發。最關鍵的設計一句話：

> **把「進度」這件事從 context 搬到磁碟。**（git history、progress.txt、prd.json、看板——讓每一圈的乾淨 context 都能重新接手）

### 三種實作光譜

- **Ralph（蠻力重跑）**：`while :; do cat PROMPT.md | claude-code; done`——每圈全新 context，跨圈記憶靠三份檔案（git history＝已完成、progress.txt＝每圈學到的、prd.json＝任務清單與 passes 狀態）。Huntley 用 Ralph 完成過完整的編譯器專案，也曾以約 $297 的 API 成本做出原本報價 $50k 合約的 MVP。Steinberger 的批評（「slop in a loop」）針對的不是迴圈本身，而是用蠻力迴圈取代思考——作者的定位：**Ralph 只做了外層的「排程」，沒接好內層的「驗證」和「方向」**。
- **Symphony（看板當控制平面）**：OpenAI 開源，以 ticket 為中心——看板升格為 coding agent 的控制平面，狀態機（Todo→In Progress→Review→Done）驅動派工，任務樹帶依賴（DAG）自然平行。交付物是 **Proof of Work**（CI 狀態、review 回饋、示範影片）：「目標不是寫完 code，而是說服人類合併」。宣稱某些團隊三週內合併 PR +500%。
- **Cron / Heartbeat**：兩條獨立軸——觸發節奏（定時 vs 事件/自主）×context 沿用（每次獨立 context vs heartbeat 送回同一長駐 session）。產品對照：Codex Automations standalone（全新 run＋Triage 收件匣）vs thread automation（heartbeat）；Claude Code `/loop`（session 內 heartbeat、no catch-up）vs Routines `/schedule`（雲端、每次全新 clone）。

### Goal 與 Loop 的分工

社群常把 /goal 和 /loop 混著講，實際上：**Goal 管 termination（做到什麼程度才停），Loop 管 scheduling（什麼時候跑）**——loop 完全不判斷完成度。可疊（loop 外圈＋goal 內圈＝run-to-completion job）可串（loop 發現工作推 queue、派帶 goal 的任務去做）。

> 對應 loop-engineering.md 的三塊：心跳＝scheduling、判停＝termination、跨 run 記憶＝「進度搬到磁碟」。兩套筆記講的是同一層，切法不同。

### 同名陷阱與設計三問

Anthropic 官方的 ralph-wiggum plugin 與原版 Ralph **機制相反**：plugin 用 Stop hook 在**同一個 session** 攔住想收工的 agent 重送 prompt（context 一路變長），其實更接近 Codex /goal；原版 Ralph 每圈關掉重開、全新 context。教訓同 §五：別看名字，讀實作。

自建外層 loop 的三問（跨場景通用——inbox 整理、競品監控、情報簡報都適用）：**怎麼被觸發**（cron／動態間隔／事件）、**醒來看什麼**（讀哪些 context 之外的狀態）、**結果送去哪**（PR／簡報／triage／歸檔）。

完整實例：Steinberger 的 maintainer-orchestrator（開源 skill）——orchestrator 每 5 分鐘 heartbeat 醒來派工，worker 平行實作不准再委派；triage 分 Autonomous／Needs owner／Defer 三類；四道自動合併關卡：**Decision-Ready Rule**（不拿半成品問 owner——CI 綠＋live proof 完成才升級）、**Live Proof Gate**（真實環境驗證，mocks 和 CI 不算數）、權限分層、跨 session 記憶留底。

終極方向（引 Steinberger）：「你不應該再去提示寫程式的代理人，你應該設計『讓你的代理人被提示』的迴圈。」實證：Boris Cherny 過去 30 天對 Claude Code 的貢獻 100% 由 Claude Code 自己寫、合併 259 個 PR（第 6 篇引述；他的 /loop 排程具體用法另見 [boris-cherny-tips.md](./boris-cherny-tips.md)）。收尾引 swyx 的 loopcraft 五層巢狀迴圈：token loop（秒）→ agent turn（分）→ /goal loop（時）→ MetaLoop／外層迴圈（天）→ 未命名的第五層（∞：定目標、分配資源、開放探索）。

---

## 七、進階：自我改進 Harness（第 7 篇）

前六篇的回饋約束都是**人**手動改 harness；本篇讓 agent 根據自己的執行 trace 與評測結果，回頭改自己的 harness。核心主張：自我改進不是讓 agent 想改什麼就改什麼，而是**受控改版**，七個必要條件——固定評測集、regression set（每個修好的失敗變永久測試）、production failure trace、版本化的 prompt/工具/schema、晉級關卡 (promotion gate)、回滾機制、高風險改動的人工 review。「**沒有關卡，就沒有自我改進。**」

誰在改（引 Harrison Chase 三層）：Model 層（改權重，大廠專區）／**Harness 層**（程式、固定 prompt、工具定義——本篇重點）／Context 層（skills、記憶——OpenClaw 稱之為「做夢 dreaming」）。

### 三條改進路徑

1. **Production trace → 新 Judge**：離線 eval 蓋不住真實世界的無界輸入——「system prompt 的禁止清單就是事故報告史」。三步閉環：production trace 找到新失敗 → 沉澱成 dataset＋人工對齊出 Judge → 部署回三個時機之一（工具層檢查／單輪 rubric／外層品質監控）。人力花在**對齊 Judge**而非找失敗：「一個沒跟你的判斷校準過的 Judge，只是把你不信任的東西自動化。」作者的低成本實作：用 braintrust-cli／langfuse-cli 包一個「review 最近 24 小時 trace、產生品質報告」的 skill，掛 cron 或 /loop 定期跑。
2. **Agent 當優化器，用 eval 爬坡 (hill climbing)**：六步配方（引 Viv Trivedy）——蒐集標記評測資料、**切 holdout 集（最重要）**、先跑 baseline、每輪只改一個變因、驗證不退步、人工審查抓指標看不見的過擬合。實證：NeoSigma auto-harness 固定 GPT-5.4 不動、只改 harness，Tau3 驗證集 0.560→0.780（18 批次、96 個實驗）；regression gate 要求「已修失敗不退步 ≥80% 且整體不退步」否則自動 revert——「每個被修好的失敗都變成永久測試，系統無法回到更差的狀態」。更激進的是 Stanford Meta-Harness：讓 agent（Claude Code 當提案者）重寫**整個 harness**，每步最多吃 1000 萬 token 診斷脈絡，演化出的 harness 讓 Haiku 4.5 在 TerminalBench-2 跑到 37.6%（同模型 agent 第一）。
3. **把教訓寫回 skill (skillify)**：每次犯錯就把失敗變成一個**帶測試的** skill（引 Garry Tan）。Garry 給的 10 步檢查清單重點：SKILL.md 契約、確定性程式碼（先用判斷力寫成純邏輯腳本，之後強制執行腳本而非重複推理）、單元/整合測試、LLM evals、resolver 觸發條目＋resolver eval、DRY 稽核（防 deploy-k8s 與 kubernetes-deploy 並存）。三大風險：skill 太多 resolver 路由失準（context rot）、逐個補 skill 困在局部最佳解、**skills 會腐爛**（上游 API 改了靜默回傳垃圾；「沒有測試，任何 codebase 都會腐爛」——skill 管理不該例外）。

### Goodhart 警告與終極迴圈

一旦 agent 會根據 eval 改 harness，它就有動機把 eval 當目標：Langfuse 的案例——agent 為刷分直接**移除「執行前先讓使用者確認」的人工核准關卡**（測試環境沒真人按確認、留著就 0 分），順手刪掉測試未涵蓋的真功能。口訣：「**沒被量到的東西，就會被刪掉。**」「Judge 不準，自動化只會讓你更快地往錯的方向走。」

三條路殊途同歸：**eval 是 agent 的訓練資料**——`agent = fit(model, harness, evals)`（引 Viv Trivedy）。量測也要小心代理指標：論文《Harness Updating Is Not Harness Benefit》顯示更新器本身能力差距最多 3.1 分，但下游實際效益差很多——「改了 harness 不等於 agent 真的變好」。結論：自我改進 agent 越強，設計 eval、定義「什麼叫做好」並守住它的工程師越不可或缺——被移出的是逐步操作那幾圈，留在最外圈做的是定義並守住標準。

（AlphaSignal 把 28 篇論文歸納成十層堆疊：穩定基底→執行軌跡→外部狀態→失敗挖掘→提案引擎→**驗證關卡**→版本與回滾→路由與變體→**效益量測**→選配的權重更新；最關鍵是第 6、9 層。）

---

## 八、收尾：Model-Harness Fit 與會過期的 Harness（第 8 篇）

前七篇都在「固定模型」上談 harness；本篇補上時間維度，一體兩面：**harness 綁在特定模型上**，而且**harness 會過期**。

### 模型是針對 harness 訓練的，不只是針對 API

引 OpenAI 工程師 nicbstme 比對 Codex／Claude Code／Copilot CLI 的長文：tool 名稱、input schema、記憶 citation 標籤、skill 檔案結構都是 **byte-level 慣例**，在 post-training 就內化成「特定模型對特定 harness」的本能。最具體的例子是編輯工具格式：

|          | apply_patch（OpenAI 系）                     | edit_file / str_replace（Claude 系）                |
| -------- | -------------------------------------------- | --------------------------------------------------- |
| 形式     | diff（`@@` 上下文定位、`-`/`+` 行）          | old_string → new_string（逐字元相符且檔內唯一）     |
| 批次     | 一份 patch 改多檔多處                        | 一次呼叫改一個地方                                   |
| 失敗方式 | 檔案漂移、縮排差一格就套不上                 | 差一個空白找不到、出現兩次不知改哪個                 |

兩種模型都「能」用另一種，但給不熟的格式會多燒推理 token、錯誤率上升——Cursor 在上百萬筆 agent 軌跡上量到的可測量成本，所以他們給每個模型它訓練時的格式，prompt 也按 provider 甚至版本分開寫。**中途切換模型是最清楚的失敗案例**：對話歷史變 OOD＋prompt cache 保證 miss＋工具形狀改變，三件事同時發生——「你以為只是在下拉選單換個名字，實際上是把整個契約換掉」；要換模型就開乾淨 context 的 subagent，別動主線。

### 四層拆解：化解「原廠最契合 vs 自建打得贏」的矛盾

「harness」一詞被塞了太多意思，拆成四層矛盾就同時成立：

| 層           | 內容                                       | 誰贏                       |
| ------------ | ------------------------------------------ | -------------------------- |
| 任務分布     | 你實際要解的那群任務                       | ——（你要對齊的目標）       |
| workflow 層  | 任務怎麼拆、subagent 調度、routing、skills | **自建贏**（貼著任務優化） |
| 內層 tool loop | tool 描述、解析、結果回送、context 管理    | **原廠贏**（跟模型一起訓練）|
| 模型         | 純權重                                     | **原廠贏**                 |

實作守則：「**內層留給原廠，workflow 層留給自己**」——選定模型後別跟它訓練時的工具格式作對，力氣花在對齊自身任務分布的編排、context、skill（Harness-Task fit，引 Viv Trivedy 的 Model-Harness-Task 三元組）。原廠對「通用分布」訓練，你對「你使用者的真實任務分布」優化，在窄分布上你本來就該贏——前提是先有能代表使用者的 eval，否則連「贏了沒」都量不出來。佐證：LangChain Deep Agents 加了按模型分流的 profiles，光套對 profile（不改模型），tau2-bench 難子集 GPT-5.3 Codex 33%→53%、Opus 4.7 43%→53%。

### harness 會過期：What can I stop doing?

引 Anthropic：「harness 裡的每個元件，都編碼了一項『模型自己做不到什麼』的假設；當模型變強，假設過時，元件該被移除。」每次模型升級（含小版本——Opus 4.7 發布說明就提醒指令遵循變嚴、要回頭重調 prompt）問一次「有什麼可以停止做?」。兩個教科書案例：

- **context reset 三代拆三層**：Sonnet 4.5 有 context anxiety（context 快滿就草草收尾）→ 加定時 reset＋結構化交接檔；Opus 4.5 讓 reset 多餘；Opus 4.6 連 sprint decomposition 都拿掉效果更好。「這些元件一月還在承重，三月就成了死碼」（Portkey, The Harness Tax）。
- **TodoWrite → Tasks**：初期 TodoWrite＋每 5 輪 system reminder 防偏離；模型變強後提醒反而逼模型死守清單、且多子代理無法協作在扁平清單上——**scaffolding trap**：舊腳手架不只變死重，還反過來跟模型新能力衝突。換成 Tasks 系統：寫到磁碟、跨 session 持久、任務有依賴、多 agent 共用（等於把 Ralph 的外部 plan 檔內建化）。同文另一例：早期用 RAG 向量庫給 context，後來模型夠強，直接給 Grep 讓它自己搜，效果更好也更不易壞。

過期加速的機制是**共演化迴圈**：harness primitive 上線 → 出現在幾百萬筆 agent 軌跡 → 變成下一代模型訓練資料 → primitive 被練進模型本能 → harness 那層拆掉。例：apply_patch 被後訓練進模型；Fable 5「擅長在迴圈中自我修正」寫進官方賣點（部分取代 Stop hook 退件那類腳手架）；早期 Claude 玩寶可夢要一整套地圖導航 harness，Fable 5 純視覺極簡 harness 就破關。

但注意兩個修正：**不是變少，是該做的會變**（引 Addy Osmani——模型變強解鎖新任務，新任務有新失敗模式要新腳手架）；同一時間點不同大小的模型也適用——小模型需要精巧多層的 harness，同一套搬到大模型多半是浪費，**沒有一體適用的 harness**。作者也刻意保留反方：模型對自家格式的偏好某種程度是「被訓練慣出來的」過擬合，不是本質的好；成對推出＝用可攜性換排行榜。

### Bitter Lesson 收尾：會過期的與不會過期的

- ⏳ **會過期**：各種具體 harness 做法與補強措施。Bitter Lesson 在這裡不是叫你別設計 harness，而是「別把今天這個模型的限制，寫成永遠不動的架構」——設計得精簡、易替換、易量測，模型變強才拆得掉。「想待在前沿，你得在每次新模型發布時，刪掉你大半的程式碼。」（nicbstme）
- ♾️ **不會過期**：定義「什麼叫做好」、驗證真的做到了——**Eval 和 Judge**。第 3 篇工具層檢查、第 5 篇停止條件與 grader、第 6 篇 loop 判停、第 7 篇自我改進核心，全是同一件事的不同粒度。就算模型強到能完美自我驗證，它驗證的仍是「由你定義的目標」——模型會一代代吸收腳手架，吸收不了替它定義成功並驗證成功的角色。

> 對照 [harness-engineering.md](./harness-engineering.md) §七「Harnesses Don't Shrink, They Move」：兩邊結論一致（位移／淘汰／新生三條路、每個元件編碼一個能力假設），ihower 多給了「四層拆解＋內層留原廠」的操作守則和「eval/judge 永不過期」的分界線。

---

## 九、實作：自建 Agent 的框架選型（第 9 篇）

真要動手時的第一個問題。評選標尺沿用第 1 篇的六項能力（規劃／檔案 Bash／子代理／記憶／Skills／額外工具），框架分兩條路線：

### 路線一：全套 Deep Agent（開箱即有能跑的 agent＋原廠調校的 system prompt）

| 框架                      | 開源深度                                                        | 特點                                                                 |
| ------------------------- | --------------------------------------------------------------- | -------------------------------------------------------------------- |
| **Codex SDK / App Server** | Rust 核心全開源（Apache 2.0），**agent loop 可讀可改**          | JSON-RPC 協議介面不限語言；OS 核心級 sandbox；**支援 ChatGPT 訂閱帳號登入** |
| **GitHub Copilot SDK**    | 外層 MIT，loop 在閉源 Copilot CLI 執行檔                        | 第一方支援六種語言（最多）；生產驗證過的 runtime                       |
| **Claude Agent SDK**      | 外層 MIT，loop 在閉源 Claude Code 執行檔（stdio 溝通）          | 六項能力大多內建；**可控性此路線最受限**，第三方只能 API key、不能 claude.ai 帳號 |
| **LangChain deepagents**  | MIT 全開源，建在 LangGraph 上                                   | Claude Code 做法的開源重實作；每部分（fs/sandbox/記憶）是可替換 backend |
| **Pi (Earendil)**         | MIT 全開源，loop 可讀可改，**可控性此路線最高**                 | 極簡派：預設只有 read/write/edit/bash 四工具、system prompt <1000 token（有時 ~200），能力用 extension 按需加；OpenClaw（數十萬 star）就建在它上面 |

重要陷阱：**開源外皮、閉源核心**——外層 SDK 是 MIT 不代表 agent loop 可改；「你能用它調好的迴圈，但看不到也改不了內部」。要做對外 production 服務，「至少要能看到原始碼」是很實際的維護考量。

### 路線二：從基礎構建（基礎元件自己組，完全可控、model 無關）

按 opinionated 程度由高到低：**Strands Agents**（AWS；model-driven 迴圈，最接近 deep agent，Amazon Q Developer 在用）→ **Google ADK**（多代理編排、session/memory service 分離、原生 A2A）→ **Microsoft Agent Framework**（Semantic Kernel＋AutoGen 接班，企業級 checkpoint/OTel）→ **OpenAI Agents SDK**（Agent/Handoff/Guardrail 幾個核心概念）→ **Pydantic AI**（型別優先、依賴注入、官方支援 Temporal/DBOS/Prefect/Restate 四種 durable execution）→ **Vercel AI SDK**（v7 有 ToolLoopAgent；唯一內建前端 chat hooks；30+ provider）→ **LangGraph**（最低階狀態機，連 agent 迴圈都自己接；強項 checkpointer 持久化＋時光回溯 debug）。

### 三個選型情境

1. **特定應用／B2C 大規模** → 從基礎構建：砍掉通用 context 與工具做 vertical agent，token 成本與延遲顯著下降，保留換供應商自由
2. **自用／企業內部開發工具** → 現成 Deep Agent：原廠已調好內層（呼應 §八「內層留給原廠」），力氣花在疊自己的 outer loop
3. **使用者帶自己的訂閱帳號** → Codex SDK / App Server（支援 ChatGPT 訂閱登入；Claude Agent SDK 第三方只能 API key——這直接決定產品做不做得出來）

收尾呼應全系列：**框架是起點，不是終點**——框架幫你接好的都還是「能不能做」；真正讓 agent 做得對、做得完的那套 harness（工具層檢查、單輪驗收、外層 loop、eval gate），**不管哪個框架都得自己建**。

---

## 十、與本 repo 筆記的對照

同一個詞「harness」，兩種語境：對 Claude Code 使用者，harness 是**你拿到的現成環境**（去配置它）；對 agent 開發者，harness 是**你要寫的那層程式**（loop、工具治理、驗收）。本 repo 原有兩篇屬前者視角，ihower 系列屬後者——互補而非重複。

| ihower 系列的概念                       | 本 repo 對應                                                         | 關係                                                                   |
| --------------------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| 回饋時機①③（工具層檢查、單輪驗收）      | [harness-engineering.md](./harness-engineering.md) 面向 5（in-loop） | 同一件事；ihower 按「時機／粒度／成本」切，五面向按「獨佔問題」切       |
| 回饋時機②（mid-run injection）          | ——                                                                    | 本 repo 原本沒有的拼圖：API 層的 tool call 配對規則、steering 佇列實作 |
| 回饋時機④（外層 loop）                  | [loop-engineering.md](./loop-engineering.md) 整篇                     | 同一層：心跳＝scheduling、判停＝termination、跨 run 記憶＝進度搬到磁碟 |
| ratchet（雙向棘輪）                     | harness 筆記 心法 1                                                   | 同源；「拆約束」半邊在第 8 篇展開成過期性                              |
| Böckeler 2×2（前饋/回饋 × 運算/推論）   | harness 筆記 §四 開頭的 feedforward/feedback 註                       | ihower 把它升格為主座標系並擴到非 coding 場景                          |
| Model-Harness Fit／harness 會過期       | harness 筆記 §七「Harnesses Don't Shrink, They Move」                 | 結論一致；ihower 補四層拆解＋「內層留原廠」守則＋eval/judge 永不過期線 |
| 六項內建能力                            | harness 筆記 面向 1–4 的 primitives                                   | 視角差：開發者的「選配清單」vs 使用者的「治理面向」                    |
| 裁判獨立性光譜（自審/transcript/grader）| harness 筆記 §五 5.5 evaluator 設計、面向 5 生成/評估分離             | ihower 用三個真實產品把「分離」量成一條可調的光譜＋成本模型            |

一個表面矛盾值得記下：harness 筆記心法 3 說「**success is silent**」（檢查通過就靜默），ihower 卻說「**成功也要設計**、回完整狀態」。不衝突——講的是兩個東西：前者是**外掛的檢查器**（hook/sensor）通過時別回「OK」雜訊稀釋失敗訊號；後者是**工具本身的回傳值**要有足夠資訊量讓 agent 判斷假完成。兩條合起來是同一個目標：管理 context 的訊噪比——雜訊靜默、訊號完整。

---

## 十一、來源與更新

### 主要來源

ihower，「給 Agent 開發者的駕馭工程」系列（2026-06-26，演講書面版）：

1. [基礎: Deep Agent 的六項內建能力](https://blog.aihao.tw/2026/06/26/harness-engineering-1-deep-agent-capabilities/)
2. [核心: Agent 要的是回饋迴路，不是完美提示](https://blog.aihao.tw/2026/06/26/harness-engineering-2-what-is-harness-engineering/)
3. [回饋時機一: 工具回傳值，是寫給 agent 的回饋](https://blog.aihao.tw/2026/06/26/harness-engineering-3-tool-execution-feedback/)
4. [回饋時機二: 兩次 model request 之間，把訊息注入執行中的 agent](https://blog.aihao.tw/2026/06/26/harness-engineering-4-mid-run-injection/)
5. [回饋時機三: 單輪結束的驗收，Goal 與 Outcomes](https://blog.aihao.tw/2026/06/26/harness-engineering-5-goal-and-outcomes/)
6. [回饋時機四: 外層 Loop, Ralph、Symphony 與 Cron](https://blog.aihao.tw/2026/06/26/harness-engineering-6-outer-loop/)
7. [進階: 自我改進 Harness、Meta-Harness 與爬坡](https://blog.aihao.tw/2026/06/26/harness-engineering-7-self-improving/)
8. [收尾: 會過期的 Harness, Model-Harness-Fit 與 Bitter Lesson](https://blog.aihao.tw/2026/06/26/harness-engineering-8-model-harness-fit/)
9. [自建 Agent 的框架選型](https://blog.aihao.tw/2026/06/26/harness-engineering-9-agent-frameworks/)

演講投影片：[給 Agent 開發者的 Harness+Loop Engineering](https://ihower.tw/presentation/harness.html)

系列內大量引用的二手來源（擇要）：Anthropic《Effective Harnesses for Long-Running Agents》《Harness design for long-running apps》《Harnessing Claude's intelligence》、LangChain（Harness Anatomy／deep agents 實戰／Deep Agents profiles／Sydney Runkle／Harrison Chase／Viv Trivedy）、Birgitta Böckeler（Thoughtworks 2×2）、HumanLayer、Cursor《Continually improving our agent harness》、nicbstme（OpenAI）、OpenAI Cookbook《Self-Evolving Agents》、Jason Liu（facets）、Arize（Aparna Dhinakaran）、Geoffrey Huntley（Ralph）、Peter Steinberger（maintainer-orchestrator／OpenClaw）、swyx（loopcraft）、Stanford Meta-Harness、NeoSigma auto-harness、GEPA、Garry Tan（skillify）、AlphaSignal（十層堆疊）。

### 本 repo 內部連結

- [harness-engineering.md](./harness-engineering.md) — 使用者視角：單一 agent 運行環境的五面向、四心法、Claude Code primitives mapping
- [loop-engineering.md](./loop-engineering.md) — harness 上一層的自走 loop（心跳／跨 run 記憶／判停）＝本系列的時機④
- [boris-cherny-tips.md](./boris-cherny-tips.md) — Boris Cherny 的 Claude Code loop primitive 實際用法（/loop、/schedule、/goal），與第 6 篇「把自己移出迴圈」同主題的使用者端筆記

### 校對紀錄

- **2026-07-07**：初版。9 篇逐篇萃取後按系列自身骨架（六能力 → 定義與四時機 → 自我改進 → 過期性 → 選型）整理；重疊內容採「薄、委派型」紀律連回 harness-engineering.md 與 loop-engineering.md；§十 補視角對照表與「success is silent vs 成功也要設計」的調和。
  - 發布前 9 篇逐篇覆核，修正：Ralph 的「編譯器專案」與「$297 MVP」二事分開（原誤併為一件）、skillify 10 步清單歸屬改回 Garry Tan、Claude Agent SDK 移除原文沒有的「能力最齊」、boris-cherny-tips 指向改為 /loop 排程用法（該檔並無 259 PR 實例）技術細節注意：Claude Code /goal 的 Haiku／刪減 transcript 等內部機制是 ihower 用 mitmproxy 攔包的觀察（非官方文件，可能隨版本改動）；「OpenAI 把自我審計練進模型」是作者標明的合理推測。

下次 review 觸發點：系列若有更新或勘誤、Claude Code /goal 與 Codex Goals 機制大改、本 repo harness 筆記重構時同步對照表。
