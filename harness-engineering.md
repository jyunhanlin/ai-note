---
title: Harness Engineering
tags: [agent, claude-code, harness, ai-engineering, prompt-engineering]
created: 2026-05-01
type: reference
status: living-document
---

# Harness Engineering:讓 AI Agent 從 Demo 走向生產的工程體系

> 一份綜合性的參考筆記。涵蓋定義、六大面向、技術分層,以及對應到 Claude Code 的具體 primitives。

## 目錄

- [前言:被命名之前已經存在的工程現實](#前言)
- [一、Harness 到底是什麼](#一harness-到底是什麼)
- [二、為什麼這個詞現在火了](#二為什麼這個詞現在火了)
- [三、Harness 的六大面向](#三harness-的六大面向)
  - [面向 1:上下文與記憶管理](#面向-1上下文與記憶管理)
  - [面向 2:工具治理](#面向-2工具治理)
  - [面向 3:安全與審批](#面向-3安全與審批)
  - [面向 4:工作流程編排](#面向-4工作流程編排)
  - [面向 5:回饋與狀態](#面向-5回饋與狀態)
  - [面向 6:架構約束與熵管理](#面向-6架構約束與熵管理)
- [四、對應到 Claude Code:Primitives Mapping](#四對應到-claude-codeprimitives-mapping)
- [五、技術分層:Vibe → Spec → Harness](#五技術分層vibe--spec--harness)
- [六、落實考量](#六落實考量)
- [七、結語](#七結語)

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

可以這樣定義:

> **Harness 是 Agent 運行時的「工程環境總和」,是讓模型在約束中穩定完成任務的整套運行時系統。**

這個「環境總和」至少包含:

1. 任務如何被表達
1. 上下文如何被組織
1. 工具如何被暴露、治理、攔截
1. 狀態如何被儲存、恢復、裁剪
1. 回饋如何回流給模型
1. 錯誤如何被分類、重試、升級
1. 安全邊界如何被建立
1. 系統如何驗證 agent 真的把事做好了

如果把 Agent 想成一個「會呼叫工具的大腦」,那 Harness 就是給這個大腦配的身體、感測器、護欄、操作台、流程制度、回執系統,以及出事後的保險。

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

幾組業界資料佐證了這個論點:

- **LangChain 實驗**:只優化 Harness、不換底層模型,寫程式 agent 在 Terminal Bench 2.0 得分從 52.8% 跳到 66.5%,排名從前 30 升到前 5
- **Vercel 實驗**:移除 80% 的 agent 工具,反而步驟更少、Token 消耗更低、任務成功率更高
- **OpenAI 案例**:3 名工程師 + Codex Agent,5 個月生成 100 萬行程式碼

這些資料共同指向一個結論:**Agent 的能力 = 模型 × 環境**。

這也意味著一件事:**驗證一套 harness 的方向是否正確,最好的方式不是用最新最強的模型測,而是用一個發布將近一年的舊模型測**。如果一個相對古早的模型在這套環境裡仍能做成事,那才說明工程方向是對的。

-----

## 三、Harness 的六大面向

### 面向 1:上下文與記憶管理

最基礎、也最容易翻車的一層。核心要對抗的問題叫 **context rot**:當關鍵內容落在上下文中間位置時,模型表現會下降 30%+。即使是百萬 token 的視窗,內容一多,指令遵循能力依舊會下降。

**要做的事:**

- **持久化指令文件**:repo 根目錄放 `CLAUDE.md` / `AGENTS.md`,寫架構、build/test 指令、命名約定。OpenAI 在程式碼庫散佈 88 個 `AGENTS.md`,agent 進入對應目錄時自動載入規則
- **作用域組裝**:組織級 / 使用者級 / 專案根 / 父目錄 / 子目錄分層載入,monorepo 必備
- **分層記憶**:精簡索引常駐 context(幾百行內),相關內容按需載入,完整歷史寫入磁碟
- **記憶整合(Dream Consolidation)**:背景跑「垃圾回收」邏輯,在空閒時定期去重、刪舊、重組結構
- **漸進式上下文壓縮**:新對話保留細節,稍舊內容做輕量總結,再往前的逐步折疊成短摘要——越久遠的資訊保留得越粗
- **動態提示詞組裝**:把 prompt 拆成多層(基礎模板 / 使用者偏好 / 專案上下文 / Skills / 工具描述 / 工具 JSON),不同新鮮度、不同優先級分開注入

關鍵心態的轉變:**提示詞不是靜態文件,而是動態組裝系統**。不能把所有來源的資訊全塞進一個大 prompt 裡,然後指望模型自己理解層級關係。必須先在工程上把層級關係建好,再交給模型。

### 面向 2:工具治理

只要 agent 開始接工具,複雜度會立刻上升。工具系統背後至少有四個問題:模型如何發現工具、參數如何驗證、風險如何分級、呼叫如何被統一攔截。

**要做的事:**

- **宣告式工具定義 DSL**:把工具寫成可驗證的 schema,而非裸 function
- **漸進式工具擴展**:預設只給少數常用工具(讀寫檔、搜尋),複雜工具按需載入
- **單一用途工具設計**:把 `cat` / `sed` / `grep` 拆成專用的讀檔、改檔、搜尋工具,邊界明確、權限好控
- **指令風險分類**:低風險自動放行、高風險才人工確認(read / write / execute / critical 四級)
- **統一編排層**:輸入參數驗證、審批攔截、結果裁剪、錯誤歸類四件事必須在這層做完

許多 Agent 一開始能跑、後來越來越不穩,根因是它們的工具系統不是「系統」,只是「工具列表」。Harness 的核心任務是把工具從「模型可以呼叫的能力」,提升成「可控、可審計、可演化的運行時能力」。

### 面向 3:安全與審批

只要 agent 能寫文件、跑 shell、改設定,它就不再是「聊天模型」,而是作業系統參與者。安全問題不是「以後再說」,而是第一天就存在。

**最關鍵的一個原則:**

> 安全不能只靠模型自覺。不能把「請不要執行危險指令」寫在 prompt 裡,然後假裝問題解決了。**真正的安全邊界,必須落在運行時**。

**三道防線:**

1. **子程序管理**:會話池、數量限制、輸出截斷、逾時終止、自動清理
1. **指令守衛**:指令解析、危險指令黑名單、system hint 回傳
1. **審批系統**:風險分級、審批模式、指紋快取、dangerous 模式保底

加上 **Human-in-the-loop 強制節點**:資金操作、資料去識別化、系統變更等高風險操作前強制暫停並等待人工確認。可以類比財務審批的「四眼原則」。

Demo 級別的 agent 靠的是「模型大概會聽話」;Harness 級別的 agent 靠的是「即便模型不聽話,系統也有邊界」。

### 面向 4:工作流程編排

核心就是一個詞——**分離**。把讀取和寫入拆開,把「查資料」和「改程式碼」的上下文拆開,把順序執行和並行執行拆開。

大多數 Agent 的預設做法是把這些事情混在一起,剛開始可能沒問題,但任務一複雜,品質很容易崩——所有東西都會堆在同一個上下文裡,研究內容、規劃討論、程式碼修改、日誌輸出全混在一起,等真正開始改程式碼時,很多無關資訊已經在干擾判斷。

**要做的事:**

- **探索-規劃-行動三段式**:權限逐步放開——先只讀模式探索結構 → 對齊規劃方案 → 才給執行權限
- **上下文隔離子 agent**:研究 agent 只能讀、規劃 agent 只設計、執行 agent 才有完整工具權限。每個子 agent 只接觸自己需要的資訊,避免被「流雜訊」污染
- **Fork-Join 並行**:跨檔案互不依賴的改動可以拆分,每個子 agent 在獨立的程式碼副本(例如 `git worktree`)裡跑,互不干擾,等都完成後再合併

代價是:多了協調成本,小任務會顯得「流程過重」,合併分支也可能比順序處理更難解。但對複雜、不熟悉的程式碼庫來說,這個取捨划算。

### 面向 5:回饋與狀態

Agent 之所以能持續做事,不是因為它會一直「想」,而是因為它會不斷收到回饋——工具執行結果、審批是否通過、輸出是否被截斷、搜尋結果是否過多、指令是否被拒絕、目前任務計畫是否更新、會話是否可恢復。

**最關鍵的一個原則:**

> 要把系統內部發生的事,翻譯成模型能消費的回饋語言。如果沒有這層翻譯,模型只能看到「失敗了」。有了它才能理解失敗是因為權限問題、是因為輸出被裁剪、是因為搜尋範圍太大、還是因為指令危險而被拒絕。

這直接決定了 agent 是否能做出下一步正確行動。

**要做的事:**

- **異常翻譯機制**:用結構化的 system_hint(例如 XML 標籤)承載異常狀態
- **Checkpoint 機制**:長時間運行任務中定期儲存狀態快照,讓 agent 從失敗點恢復而非從頭開始
- **確定性生命週期 hooks**:必須每次都做的事——改完程式碼跑 format、執行前做驗證、切換目錄時重新載入設定——絕對不能寫在 prompt 裡靠模型記。掛到 agent 生命週期的關鍵節點上自動執行

判斷準則:**凡是「不能出錯、不能漏」的事,都不該交給模型記,而應該交給系統保底**。

### 面向 6:架構約束與熵管理

最容易被忽略,但在長期運行中最關鍵。

**架構約束**的核心理念是:放棄 LLM 的軟性約束,用確定性規則引擎做硬性管控。具體做法包括 CI/CD pipeline 的自定 lint 規則、驗證架構模式的結構測試、清晰的模組邊界定義——agent 輸出結果必須通過「硬檢查」才能落實,違規直接攔截。這是企業級系統的永恆取捨:**放棄「生成任何東西」的靈活性,換取系統的可靠性**。

**熵管理**對抗的是另一個必然發生的問題:agent 一旦持續運行,系統一定會逐漸「變髒」——prompt 越來越長、rules 越來越舊、AGENTS.md 越來越像垃圾場、工具定義越來越多但沒人維護、會話歷史越來越重、memory 混入大量無效資訊。

> 當一切都重要時,一切都不重要。大而全的說明文件會迅速腐爛。

所以 Harness 不是「搭完就完了」的東西。它有一個長期任務:**持續做熵管理,讓系統不斷回到「對模型友好」的狀態**。

**要做的事:**

- 定期跑專職「垃圾回收 agent」:掃描文件矛盾、發現架構違規、清理技術債——這批 agent 不創造新功能,只做清潔工
- 過期規則失效機制
- 長上下文壓縮
- 低價值歷史退出 context
- 專案知識以可維護方式沉澱

-----

## 四、對應到 Claude Code:Primitives Mapping

Claude Code 本身就是一個 harness——打開 terminal 輸入 `claude` 那一刻,就已經在一個 perception-action-verification 環境裡工作了。它的 extension layer 提供了一組 primitives,讓使用者把預設 harness 塑造成「自己的 harness」。

### 速查表

|面向         |主要 Primitives                                                  |
|-----------|---------------------------------------------------------------|
|1. 上下文與記憶  |`CLAUDE.md`、`.claude/rules/`、Skills、`/compact`、Imports         |
|2. 工具治理    |MCP servers、Permissions、`allowedTools` / `deniedTools`         |
|3. 安全與審批   |`/sandbox`、PreToolUse hook、Permission modes                    |
|4. 工作流程編排   |Plan Mode、Sub-agents (Task tool)、Slash commands                |
|5. 回饋與狀態   |Hooks(13 個 lifecycle events)、Session logs、`--resume`、Statusline|
|6. 架構約束與熵管理|PostToolUse hooks、Stop hooks、清潔 sub-agent                      |

### 4.1 上下文與記憶 → CLAUDE.md + Rules + Skills

|Primitive                 |對應做法                                                              |
|--------------------------|------------------------------------------------------------------|
|`CLAUDE.md`(專案級)          |repo 根目錄,session 啟動時自動載入。寫架構、build/test 指令、命名約定、已知陷阱              |
|`~/.claude/CLAUDE.md`(全域) |個人通用偏好(commit convention、code review 標準、禁止行為)                     |
|`.claude/rules/`          |條件式規則目錄,用 `paths:` glob 指定「只有碰到某些檔案時才載入」。前端/後端規則可分開               |
|Skills (`.claude/skills/`)|可重用的指令集封裝領域知識,Claude 自動判斷何時啟用                                     |
|`@import` 語法              |CLAUDE.md 可 `@filename.md` 匯入其他文件,實現作用域組裝(對應 88 個 AGENTS.md 的散佈策略)|
|`/compact` 指令             |手動觸發上下文壓縮;auto-compaction 在視窗接近滿時自動執行                             |

### 4.2 工具治理 → MCP Servers + Permissions

```json
// .claude/settings.json
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

### 4.3 安全與審批 → Sandbox + Hooks + Permission Modes

**`/sandbox` 指令**:啟用 OS 級隔離。

- macOS:Seatbelt(開箱即用)
- Linux/WSL:bubblewrap + socat(需要安裝)
- 設定 allowed paths、allowed hosts,內部測試可減少 84% 的 permission prompt

**PreToolUse hook 範例**:

```bash
#!/bin/bash
# .claude/hooks/pre-tool-use.sh
input=$(cat)
tool=$(echo "$input" | jq -r .tool_name)
args=$(echo "$input" | jq -r .tool_input.command)

if [[ "$tool" == "Bash" && "$args" =~ rm[[:space:]]+-rf ]]; then
  echo '{"decision": "block", "reason": "rm -rf blocked"}'
  exit 0
fi
```

**Permission modes**:

- `default`:每個 tool 都問
- `acceptEdits`:檔案編輯自動通過
- `plan`:純規劃模式,完全不能執行
- `bypassPermissions`:全部跳過,僅建議在 sandbox 內使用

社群最推薦的「既安全又少打擾」設定:`/sandbox` + `acceptEdits`。

### 4.4 工作流程編排 → Plan Mode + Sub-agents + Slash Commands

|Primitive                 |用途                                                   |
|--------------------------|-----------------------------------------------------|
|Plan Mode(Shift+Tab)      |純規劃模式,Claude 只讀不寫,直到 approve plan 才執行。對應「探索-規劃-行動三段式」|
|Sub-agents(Task tool)     |把任務委派給隔離的 Claude 實例,各自有獨立 context                    |
|`.claude/agents/`         |自訂 sub-agent,設定 system prompt 和 allowed tools       |
|`.claude/commands/`       |Custom slash commands,封裝重複工作流程                        |
|Git worktree + 多 Claude 實例|對應 Fork-Join 並行,每個 worktree 跑獨立 session              |

常見 sub-agent 模式:

- **Researcher**:只做研究,不污染主 context
- **Reviewer**:檢查實作,獨立判斷
- **Cleaner**:做熵管理(見 4.6)

常見 slash commands:

- `/review`:只看 bug 和 security
- `/plan`:先產出 plan 到 `plans.md` 等 approve
- `/setup`:一鍵初始化開發環境

### 4.5 回饋與狀態 → Hooks + Session Logs + Statusline

**13 個 lifecycle hooks**(節錄關鍵):

|Hook              |用途                                 |
|------------------|-----------------------------------|
|`PreToolUse`      |tool 執行前攔截                         |
|`PostToolUse`     |tool 完成後驗證(format、type check)      |
|`UserPromptSubmit`|使用者送出訊息前                           |
|`Stop`            |agent 結束回應時觸發最終驗證                  |
|`SubagentStop`    |sub-agent 結束時                      |
|`PreCompact`      |上下文壓縮前(備份 transcript 用)            |
|`SessionStart`    |session 開始(載入 git diff、open issues)|
|`SessionEnd`      |session 結束                         |
|`Notification`    |通知事件                               |

**Session logs(JSONL)**:每個 session 完整記錄落在 `~/.claude/projects/`,可事後分析 agent trace。

**`--resume`**:恢復先前 session,對應 Checkpoint 機制。

**Statusline 設定**(`.claude/statusline.sh`):自訂底部狀態欄,顯示 context 用量、git branch、cost、token rate——把系統內部狀態翻譯成可消費的回饋。

**output-style 自訂**:設計 Claude 回應結構,例如「永遠先說做了什麼、再說結果」這種結構化回饋格式。

### 4.6 架構約束與熵管理 → PostToolUse + 清潔 Sub-agent

**PostToolUse 跑硬檢查**:

```bash
#!/bin/bash
# .claude/hooks/post-tool-use.sh
input=$(cat)
tool=$(echo "$input" | jq -r .tool_name)

if [[ "$tool" == "Edit" || "$tool" == "Write" ]]; then
  npm run lint && npm run typecheck
  if [ $? -ne 0 ]; then
    echo '{"decision": "block", "reason": "lint or typecheck failed"}'
    exit 2
  fi
fi
```

exit code 2 會把錯誤回傳給 Claude,讓它修(對應「翻譯成回饋語言」)。

**Stop hook 跑最終驗證**:agent 認為自己做完時,觸發完整測試,沒過就要求繼續。

**定期清潔 Sub-agent**:在 `.claude/agents/cleaner.md` 設計專職 maintenance agent,定期跑:

- 掃描 CLAUDE.md 中過期的指令
- 偵測 `.claude/rules/` 中互相矛盾的規則
- 列出超過 N 行的 prompt 內容建議壓縮
- 整理 commands 目錄,合併重複功能

### 兩層式架構:全域 vs 專案

對於跨多專案的開發者,Claude Code 設定本身就有分層機制:

|層級|位置                               |放什麼                      |
|--|---------------------------------|-------------------------|
|全域|`~/.claude/`                     |個人通用偏好、危險指令攔截、慣用 commands|
|專案|repo 根目錄 `CLAUDE.md` + `.claude/`|專案架構、特定 hooks、領域 commands|

對應到面向 1(上下文)、面向 3(安全)、面向 4(工作流)的分層落實。

-----

## 五、技術分層:Vibe → Spec → Harness

Vibe Coding、Spec Coding、Harness Engineering 不是相互競爭的方案,而是層層疊加、向上包含的技術棧:

|層                  |核心問題        |優化目標  |典型工具                 |
|-------------------|------------|------|---------------------|
|Vibe Coding        |怎麼快速生成程式碼?   |生成速度  |Cursor、Openclaw      |
|Spec Coding        |怎麼生成符合規格的程式碼?|規格對齊  |Claude Code + Spec 文件|
|Harness Engineering|怎麼讓系統長期可靠運行?|系統可信賴性|OpenAI Codex Harness |

三者是包含關係,不是替代:

- **Vibe 是 Spec 的基礎**:先用 Vibe 快速試錯找感覺,把穩定模式抽成 Spec
- **Spec 是 Harness 的核心輸入**:Harness 裡的約束、規則、上下文 = 把 Spec 變成可執行系統。沒有 Spec,Harness 就是空殼
- **Harness 讓 Vibe + Spec 真正落實**:沒有 Harness,Vibe 就是純玩具不敢上生產;Spec Coding 只是紙上規範,AI 依然會亂執行、崩、不可恢復

值得強調的是:**在 Harness Engineering 體系內,仍可使用 Vibe Coding 快速探索需求**。只是 Harness 會為這種探索劃定明確邊界,避免探索結果變成無法收拾的「屎山程式碼」。

-----

## 六、落實考量

### Token 成本

Harness 的上下文注入機制會增加 Token 消耗,但 Harness Engineering 本身也提供成本優化手段:

- **KV-cache 優化**:透過穩定的上下文前綴設計、只追加的上下文結構、確定性序列化邏輯,可將 Token 成本降低 90%(無需修改底層模型)
- **工具精簡原則**:移除非核心工具,減少執行步驟,實現「少工具、少 Token、高成功率」

### 適用邊界

**適合落實(滿足其一即可):**

- 任務複雜度高,單 Agent 無法覆蓋,需要多 Agent 協同
- 操作風險高,錯誤代價不可接受(財務、客戶資料、核心系統變更)
- 任務週期長,需要狀態管理與斷點恢復
- 合規要求明確,需要完整審計追蹤與人工確認節點

**不適合落實:**

- 業務流程簡單確定,現有 RPA 方案運作良好
- 數位化基礎設施薄弱,無法支撐 Harness 的上下文工程與架構約束
- ROI 過低,Harness 的初期投入遠高於業務收益

### 模型夠強之後還需要 Harness 嗎?

Harness 的價值存在「模型能力閾值」:

- **低於閾值**:模型推理能力不足,任何 Harness 都無法彌補
- **高於閾值**:模型可獨立完成複雜任務,Harness 的大部分複雜性將不再必要

但在目前模型能力下,沒有任何一個 AI Agent 能可靠完成所有企業複雜任務,多 Agent 的細分與協同是必然選擇,Harness Engineering 則是解決多 Agent 治理、安全、合規問題的核心方案。

-----

## 七、結語

如果一定要用一句話概括 Harness 的本質:

> **Harness Engineering 就是把 Agent 從「會呼叫工具的模型」,變成「能在約束中穩定完成任務的系統」。**

最重要的三個詞不是 Agent、不是模型,而是:

- **約束**
- **回饋**
- **穩定**

真實世界裡的 agent,不是靠「聰明」活下來的,而是靠**有邊界、有結構、有狀態、有回饋、有保底**活下來的。

模型當然重要,但模型更像發動機。真正決定這台車能不能上路、能不能跑長途、會不會翻車的,是底下那整套工程環境——也就是 Harness。

工程師的工作正在從「寫程式碼」轉向「設計環境、指定意圖、建立回饋迴路,讓 agent 能可靠工作」。在這個轉折點上,**那些複雜的技術名詞終將消解,但「讓技術服務於業務、讓智能體可控可靠」的核心訴求永遠不變**。

本質上,Harness Engineering 並非全新概念,而是企業架構治理、DevOps、RPA 等已有實踐在 AI Agent 時代的自然延伸。它只是被系統化、被命名,形成了行業通用的討論框架——讓那些一直在做、卻沒被明確指認的工程現實,終於擺到桌面上認真討論。
