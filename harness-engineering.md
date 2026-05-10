---
title: Harness Engineering
tags: [agent, claude-code, harness, ai-engineering, prompt-engineering]
created: 2026-05-01
last_reviewed: 2026-05-10
type: reference
status: living-document
sources:
  - Addy Osmani, "Agent Harness Engineering" (2026-05-10)
  - Vaibhav Trivedy (@Vtrivedy10) — "harness engineering" 命名
  - HumanLayer / @dexhorthy — "skill issue" reframe
  - Anthropic — context engineering 研究
---

# Harness Engineering:讓 AI Agent 從 Demo 走向生產的工程體系

> 一份綜合性的參考筆記。涵蓋定義、核心心法、七大面向、技術分層,以及對應到 Claude Code 的具體 primitives。

## 目錄

- [前言:被命名之前已經存在的工程現實](#前言)
- [一、Harness 到底是什麼](#一harness-到底是什麼)
- [二、為什麼這個詞現在火了](#二為什麼這個詞現在火了)
- [三、Harness 的核心心法](#三harness-的核心心法)
  - [心法 1:The Ratchet — 錯誤的單向累積](#心法-1the-ratchet--錯誤的單向累積)
  - [心法 2:Working Backwards from Behavior](#心法-2working-backwards-from-behavior)
  - [心法 3:Success Is Silent, Failures Are Verbose](#心法-3success-is-silent-failures-are-verbose)
  - [心法 4:軟硬約束光譜](#心法-4軟硬約束光譜)
- [四、Harness 的七大面向](#四harness-的七大面向)
  - [面向 1:上下文與記憶管理](#面向-1上下文與記憶管理)
  - [面向 2:工具治理](#面向-2工具治理)
  - [面向 3:安全與審批](#面向-3安全與審批)
  - [面向 4:工作流程編排](#面向-4工作流程編排)
  - [面向 5:回饋與狀態](#面向-5回饋與狀態)
  - [面向 6:架構約束與熵管理](#面向-6架構約束與熵管理)
  - [面向 7:觀測與計量](#面向-7觀測與計量)
- [五、對應到 Claude Code:Primitives Mapping](#五對應到-claude-codeprimitives-mapping)
- [六、技術分層:Vibe → Spec → Harness](#六技術分層vibe--spec--harness)
- [七、落實考量](#七落實考量)
- [八、結語](#八結語)
- [九、來源與更新](#九來源與更新)

### 閱讀路徑建議

- **第一次讀**:§一 → §三 → §八(掌握核心概念與心法)
- **動手設定 Claude Code**:直接看 §五
- **制度/規範設計者**:§三 + §四 + §七
- **想知道與其他框架關係**:§六

-----

## 前言

Agent 圈密集討論一個詞:Harness。許多人第一次看到,會以為這是新框架、新範式,甚至像是什麼「下一代 Agent 技術名詞」。

但仔細看完會發現,Harness 不是新技術發明——它更像是 Agent 工程裡那部分一直存在、但過去沒被完整命名的「軟體工程現實」。

從原理上講,Agent 真的不複雜。一句話就能寫出來:

```
Agent = Loop(LLM + Context + Tools)
```

在一個迴圈裡提供工具、維護上下文、呼叫大模型——十幾分鐘就能寫出最小版本。給模型一個 system prompt、給它幾個 tools、讓它在迴圈裡不斷決定要不要繼續做事,這就已經是 agent 了。

但為什麼相同的迴圈,在生產環境會爛掉、在 demo 階段卻看起來很厲害?

答案通常不是因為模型突然變笨,而是因為這些工程問題開始集中出現:上下文越來越髒、工具越來越多但沒有治理、狀態沒有結構化、安全邊界靠「相信模型懂事」、錯誤沒有歸類、長任務沒有階段性 checkpoint、文件和規則堆成一坨模型根本吃不動、人和 agent 的協作邊界不清晰。

這些問題,全都不是「模型理論問題」,它們是徹頭徹尾的軟體工程問題。Harness 這個詞的價值就在這裡:它提醒人們,**Agent 專案的真正主戰場,不是模型層,而是執行環境層**。

-----

## 一、Harness 到底是什麼

### 詞源與致謝

「Harness Engineering」一詞的定形源於以下幾位的工作:

- **Vaibhav Trivedy**(@Vtrivedy10)— 命名與核心定義
- **HumanLayer / @dexhorthy** — 提出「failures are configuration problems, not model problems」reframe
- **Anthropic** — context engineering 研究與 Claude Code 官方文件,提供大量底層思路
- **Addy Osmani** — 將上述思路整合成完整論述(《Agent Harness Engineering》,2026-05-10)

本筆記在這些基礎上,結合中文社群討論與 Claude Code 實作經驗綜合整理。

### 定義

可以這樣定義:
> **Harness 是 Agent 運行時的「工程環境總和」,是讓模型在約束中穩定完成任務的整套運行時系統。**

兩種互補的觀察視角:
- **結構式**:`Agent = Loop(LLM + Context + Tools)`
- **能力式**:`Agent 表現 ≈ 模型 × Harness`(任一邊弱,整體就崩;Addy Osmani 用加法 `Agent = Model + Harness` 強調「不是模型的就是 harness」,語氣稍有差別但結論一致)

這個「環境總和」(以**關注維度**列舉)至少包含:

1. 任務如何被表達
1. 上下文如何被組織
1. 工具如何被暴露、治理、攔截
1. 狀態如何被儲存、恢復、裁剪
1. 回饋如何回流給模型
1. 錯誤如何被分類、重試、升級
1. 安全邊界如何被建立
1. 系統如何驗證 agent 真的把事做好了
1. 執行成本與行為如何被觀測(log、trace、cost、latency)

對照 Trivedy 的 inventory(以**具體 artifact** 列舉)則包括:

- system prompts、CLAUDE.md、AGENTS.md、skill files、subagent instructions
- tools、skills、MCP servers 與其技術描述
- bundled infrastructure(filesystem、sandbox、headless browser)
- orchestration logic(spawning subagents、handoffs、model routing)
- hooks 與 middleware(deterministic execution、lint check、context compaction)
- observability tools(logs、traces、cost、latency 度量)

如果把 Agent 想成一個「會呼叫工具的大腦」,那 Harness 就是給這個大腦配的身體、感測器、護欄、操作台、流程制度、回執系統,以及出事後的保險。

### Working Backwards from Behavior(設計方法論)

設計 harness 的最有效方式不是「先設計元件、再看能做什麼」,而是反向:
> **想要的行為 → 達成該行為的元件設計**

每個 harness 元件都該能說出它服務的具體行為。**講不出職責的元件,就應該被刪掉**。這條原則同時也是熵管理的篩選器(見 §四 面向 6)。詳見 §三 心法 2。

### 與相鄰概念的差異

值得釐清三組常被混淆的概念:

|詞彙                 |關注點                |
|-------------------|-------------------|
|Prompt Engineering |怎麼說(措辭、few-shot 編排)|
|Context Engineering|給模型什麼資訊            |
|Harness Engineering|模型在什麼環境裡做事         |

Context Engineering 是 Harness 的一個組成部分,但 Harness 不等於 Context。它比 Context 更大一層,因為它不只管「餵什麼」,還管:什麼時候能動手、動手時有哪些工具、工具是否需要審批、輸出太長怎麼辦、危險操作怎麼攔、會話如何恢復、中間產物如何持久化、任務失敗後如何修復。

同樣地,LangChain、LangGraph、CrewAI 這類 SDK 也不等於 Harness——它們回答的是「怎麼造 Agent」,而 Harness 回答的是「Agent 運行時,世界如何與它互動」。可以用 LangChain 實現 Harness 的某個模組,但 LangChain 本身並非 Harness。

-----

## 二、為什麼這個詞現在火了

因為行業終於開始承認一件事:

> **Agent 的問題,越來越不是「模型夠不夠聰明」,而是「環境夠不夠好」。**

幾組業界資料佐證了這個論點(資料時點:2025 年中至 2026 年初,具體百分比可能隨後續實驗變動):

- **LangChain 實驗**:只優化 Harness、不換底層模型,寫程式 agent 在 Terminal Bench 2.0 得分顯著提升(從中段班跳到前段班)
- **Vercel 實驗**:大幅移除(約八成)agent 工具,反而步驟更少、Token 消耗更低、任務成功率更高
- **OpenAI 案例**:少數工程師 + Codex Agent,在數個月內生成百萬量級程式碼

這些資料共同指向一個結論:**Agent 表現 ≈ 模型 × Harness**。

Addy Osmani 提供了另一個更鮮明的 framing:

> **頂級模型放進現成框架的得分,常輸給次級模型放進精調 harness 的得分。**

換句話說,benchmark 上看到的差距,不全是模型差距,**很大一部分是工程設計差距**。

這也意味著一件事:**驗證一套 harness 的方向是否正確,最好的方式不是用最新最強的模型測,而是用一個發布將近一年的舊模型測**。如果一個相對古早的模型在這套環境裡仍能做成事,那才說明工程方向是對的。

-----

## 三、Harness 的核心心法

進入元件目錄之前,先建立四條貫穿全文的設計原則。每個面向、每個 primitive 都該對應到至少一條心法。

### 心法 1:The Ratchet — 錯誤的單向累積

最核心的一條心法。HumanLayer 的講法是:
> **"It's not a model problem. It's a configuration problem."**

實作上的紀律:

- agent 每次失敗 → 寫成永久規則 / hook / AGENTS.md 條目
- 規則**只在觀察到實際失敗**時才加(不要預先假設)
- 規則**只在模型升級到能自己處理**時才移除
- 一條好的 system prompt,**每一行都該追溯到一次具體的歷史失敗**

換句話說,harness 是一個只進不出(或極少出)的棘輪——錯誤一旦被觀察到,就要轉成永久信號,不要重複付一樣的學費。

> Ratchet 與後面 §四 面向 6 的「熵管理」剛好是兩個方向:Ratchet 在加,熵管理在減。兩者合起來才是完整迴圈——規則該加時果斷加,該減時果斷減。

### 心法 2:Working Backwards from Behavior

從「期望行為」反推 harness 元件,而不是從元件目錄逐項上架。

- 每個 hook、每個 sub-agent、每條 rule、每個 MCP server,都該能說出它服務的**具體行為**
- 講不出職責 → 刪掉

這條同時也是 review harness 的標準:定期問「這個元件到底在防哪一類失敗?」「沒有它會發生什麼?」如果答不出來,它就是熵的來源。

### 心法 3:Success Is Silent, Failures Are Verbose

Hook 與 feedback 設計的核心心法。

- 檢查通過 → agent **完全聽不到**(不污染 context、不消耗 token)
- 檢查失敗 → 錯誤訊息**直接注入回 context**,讓 agent 自己看見並修正

實作意義:PostToolUse 跑 lint / typecheck,過了就靜默;沒過就用 exit code 把錯誤回傳。這比把錯誤丟給人類看再轉達更省時間,也讓 agent 自成回饋迴路。

### 心法 4:軟硬約束光譜

每條規則都該決定它的位階:

- **軟約束**:寫進 prompt(CLAUDE.md / AGENTS.md / Rules)— 模型「應該」遵守
- **硬約束**:寫進 hook / 規則引擎 / CI lint — 模型「無法違反」

從軟到硬的升級路徑(也是 Ratchet 的預設動作):

1. 先在 AGENTS.md 寫一條規則(軟)
1. 觀察到 agent 反覆違反或誤讀(Ratchet 觸發)
1. 升級成 PreToolUse / PostToolUse hook(硬)
1. 進一步固化成 CI 階段的結構測試(最硬)

不該所有規則都直接做成 hook(過度工程),也不該所有規則都靠 prompt(不可靠)。**用「失敗成本 × 失敗頻率」決定該升到哪一層**。

-----

## 四、Harness 的七大面向

### 面向 1:上下文與記憶管理

最基礎、也最容易翻車的一層。核心要對抗的問題叫 **context rot**:當關鍵內容落在上下文中間位置時,模型表現會下降 30%+。即使是百萬 token 的視窗,內容一多,指令遵循能力依舊會下降。

**要做的事:**

- **持久化指令文件**:repo 根目錄放 `CLAUDE.md` / `AGENTS.md`,寫架構、build/test 指令、命名約定。OpenAI 在程式碼庫散佈大量 `AGENTS.md`(報告稱 80+ 個,具體數字會變),agent 進入對應目錄時自動載入規則
- **作用域組裝**:組織級 / 使用者級 / 專案根 / 父目錄 / 子目錄分層載入,monorepo 必備
- **分層記憶**:精簡索引常駐 context(幾百行內),相關內容按需載入,完整歷史寫入磁碟
- **記憶整合(Dream Consolidation)**:背景跑「垃圾回收」邏輯,在空閒時定期去重、刪舊、重組結構
- **漸進式上下文壓縮**:新對話保留細節,稍舊內容做輕量總結,再往前的逐步折疊成短摘要——越久遠的資訊保留得越粗
- **Tool-call offloading**:工具呼叫產生的大量輸出(例如 2,000 行 log),寫入 filesystem,context 只保留**指針 + 摘要**。模型需要時再用 grep/read 局部存取——避免把 2k 行直接吃進 context
- **動態提示詞組裝**:把 prompt 拆成多層(基礎模板 / 使用者偏好 / 專案上下文 / Skills / 工具描述 / 工具 JSON),不同新鮮度、不同優先級分開注入

**Filesystem 與 Git 是 agent 的外部認知架構**

- Filesystem 把模型「能操作的東西」從 context window 解放——文件可以遠大於 context,模型用 Read/Edit/Glob 局部存取
- **Git 不只是版本控制工具,是 agent 的 free 回溯狀態**:branch 試錯、checkpoint、rollback、diff 自我檢查——這些都是 harness 層免費繼承的能力,不需要自己實作

關鍵心態的轉變:**提示詞不是靜態文件,而是動態組裝系統**。不能把所有來源的資訊全塞進一個大 prompt 裡,然後指望模型自己理解層級關係。必須先在工程上把層級關係建好,再交給模型。

### 面向 2:工具治理

只要 agent 開始接工具,複雜度會立刻上升。工具系統背後至少有四個問題:模型如何發現工具、參數如何驗證、風險如何分級、呼叫如何被統一攔截。

**雙軌策略:Bash + 專用工具**

業界對「該給 agent 多少工具、長什麼樣」存在兩種看似對立的主張:

- **「少而專用」**派:把 cat / sed / grep 拆成專用的 Read / Edit / Grep 工具,邊界明確、權限好控、審計清晰(Claude Code 的預設選擇)
- **「Bash + Code Exec」**派:Agent 對 shell 已經很熟,給它 bash + sandbox,讓它自己組合(Addy Osmani:「agents generally excel at shell commands」)

實務上**兩者並存**:日常讀寫用專用工具(可審計、可攔截),臨時計算/組合操作落 bash(彈性、低 token)。Claude Code 同時提供 Read/Edit/Glob/Grep 與 Bash 就是這個設計。

**其餘要做的事:**

- **宣告式工具定義 DSL**:把工具寫成可驗證的 schema,而非裸 function
- **漸進式工具擴展**:預設只給少數常用工具(讀寫檔、搜尋),複雜工具按需載入
- **工具收斂原則**:**10 個聚焦、互不重疊的工具,勝過 50 個重疊不清的工具**(Vercel 的 80% 移除實驗就是這個道理)。每多一個工具,模型決策成本上升、prompt token 上升、審計面積上升
- **指令風險分類**:低風險自動放行、高風險才人工確認(read / write / execute / critical 四級)
- **統一編排層**:輸入參數驗證、審批攔截、結果裁剪、錯誤歸類四件事必須在這層做完

**安全角度:工具描述是 prompt surface**

工具的 description 欄位會被注入到 prompt 餵給模型——這意味著:

- **來路不明的 MCP server 等同於 prompt injection 攻擊面**
- 第三方 MCP 應該被當成「不受信內容」審查,description 內容也要 review
- 自家工具的 description 設計亦應遵守 prompt-as-code 的紀律(避免敏感邏輯外洩、避免讓模型產生錯誤聯想)

許多 Agent 一開始能跑、後來越來越不穩,根因是它們的工具系統不是「系統」,只是「工具列表」。Harness 的核心任務是把工具從「模型可以呼叫的能力」,提升成「可控、可審計、可演化的運行時能力」。

### 面向 3:安全與審批

只要 agent 能寫文件、跑 shell、改設定,它就不再是「聊天模型」,而是作業系統參與者。安全問題不是「以後再說」,而是第一天就存在。

**最關鍵的一個原則:**

> 安全不能只靠模型自覺。不能把「請不要執行危險指令」寫在 prompt 裡,然後假裝問題解決了。**真正的安全邊界,必須落在運行時**。

(對應心法 4:這條規則必須是**硬約束**,不能停在軟約束。)

**三道防線:**

1. **子程序管理**:會話池、數量限制、輸出截斷、逾時終止、自動清理
1. **指令守衛**:指令解析、危險指令黑名單、system hint 回傳
1. **審批系統**:風險分級、審批模式、指紋快取、dangerous 模式保底

加上 **Human-in-the-loop 強制節點**:資金操作、資料去識別化、系統變更等高風險操作前強制暫停並等待人工確認。可以類比財務審批的「四眼原則」。

Demo 級別的 agent 靠的是「模型大概會聽話」;Harness 級別的 agent 靠的是「即便模型不聽話,系統也有邊界」。

### 面向 4:工作流程編排

核心就是一個詞——**分離**。把讀取和寫入拆開,把「查資料」和「改程式碼」的上下文拆開,把順序執行和並行執行拆開,把**生成和評估**拆開。

大多數 Agent 的預設做法是把這些事情混在一起,剛開始可能沒問題,但任務一複雜,品質很容易崩——所有東西都會堆在同一個上下文裡,研究內容、規劃討論、程式碼修改、日誌輸出全混在一起,等真正開始改程式碼時,很多無關資訊已經在干擾判斷。

**要做的事:**

- **探索-規劃-行動三段式**:權限逐步放開——先只讀模式探索結構 → 對齊規劃方案 → 才給執行權限
- **上下文隔離子 agent**:研究 agent 只能讀、規劃 agent 只設計、執行 agent 才有完整工具權限。每個子 agent 只接觸自己需要的資訊,避免被「流雜訊」污染
- **生成 / 評估分離**:**模型對自己的產出有正向偏誤**(positive bias)——它寫的程式碼,它自己 review 容易說「沒問題」。把生成 agent 和評估 agent 拆成不同實例(各自獨立 context、甚至不同 system prompt),評估 agent 才能有獨立判斷
- **長時任務的 force-continuation**:agent 很容易在任務沒做完時提早宣告完成。在 Stop hook 攔截「我已完成」訊號,對照原始 completion goal 檢查,沒達成就把它丟回 loop 繼續做(這是「事情沒做完」,跟面向 5 的「事情沒做對」不同——前者是進度問題,後者是正確性問題)
- **Fork-Join 並行**:跨檔案互不依賴的改動可以拆分,每個子 agent 在獨立的程式碼副本(例如 `git worktree`)裡跑,互不干擾,等都完成後再合併

代價是:多了協調成本,小任務會顯得「流程過重」,合併分支也可能比順序處理更難解。但對複雜、不熟悉的程式碼庫來說,這個取捨划算。

### 面向 5:回饋與狀態

Agent 之所以能持續做事,不是因為它會一直「想」,而是因為它會不斷收到回饋——工具執行結果、審批是否通過、輸出是否被截斷、搜尋結果是否過多、指令是否被拒絕、目前任務計畫是否更新、會話是否可恢復。

**最關鍵的兩個原則:**

> 1. 要把系統內部發生的事,翻譯成模型能消費的回饋語言。
> 2. **Success is silent, failures are verbose**(對應心法 3)。

如果沒有第一層翻譯,模型只能看到「失敗了」。有了它才能理解失敗是因為權限問題、是因為輸出被裁剪、是因為搜尋範圍太大、還是因為指令危險而被拒絕。

如果沒有第二層紀律,通過的檢查也會餵給模型一堆「OK」雜訊,稀釋真正重要的失敗訊號。

這直接決定了 agent 是否能做出下一步正確行動。

**要做的事:**

- **異常翻譯機制**:用結構化的 system_hint(例如 XML 標籤)承載異常狀態
- **Checkpoint 機制**:長時間運行任務中定期儲存狀態快照,讓 agent 從失敗點恢復而非從頭開始
- **確定性生命週期 hooks**:必須每次都做的事——改完程式碼跑 format、執行前做驗證、切換目錄時重新載入設定——絕對不能寫在 prompt 裡靠模型記。掛到 agent 生命週期的關鍵節點上自動執行
- **Silent on success**:hook 通過時不要產生 acknowledgement,失敗時才把 stderr 注入 context

判斷準則:**凡是「不能出錯、不能漏」的事,都不該交給模型記,而應該交給系統保底**。

### 面向 6:架構約束與熵管理

最容易被忽略,但在長期運行中最關鍵。

**架構約束**的核心理念是:放棄 LLM 的軟性約束,用確定性規則引擎做硬性管控(對應心法 4 的「硬約束」位階)。具體做法包括 CI/CD pipeline 的自定 lint 規則、驗證架構模式的結構測試、清晰的模組邊界定義——agent 輸出結果必須通過「硬檢查」才能落實,違規直接攔截。這是企業級系統的永恆取捨:**放棄「生成任何東西」的靈活性,換取系統的可靠性**。

**熵管理**對抗的是另一個必然發生的問題:agent 一旦持續運行,系統一定會逐漸「變髒」——prompt 越來越長、rules 越來越舊、AGENTS.md 越來越像垃圾場、工具定義越來越多但沒人維護、會話歷史越來越重、memory 混入大量無效資訊。

> 當一切都重要時,一切都不重要。大而全的說明文件會迅速腐爛。

所以 Harness 不是「搭完就完了」的東西。它有一個長期任務:**持續做熵管理,讓系統不斷回到「對模型友好」的狀態**。

**要做的事:**

- 定期跑專職「垃圾回收 agent」:掃描文件矛盾、發現架構違規、清理技術債——這批 agent 不創造新功能,只做清潔工
- 過期規則失效機制(規則需要定期 review,沒人記得當初為什麼加的就是熵)
- 長上下文壓縮
- 低價值歷史退出 context
- 專案知識以可維護方式沉澱

> Ratchet(心法 1)+ 熵管理(本面向)= 規則的生命週期完整迴圈。沒有 Ratchet 系統會學不到教訓,沒有熵管理規則會堆成廢墟。

### 面向 7:觀測與計量

> 生產級 agent 沒有觀測層 = 盲飛。

這個面向回答的問題不是「agent 做什麼」(那是面向 5 的回饋),而是「**作為 operator,我怎麼知道 agent 在做什麼、花了多少錢、品質好不好**」。

**涵蓋:**

- **執行 trace**:完整 tool call 序列、輸入輸出、執行時間。可重播、可診斷
- **Cost / Latency 計量**:每次 session 的 token 用量、$ 成本、p50/p95 latency。沒有這層,看不到「最近哪個 agent 變貴了」「哪個 hook 拖慢了 90% 的 session」
- **失敗分類**:依錯誤類型、發生位置、recovery 結果聚合。**這是 Ratchet 的最重要輸入源**——沒有失敗統計,就找不到該升級成 hook 的高頻錯誤
- **回顧分析**:定期跑 batch 分析過去 N 天的 session,找重複失敗模式、找 token 浪費熱點、找 sub-agent 編排是否合理

**為什麼觀測值得單獨成一個面向?**

因為它的「使用者」是 operator(人類工程師)而不是 agent 本身。前 6 個面向都在優化模型的執行環境,觀測在優化**工程師對 agent 的理解**。沒有它,前 6 個面向的所有改善都是黑箱中的盲目調整。

對應到 Claude Code 的具體 primitives 見 §五.7。

-----

## 五、對應到 Claude Code:Primitives Mapping

> ⚠️ 本節內容對應到 **Claude Code 2026-05 前後版本**。Primitive 名稱、CLI flag、快捷鍵、設定 schema 會隨版本演進——最新規格以 [Anthropic 官方文件](https://docs.claude.com/en/docs/claude-code) 為準,本節僅捕捉設計理念。

Claude Code 本身就是一個 harness——打開 terminal 輸入 `claude` 那一刻,就已經在一個 perception-action-verification 環境裡工作了。它的 extension layer 提供了一組 primitives,讓使用者把預設 harness 塑造成「自己的 harness」。

### 速查表

|面向         |主要 Primitives                                                  |
|-----------|---------------------------------------------------------------|
|1. 上下文與記憶  |`CLAUDE.md`、`.claude/rules/`、Skills、`/compact`、Imports         |
|2. 工具治理    |MCP servers、Permissions、`allowedTools` / `deniedTools`         |
|3. 安全與審批   |`/sandbox`、PreToolUse hook、Permission modes                    |
|4. 工作流程編排  |Plan Mode、Sub-agents (Task tool)、Slash commands、Stop hook       |
|5. 回饋與狀態   |Lifecycle hooks、Session logs、`--resume`、Statusline             |
|6. 架構約束與熵管理|PostToolUse hooks、Stop hooks、清潔 sub-agent                      |
|7. 觀測與計量   |Session JSONL、Statusline cost/token、SessionEnd hook 外送遙測       |

### 5.1 上下文與記憶 → CLAUDE.md + Rules + Skills

|Primitive                 |對應做法                                                              |
|--------------------------|------------------------------------------------------------------|
|`CLAUDE.md`(專案級)          |repo 根目錄,session 啟動時自動載入。寫架構、build/test 指令、命名約定、已知陷阱              |
|`~/.claude/CLAUDE.md`(全域) |個人通用偏好(commit convention、code review 標準、禁止行為)                     |
|`.claude/rules/`          |條件式規則目錄,用 `paths:` glob 指定「只有碰到某些檔案時才載入」。前端/後端規則可分開               |
|Skills (`.claude/skills/`)|可重用的指令集封裝領域知識,Claude 自動判斷何時啟用                                     |
|`@import` 語法              |CLAUDE.md 可 `@filename.md` 匯入其他文件,實現作用域組裝(對應分散式 AGENTS.md 的散佈策略)|
|`/compact` 指令             |手動觸發上下文壓縮;auto-compaction 在視窗接近滿時自動執行                             |

### 5.2 工具治理 → MCP Servers + Permissions

```json
// .claude/settings.json (schema 隨版本變動,以官方文件為準)
{
  "permissions": {
    "allow": ["Bash(npm test:*)", "Bash(git status)"],
    "deny": ["Bash(rm -rf:*)", "Read(.env)"],
    "ask": ["Bash(git push:*)"]
  }
}
```

- **MCP servers**:把外部工具(GitHub API、database、CI/CD)透過 MCP 介接,而不是讓 agent 自己亂 curl。每個 tool 邊界明確、可檢視
- **指令前綴授權**:Bash 工具可以細到「只允許 `git commit`,不允許 `git push`」這種粒度
- **WebFetch / WebSearch 黑白名單**:限制 agent 能存取的網域
- **MCP description 安全審查**:第三方 MCP server 的 tool description 會吃進 prompt,等同於可被任意人撰寫的 prompt 段落——**新增 MCP 前 review description 內容**,避免引入 prompt injection 面

### 5.3 安全與審批 → Sandbox + Hooks + Permission Modes

**`/sandbox` 指令**:啟用 OS 級隔離。

- macOS:Seatbelt(開箱即用)
- Linux/WSL:bubblewrap + socat(需要安裝)
- 設定 allowed paths、allowed hosts,內部測試可顯著減少 permission prompt 觸發次數

**PreToolUse hook 範例(攔破壞性指令):**

```bash
# .claude/hooks/pre-tool-use.sh
input=$(cat)
tool=$(echo "$input" | jq -r .tool_name)
cmd=$(echo "$input" | jq -r .tool_input.command)
if [[ "$tool" == "Bash" && "$cmd" =~ rm[[:space:]]+-rf ]]; then
  echo '{"decision": "block", "reason": "rm -rf blocked"}'
fi
```

**Permission modes**:

- `default`:每個 tool 都問
- `acceptEdits`:檔案編輯自動通過
- `plan`:純規劃模式,完全不能執行
- `bypassPermissions`:全部跳過,僅建議在 sandbox 內使用

**通用策略**:在能隔離的環境裡放寬編輯權限,保留高風險操作的人工審批。例如 `/sandbox + acceptEdits` 的組合在隔離前提下兼顧效率與安全。

### 5.4 工作流程編排 → Plan Mode + Sub-agents + Slash Commands

|Primitive                 |用途                                                   |
|--------------------------|-----------------------------------------------------|
|Plan Mode                 |純規劃模式,Claude 只讀不寫,直到 approve plan 才執行。對應「探索-規劃-行動三段式」|
|Sub-agents(Task tool)     |把任務委派給隔離的 Claude 實例,各自有獨立 context                    |
|`.claude/agents/`         |自訂 sub-agent,設定 system prompt 和 allowed tools       |
|`.claude/commands/`       |Custom slash commands,封裝重複工作流程                        |
|Stop hook                 |攔截「agent 認為任務完成」訊號,執行 force-continuation 或最終驗證       |
|Git worktree + 多 Claude 實例|對應 Fork-Join 並行,每個 worktree 跑獨立 session              |

常見 sub-agent 模式(對應面向 4 的「分離」原則):

- **Researcher**:只做研究,不污染主 context
- **Reviewer / Evaluator**:檢查實作。**獨立成 sub-agent 的關鍵理由是抗正向偏誤**——讓寫程式的 agent 自己 review 容易放水,需要一個沒看過實作過程、只看結果的獨立判斷者
- **Cleaner**:做熵管理(見 5.6)

常見 slash commands:

- `/review`:只看 bug 和 security
- `/plan`:先產出 plan 到 `plans.md` 等 approve
- `/setup`:一鍵初始化開發環境

### 5.5 回饋與狀態 → Hooks + Session Logs + Statusline

**Lifecycle hooks**(節錄常用,完整列表以官方文件為準):

|Hook              |用途                                 |
|------------------|-----------------------------------|
|`PreToolUse`      |tool 執行前攔截                         |
|`PostToolUse`     |tool 完成後驗證(format、type check)      |
|`UserPromptSubmit`|使用者送出訊息前                           |
|`Stop`            |agent 結束回應時觸發。**用途分兩類**:正確性驗證(沒過要求繼續)、進度檢查(force-continuation 對照 completion goal)|
|`SubagentStop`    |sub-agent 結束時                      |
|`PreCompact`      |上下文壓縮前(備份 transcript 用)            |
|`SessionStart`    |session 開始(載入 git diff、open issues)|
|`SessionEnd`      |session 結束;適合送 trace/cost 到外部觀測系統   |
|`Notification`    |通知事件                               |

**Hook 設計原則(對應心法 3):success silent, failures verbose**

- 通過 → exit 0,不輸出
- 失敗 → exit 2(或寫入 stderr),Claude 會把訊息注入下一輪 context

**Session logs(JSONL)**:每個 session 完整記錄落在 `~/.claude/projects/`,可事後分析 agent trace。

**`--resume`**:恢復先前 session,對應 Checkpoint 機制。

**Statusline 設定**(`.claude/statusline.sh`):自訂底部狀態欄,顯示 context 用量、git branch、cost、token rate——把系統內部狀態翻譯成可消費的回饋。

**output-style 自訂**:設計 Claude 回應結構,例如「永遠先說做了什麼、再說結果」這種結構化回饋格式。

### 5.6 架構約束與熵管理 → PostToolUse + 清潔 Sub-agent

**PostToolUse 跑硬檢查**(範例骨架,完整參數見官方文件):

```bash
# .claude/hooks/post-tool-use.sh
# 改完程式碼自動跑 lint+typecheck;失敗 exit 2 把錯誤丟回 Claude
input=$(cat)
tool=$(echo "$input" | jq -r .tool_name)
[[ "$tool" =~ ^(Edit|Write)$ ]] || exit 0
npm run lint && npm run typecheck || {
  echo '{"decision": "block", "reason": "lint or typecheck failed"}'
  exit 2
}
```

exit code 2 會把錯誤回傳給 Claude,讓它修(對應「翻譯成回饋語言 + failures verbose」)。

**Stop hook 跑最終驗證**:agent 認為自己做完時,觸發完整測試,沒過就要求繼續。

**定期清潔 Sub-agent**:在 `.claude/agents/cleaner.md` 設計專職 maintenance agent,定期跑:

- 掃描 CLAUDE.md 中過期的指令
- 偵測 `.claude/rules/` 中互相矛盾的規則
- 列出超過 N 行的 prompt 內容建議壓縮
- 整理 commands 目錄,合併重複功能

### 5.7 觀測與計量 → Session JSONL + Statusline + 外送遙測

對應面向 7。Claude Code 沒有內建完整觀測平台,但提供了**原料**:

- **Session JSONL**(`~/.claude/projects/<project>/<session>.jsonl`):每個 tool call、每個 message 完整落檔。可寫腳本聚合 token 用量、tool 失敗率、session 長度
- **Statusline**:即時顯示 context %、cost、token rate
- **`/cost` 指令**(若版本支援):查詢當前 session 累積成本
- **SessionEnd hook**:session 結束時送摘要(成本、tool counts、錯誤分類)到外部系統(Datadog / Grafana / 自家 DB),做跨 session 趨勢分析
- **失敗分類管線**:把 session JSONL 餵給專職分析 agent,聚合「最常被攔截的指令」「最容易讓 agent 卡住的工具」——這是 Ratchet 升級規則的最佳輸入源

實務上小型專案用 Statusline + 偶爾翻 JSONL 就夠;團隊規模上來才需要做正式遙測管線。

### 兩層式架構:全域 vs 專案

對於跨多專案的開發者,Claude Code 設定本身就有分層機制:

|層級|位置                               |放什麼                      |
|--|---------------------------------|-------------------------|
|全域|`~/.claude/`                     |個人通用偏好、危險指令攔截、慣用 commands|
|專案|repo 根目錄 `CLAUDE.md` + `.claude/`|專案架構、特定 hooks、領域 commands|

對應到面向 1(上下文)、面向 3(安全)、面向 4(工作流)的分層落實。

-----

## 六、技術分層:Vibe → Spec → Harness

Vibe Coding、Spec Coding、Harness Engineering 不是相互競爭的方案,而是**使用 harness 的不同方式**——對同一套底層工具(Cursor、Claude Code、Codex 等),不同情境下的紀律差異。

|層                  |核心問題        |優化目標  |紀律強度|
|-------------------|------------|------|----|
|Vibe Coding        |怎麼快速生成程式碼?   |生成速度  |低   |
|Spec Coding        |怎麼生成符合規格的程式碼?|規格對齊  |中   |
|Harness Engineering|怎麼讓系統長期可靠運行?|系統可信賴性|高   |

> **重要釐清**:Cursor、Claude Code、Codex、Aider、Cline **都是 harness framework**。同一個 Cursor,輕量試錯時是 Vibe,跟著 spec 文件嚴謹開發時是 Spec,搭配嚴格 hook + sub-agent 編排 + 觀測時就是 Harness Engineering。**框架是同一個,差別在你怎麼用它**。

三層是包含關係,不是替代:Vibe 是 Spec 的基礎(快速試錯找模式)、Spec 是 Harness 的核心輸入(把模式變成可執行規範)、Harness 讓 Vibe + Spec 真正落實(沒有它兩者都只是紙上)。

值得強調的是:**在 Harness Engineering 體系內,仍可使用 Vibe Coding 快速探索需求**。只是 Harness 會為這種探索劃定明確邊界(例如限制在 sandbox 與特定 sub-agent 內),避免探索結果變成無法收拾的「屎山程式碼」。

-----

## 七、落實考量

### Token 成本

Harness 的上下文注入機制會增加 Token 消耗,但 Harness Engineering 本身也提供成本優化手段:

- **KV-cache 優化**:透過穩定的上下文前綴設計、只追加的上下文結構、確定性序列化邏輯,可大幅降低 Token 成本(無需修改底層模型)
- **工具精簡原則**:移除非核心工具,減少執行步驟,實現「少工具、少 Token、高成功率」
- **Tool-call offloading**(見面向 1):大型工具輸出寫檔案,context 只留指針

### 適用邊界

> 以下適用邊界較偏向**組織導入**情境(企業/團隊規模)。個人開發者通常不需要前期完整評估,直接從 Claude Code 開始用、隨需要加 hook 即可。

**適合落實(滿足其一即可):**

- 任務複雜度高,單 Agent 無法覆蓋,需要多 Agent 協同
- 操作風險高,錯誤代價不可接受(財務、客戶資料、核心系統變更)
- 任務週期長,需要狀態管理與斷點恢復
- 合規要求明確,需要完整審計追蹤與人工確認節點

**不適合大規模投入:**

- 業務流程簡單確定,現有 RPA 方案運作良好
- 數位化基礎設施薄弱,無法支撐 Harness 的上下文工程與架構約束
- ROI 過低,Harness 的初期投入遠高於業務收益

### 模型夠強之後 Harness 還有用嗎?

Addy Osmani 在這個問題上的觀察是:

> **Harnesses Don't Shrink, They Move.**

模型升級**不會讓 harness 消失,而是讓它的責任位移**:

- 過去 hook 攔的是 `rm -rf` → 模型不再亂跑後,hook 改去攔更微妙的失敗(例如錯誤的 git 操作模式、危險的 SQL DDL)
- 過去 sub-agent 拆任務 → 模型 long-horizon 能力升級後,sub-agent 改去負責「驗證」而非「拆解」
- 過去 CLAUDE.md 條目寫「請逐步思考」 → 模型內建 reasoning 後,這些條目移除,但 CLAUDE.md 改載入更具體的領域知識
- 過去靠 hook 強制 typecheck → 模型自己會跑 typecheck 後,hook 改去檢查更高階的架構違規

換句話說,**每個 harness 元件都編碼一個對「模型現在做不到什麼」的假設**。當該假設過時,**元件職責改變但元件本身仍存在**。

這也是為什麼 harness 是 living system 而非 static config——更何況模型 post-training 過程會把流行的 harness pattern 學進去,造成模型對特定 harness 過擬合(training-loop feedback)。沒有「最佳 harness」,只有「適合當下這代模型的 harness」。

### 趨勢觀察:HaaS 與 harness convergence

- **Harness-as-a-Service(HaaS)**:產業重心從「建在 LLM API 上(自己拼 orchestration)」 → 「建在 harness framework 上(選一套穩定的 harness 來調)」。Claude Code、Cursor、Codex 等就是這種服務化 harness 的代表
- **Harness pattern 收斂**:頂級 coding agent(Claude Code、Cursor、Codex、Aider、Cline)的設計越來越像彼此,反而比模型差異更明顯——「編輯與讀取分離」「Plan Mode」「sub-agent」「lifecycle hook」這些 pattern 已是業界共識
- **實務含意**:**如果你發現 X 公司的 agent 比 Y 公司強,差距通常在 harness 工程,不是模型基礎能力**。這也意味著對 agent 採購方/使用方來說,評估重點該放在 harness 設計而非底層模型編號

-----

## 八、結語

如果一定要用一句話概括 Harness 的本質:

> **Harness Engineering 就是把 Agent 從「會呼叫工具的模型」,變成「能在約束中穩定完成任務的系統」。**

Addy Osmani 的版本更直接:**「If you're not the model, you're the harness.」**——你只要不是在訓練模型,你做的所有事就都在 harness 這一層。

最重要的三個詞不是 Agent、不是模型,而是:**約束、回饋、穩定**。真實世界裡的 agent 不是靠「聰明」活下來的,而是靠**有邊界、有結構、有狀態、有回饋、有保底**活下來的。

工程師的工作正在從「寫程式碼」轉向「**設計環境、指定意圖、建立回饋迴路,讓 agent 能可靠工作**」。Harness Engineering 並非全新概念,而是企業架構治理、DevOps、RPA 等已有實踐在 AI Agent 時代的自然延伸——它只是被系統化、被命名,讓那些一直在做、卻沒被明確指認的工程現實,終於擺到桌面上認真討論。

-----

## 九、來源與更新

### 主要來源

- **Addy Osmani**, *"Agent Harness Engineering"*(2026-05-10)— [https://x.com/addyosmani/status/2053231239721885918](https://x.com/addyosmani/status/2053231239721885918)
- **Vaibhav Trivedy**(@Vtrivedy10)— "harness engineering" 命名與核心定義
- **HumanLayer / @dexhorthy** — 「failures are configuration problems, not model problems」reframe
- **Anthropic** — context engineering 研究、Claude Code 官方文件
- **Fareed Khan** — Claude Code architecture breakdown([level up gitconnected](https://levelup.gitconnected.com/building-claude-code-with-harness-engineering-d2e8c0da85f0))
- **Fred Schott**(@FredKSchott)— [Flue framework](https://x.com/FredKSchott/status/2050274923852210397)

### 業界資料佐證(具體數字會隨後續實驗變動)

- LangChain Terminal Bench 2.0 實驗(優化 harness 不換模型,得分顯著提升)
- Vercel agent 工具精簡實驗(移除約八成工具,任務成功率反升)
- OpenAI Codex Agent 案例(少數工程師數月生成百萬行程式碼)

### 校對紀錄

- **2026-05-01**:初版,六大面向 + Claude Code mapping
- **2026-05-10**:對照 Addy Osmani《Agent Harness Engineering》整體 review
  - 新增 §三 核心心法(Ratchet / Working Backwards / Success Silent / 軟硬約束光譜)
  - 新增 面向 7 觀測與計量
  - 修正 模型升級論(Harnesses Don't Shrink, They Move)
  - 工具治理改寫為 Bash + 專用工具雙軌策略
  - 補入 tool description = prompt surface 安全角度
  - 補入 generation/evaluation 分離理由與 Loops vs Stop 區分
  - 加入 詞源致謝、frontmatter `last_reviewed`、本「來源與更新」段
  - §六 Vibe→Spec→Harness 重新定位為「使用 harness 的方式」而非「工具種類」

下次 review 觸發點:Claude Code 主版本變動、出現新的有名 harness pattern、模型世代跨越(例如下一代 reasoning model 大規模可用)。
