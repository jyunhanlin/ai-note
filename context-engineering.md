---
title: Context Engineering（Claude 5 世代）
tags: [agent, claude-code, context-engineering, prompt-engineering, ai-engineering]
created: 2026-08-02
last_reviewed: 2026-08-02
type: reference
status: living-document
sources:
  - Thariq（@trq212, Anthropic Claude Code）— "The new rules of context engineering for Claude 5 models"（2026-07-25）
  - 本機驗證 — Claude Code 系統提示詞現況、`claude --help`、binary 字串、記憶目錄位置（2026-08-02 實測）
---

# Context Engineering：Claude 5 世代的新規則

> 補 repo 的缺口欄位。[harness-engineering.md](./harness-engineering.md) §一 的三分表把 Context Engineering 列為獨立一欄（Prompt = 怎麼說／Context = 給什麼資訊／Harness = 在什麼環境做事），但這一欄散在 §四 面向 1 與 §五 5.1 兩處。本篇補上，並鎖定一個特定角度：**模型世代之間的 delta**。

## 目錄

- [前言：一個可驗證的數字](#前言一個可驗證的數字)
- [一、Unhobbling：過度約束的成本科目變了](#一unhobbling過度約束的成本科目變了)
- [二、六組 then → now](#二六組-then--now)
  - [2.1 給規則 → 讓它判斷](#21-給規則--讓它判斷)
  - [2.2 給範例 → 設計介面](#22-給範例--設計介面)
  - [2.3 全部前載 → 漸進揭露](#23-全部前載--漸進揭露)
  - [2.4 重複強調 → 簡單的工具描述](#24-重複強調--簡單的工具描述)
  - [2.5 用 CLAUDE.md 當記憶 → 自動記憶](#25-用-claudemd-當記憶--自動記憶)
  - [2.6 簡單 spec → 豐富 reference](#26-簡單-spec--豐富-reference)
- [三、分層配置：哪一層該放什麼](#三分層配置哪一層該放什麼)
- [四、套到自己 repo 的判別法](#四套到自己-repo-的判別法)
- [五、與本 repo 其他筆記的對照](#五與本-repo-其他筆記的對照)
- [六、來源與更新](#六來源與更新)

### 閱讀路徑建議

- **只想知道要改什麼**：§二 的六組表 → §四 判別法
- **想理解為什麼**：§一（2.1 與 2.4 是它的直接推論；其餘四組各有各的理由）
- **想知道跟既有筆記怎麼接、哪裡打架**：§五

> ⚠️ **本篇有保鮮期，而且是刻意的。** 其他筆記寫的是心法（跨世代成立），這篇寫的是「Claude 5 世代 vs 前一代」的差分。差分本身下一代就會再變一次——這正是 [ihower 筆記 §八](./ihower-harness-engineering.md#八收尾model-harness-fit-與會過期的-harness第-8-篇)「harness 會過期」講的事，只是這次過期的是 context 層而不是 workflow 層。

---

## 前言：一個可驗證的數字

Anthropic 自己揭露：針對 Opus 5 / Fable 5 這代模型，**Claude Code 的系統提示詞砍掉 80% 以上，內部 coding eval 沒有可測量的退步**。

先講清楚 context 跟 prompt 差在哪，因為整篇的難處都從這裡來：prompt 是一次性的、可以講得很具體；**context 要跨很多次請求通用，所以不能具體**——你在寫它的時候並不知道使用者下一句會問什麼。「不知道對方要問什麼，卻要先把話講好」就是 context engineering 的本質困難。

這個 80% 之所以值得當起點，是因為它**不是主張、是已出貨行為的說明**，而且可以在本機對得上（見 §六 驗證表）：文中引的新版系統提示詞句子、deferred tool loading 機制、auto-memory，在當下的 Claude Code 裡都能直接觀察到。

同時把反方掛在前面，這篇是第一方在講自家模型：

- 「eval 沒退步」是在 **Anthropic 的 coding eval** 上量的，不是你的任務分布。你自己刪之前該有自己的 eval——這正是 ihower 系列的結論「eval 和 judge 永不過期」在這裡的直接應用
- 原文**沒有拆解這 80% 的組成**，所以「刪掉的是哪些東西」只能從它舉的例子推，不能當清單用
- 「模型判斷力夠好」的證據是內部評測，不是你的 codebase。刪約束的正確姿勢是**觀察後刪**，不是照抄百分比

---

## 一、Unhobbling：過度約束的成本科目變了

**Unhobbling** 是原文的用詞——把先前為了保底而套在模型身上的限制器拆掉。整節的樞紐只有一句：**約束的成本從「模型不遵守」變成「模型要調和」。**

| 世代     | 不給硬規則會怎樣          | 給硬規則會怎樣                                 |
| -------- | ------------------------- | ---------------------------------------------- |
| 舊世代   | 做錯（要吃這個 tradeoff） | 安全，代價是少數情境被規則綁死做出錯的選擇     |
| Claude 5 | 多半能從周遭脈絡推對      | **規則之間互相打架，模型得先花推理去排解衝突** |

Anthropic 讀自家內部使用軌跡時看到的典型衝突，是 system prompt、skill、使用者指令三方對撞——例如一邊說文件「看情況留」、另一邊說「不准加註解」，使用者又說「照舊版那樣能動就好」。（原文的三方對撞圖標為**示意**，非任何真實 prompt 的逐字引用。）模型有能力調和，但**調和不是免費的**，它吃掉的是本來要拿去解題的推理預算。

於是原本「規則越多越安全」的直覺反過來了。文中的置換是很好的示範——舊版是一條**寫死行為**的預設值（預設不寫註解、最多一行、除非使用者要求否則不產生規劃文件），新版換成一條**指向脈絡**的原則：

> Write code that reads like the surrounding code: match its comment density, naming, and idiom.
>
> — Anthropic, Claude Code 新版系統提示詞（原文發表於 2026-07-25）

差別不在寬鬆或嚴格，在於**約束的錨點從「我規定的絕對值」移到「你眼前的相對值」**。後者不會跟使用者的偏好、跟特定檔案的實際需要打架，因為它本來就叫模型去看那些東西。

> 對照 [harness-engineering.md 心法 4](./harness-engineering.md#心法-4軟硬約束光譜)：軟硬光譜講的是「這條規則該升到哪一層（prompt / hook / CI）」。這裡是**另一條正交的軸**：這條規則**該不該用絕對值寫死**。一條規則可以留在軟約束層，但把措辭從絕對值改成脈絡錨點——心法 4 沒涵蓋這件事。（harness §四 面向 1 已經宣告過一條 compaction vs reset 的模型相依光譜，本篇這條是再一條。）

---

## 二、六組 then → now

| Then（已成迷思）    | Now                | 一句話                                            |
| ------------------- | ------------------ | ------------------------------------------------- |
| 給規則              | 讓它判斷           | 錨點從絕對值換成周遭脈絡                          |
| 給範例              | 設計介面           | few-shot 會限縮探索空間；改把語意編進參數         |
| 全部前載            | 漸進揭露           | skill / deferred tool，要用才展開                 |
| 重複強調            | 簡單的工具描述     | 用法只留在 tool description，system prompt 不重複 |
| 用 CLAUDE.md 當記憶 | 自動記憶           | `#` hotkey → Claude 自己存                        |
| 簡單 spec           | 豐富 reference     | markdown 之外：artifact、測試、rubric、他處程式碼 |

### 2.1 給規則 → 讓它判斷

見 §一。原文的作法是把「寫死行為的預設值」換成「指向脈絡的原則」，並說明舊規則之所以存在，是因為舊模型少了它會大量寫錯註解——那個 tradeoff 當時是划算的，現在不划算了。

> 實務動作（**本篇的延伸，非原文明說**）：把 CLAUDE.md / skill 裡所有「Never / Always / 一律」開頭的條目挑出來，逐條問「這是在防模型犯錯，還是在表達我的偏好？」——判別法見 §四。

### 2.2 給範例 → 設計介面

過去 tool 使用的第一守則是給範例。這代模型反過來：**範例會把它框進一個特定的探索空間**。

替代做法是把該表達的東西編進**介面本身**。文中舉 TodoWrite 為例——status 只要列成 `pending` / `in_progress` / `completed` 的 enum，本身就在暗示怎麼用；再加一條「同時只保留一個 in_progress」，就把要的行為定義出來了。設計時該問的是「參數夠不夠表達力」，不是「範例夠不夠多」。

> ⚠️ **範圍要講精確——原文其實有劃界，是我一開始讀漏了。** 原文把這條**限定在 tool 使用**（原句是「tool usage 的第一守則」），替代方案也限定在 **tools / scripts / files 的設計**。它**完全沒有談** CLAUDE.md 或 skill 裡的範例，也沒談單次請求內的格式範例。
>
> 所以 repo 內兩處「疑似衝突」都要標成**本篇的外推、未驗證**：
>
> - [prompt.md](./prompt.md) §Few-Shot Learning — 單次請求鎖輸出格式，離原文的射程最遠
> - [harness-engineering.md §五 5.5](./harness-engineering.md) 的「few-shot 校準」——給 evaluator 幾個評分範例防 score drift。這條是**常駐**在 evaluator context 裡的範例，離原文的射程比 prompt.md 近，但原文談的是 tool 而不是 evaluator
>
> 兩處都沒有實測基礎可以斷言該刪。已在 prompt.md 加註記；harness 那條暫不動。

### 2.3 全部前載 → 漸進揭露

Claude Code 自己的兩個做法：

- **skill 化**：code review、verification 這類「不常用但用到時很關鍵」的內容從 system prompt 抽成 skill，需要才叫
- **deferred tool loading**：工具定義預設不進 context，agent 得先用 `ToolSearch` 搜到才能用。好處是工具數量可以往上加而不吃 context

對自己的 CLAUDE.md / SKILL.md，對應的動作是**別把它當百科全書**。「怕 Claude 找不到所以先全部寫進去」是這篇點名的迷思——原文的建議是考慮改成**一棵可以按需載入的檔案樹**。

> 這一格跟 [harness-engineering.md 面向 1](./harness-engineering.md#面向-1context-脈絡) 的「分層記憶」「動態提示詞組裝」是同一件事，那邊已經寫得比原文細（含 tool-call offloading、compaction vs reset）。本篇不重複，只補 deferred tool loading 這個新 primitive。

### 2.4 重複強調 → 簡單的工具描述

舊模型有兩個傾向：指令有時要重複才吃得住、context 尾端的指令比開頭的更容易被遵守。所以過去 system prompt 會把工具用法再講一次，跟 tool description 重複。

現在可以刪掉重複，**工具怎麼用只寫在 tool description 裡**。這條對自建 agent 的人比對 Claude Code 使用者更有用。

> 順帶釐清一個容易混淆的地方：原文另外展示了把一段冗長的 tool 說明收斂成精簡描述的前後對照，但那是 **tool description 那一層**的事——**不在 §前言 那個「系統提示詞砍 80%」裡面**。兩件事同期發生、方向一致，但不是同一個數字。

> 呼應 [harness-engineering.md 面向 2](./harness-engineering.md#面向-2tools-工具) 的「工具描述是 prompt surface」——那邊講的是安全面（來路不明的描述＝注入面），這裡補上職責面：description 是工具用法的**唯一**歸屬地。

### 2.5 用 CLAUDE.md 當記憶 → 自動記憶

`#` hotkey 寫進 CLAUDE.md 的做法，被 Claude 自動存記憶取代——它自己判斷哪些跟工作、跟你有關，然後存起來。

原文沒說記憶存在哪，但這件事在本機看得到（**基礎：2026-08-02 於本機觀察，非原文內容**）：記憶落在 `~/.claude/projects/<專案>/memory/`，**per-project**、以專案路徑分槽，而且**在 git repo 之外**。

這個位置直接決定了它能取代什麼、不能取代什麼：

- **能取代**：你個人在這個專案累積的偏好與踩坑記錄
- **不能取代**：交付 git、給團隊共同維護的 CLAUDE.md——自動記憶落在 repo 外、不進版控，團隊拿不到（這是檔案位置能證明的範圍；它有沒有其他流向不在本機可觀測之列）

> ⚠️ 這條讓 [boris-cherny-tips.md](./boris-cherny-tips.md) 的 §CLAUDE.md 部分過時。要**分開看兩件事**：`#` hotkey 那套記憶用法確實被取代了；但該節主推的「每次犯錯就寫進 CLAUDE.md、無情迭代」比較微妙——**錯誤驅動的迭代產出的正好是原文要的 gotcha**，真正被點名的迷思是「把可能用到的都先寫進去」。方向不是全錯，是**該收斂到 gotcha、而且要停止無限增長**。已在該節加時效註記。

### 2.6 簡單 spec → 豐富 reference

Plan mode 過去靠 markdown 計畫檔。現在 reference 可以是：

- **HTML artifact**（artifacts 功能產生的 HTML）
- **測試套件**——一份夠細的測試就是一份 spec
- **別的 codebase 裡的一個函式**——要移植時直接指過去
- **rubric**——把「什麼叫好的 API 設計」寫成評分標準，配 dynamic workflows 開 verifier agent 去驗

一條實用原則：**能用程式碼表達就別用文字描述**。原文的說法是 HTML mockup 一般會比「描述那個設計」或「截圖」產生更好的結果——高保真、而且是模型最熟的語言。

> rubric + verifier agent 這條，[harness-engineering.md §五 5.5](./harness-engineering.md) 的「評分標準設計」已經寫過（拆維度／加權弱項／硬門檻／few-shot 校準／措辭引導）。本篇的新資訊只是：**Anthropic 現在把 rubric 歸類成 reference 的一種**（原句：rubrics are another form of references），也就是它是 context 的一部分，不只是驗證機制。

---

## 三、分層配置：哪一層該放什麼

原文給的堆疊（由上而下）：

```
Your prompt          ← 一次性、最具體
References           ← @ 提及的檔案、spec、mockup、codebase、artifact
System prompt        ← 產品層；用 Claude Code 的話大概不會去改
CLAUDE.md            ← 專案層
Skills               ← 領域／團隊的意見與最佳實踐
Memory               ← 跨 session、自動累積
```

> 原圖只是這六張卡片由上而下疊著，沒有標軸。原文明講的只有「prompt 可以最具體、context 不行」；其餘各層之間的具體程度排序是本篇的讀法。

各層的配置原則：

| 層            | 該放                                                                                     | 不該放                                      |
| ------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------- |
| System prompt | 產品脈絡：這是什麼產品、在做什麼。自建 harness 的人力氣主要花這裡                        | ——（Claude Code 使用者大概不會改到）        |
| CLAUDE.md     | **保持輕量**：先簡述這個 repo 是做什麼的，然後**大部分 token 花在 gotchas**——例如「型別全部集中在單一檔案，別處沒有」這種光看檔案系統推不出來的事 | Claude 掃一眼 repo 就知道的「顯而易見的事」 |
| Skills        | 你／團隊／產品**特有**的意見、知識、最佳實踐；長的要拆成多檔漸進揭露                      | 過度約束（除非是極重要的區域）              |
| References    | 高保真、可直接讀的東西；優先給程式碼形式                                                  | 用文字轉述本來就有檔案的東西                |

兩條原文明講、容易漏掉的細節：

- CLAUDE.md 的判準是**「這件事 Claude 自己看得出來嗎」**，看得出來的就刪。但**「這個 repo 是幹嘛的」那句要留**——原文明確要求先簡述用途，那不在「顯而易見」的範圍
- 細節走漸進揭露：例如你有好幾條獨特的驗證步驟，就**做成一個 verification skill，再從 CLAUDE.md 指過去**

---

## 四、套到自己 repo 的判別法

Anthropic 出了 `/doctor` 幫忙瘦身——原文兩個名字都提了（CLI 的 `claude doctor` 與 session 內的 `/doctor`），說可以拿來 rightsize skills 與 CLAUDE.md。但**它沒有區分兩者能做什麼：「session 內是完整健檢、可以順手修，CLI 版唯讀」出自本機 `claude --help`，不是原文**（見 §六）。

工具跑完之後，該留該刪還是得自己判。

**以下這條分界線是本 repo 的延伸，原文沒有明說：**

每條 context 規則都編碼一個假設，但假設有兩種：

| 假設種類     | 長相                               | 模型變強時                          |
| ------------ | ---------------------------------- | ----------------------------------- |
| **能力假設** | 「模型自己做不到 X，所以我規定 X」 | 過期 → 刪                           |
| **偏好假設** | 「我要的是 Y 不是 Z」              | **永不過期** → 留                   |

判別問句：**「模型下一版變強，這條會自動變成多餘的嗎？」**

- 會 → 能力腳手架，列入下次模型升級時的刪除候選
- 不會 → 品味／偏好，留著

被砍掉的 80% 幾乎全是第一種（**這是用本篇的框架去看原文舉的例子，原文自己沒有這樣分類**）。第二種原文並沒有叫你刪——它說 skill 最適合編碼的，就是「你、你的團隊、你的產品特有的意見、知識與最佳實踐」。但同一節也提醒**別把 skill 寫成過度約束**（除非是極重要的區域），所以「留」不等於「寫得更兇」。

**模型再強也猜不到你要什麼，它只會越來越擅長猜你能力上做不到什麼。**

其中「能力假設」那半並不新——[harness §七](./harness-engineering.md) 早就寫過「每個 harness 元件都編碼一個對『模型現在做不到什麼』的假設」，ihower §八 也引了 Anthropic 同樣的話。**本篇新加的只有「偏好假設」那一欄**，以及它帶來的推論：刪約束時要分兩堆看，不能整批砍。

> 這條線跟 [ihower 筆記 §八](./ihower-harness-engineering.md#八收尾model-harness-fit-與會過期的-harness第-8-篇) 的「⏳ 會過期 vs ♾️ 不會過期（eval 和 judge）」是同一條線的兩端：他從**驗證**角度講（定義什麼叫做好，模型吸收不了），這裡從**指令**角度講（表達你要什麼，模型猜不到）。合起來就是：腳手架會被吸收，**意圖不會**。

**實務節奏**：ihower §八 的作法是每次模型升級（含小版本）就問一次「What can I stop doing?」——本篇的表格是給這個問句用的**篩子**：它補的是「該砍哪一條」的判準，而不是「要不要砍」的提醒。

> ⚠️ **別讀成單向縮減。** harness §七 明講位移／淘汰／新生**三條路都會發生**，ihower §八 的講法是「不是變少，是該做的會變」。而且「加」那一半 harness 早就有了——[心法 1 Ratchet](./harness-engineering.md#心法-1the-ratchet--錯誤的單向累積)（加）與**熵管理**（減）本來就配成完整迴圈。**本篇不是來補空位的，它補的是熵管理缺的那個判準：該減的到底是哪一條。**

---

## 五、與本 repo 其他筆記的對照

| 本篇的內容                     | repo 對應                                                     | 關係                                                                                        |
| ------------------------------ | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| 六組 then→now、80% 實證        | [harness](./harness-engineering.md) §七                       | 同結論。§七 原有的 Opus 4.5→4.6 案例同樣是 Anthropic 來源但**沒有公開數字**；本篇補上帶數字的 |
| 過度約束＝要調和的成本         | harness §三 心法 4（軟硬約束光譜）                            | **正交補充**：心法 4 管「升到哪一層」，本篇管「該不該寫成絕對值」                             |
| 漸進揭露、檔案樹               | harness §四 面向 1（分層記憶／動態組裝）                      | 大量重疊，面向 1 寫得更細。本篇只補 deferred tool loading                                    |
| tool description 是單一來源    | harness §四 面向 2（description = prompt surface）            | 那邊講安全面，本篇補職責面                                                                   |
| rubric 是 reference 的一種     | harness §五 5.5 評分標準設計                                  | 那邊當驗證機制寫，本篇補上「它同時是 context」的定位                                         |
| 能力假設 vs 偏好假設           | [ihower](./ihower-harness-engineering.md) §八（會過期 vs eval/judge） | 同一條線的兩端：他講驗證，這裡講指令。能力那半 repo 已有，**新的只有偏好那半**                |
| 反 few-shot（射程未定）        | [prompt.md](./prompt.md) §Few-Shot；harness §五 5.5 few-shot 校準 | **兩處疑似衝突，都未驗證**（見 §2.2）。prompt.md 已加註記，harness 那條暫不動                |
| 自動記憶（per-project、不進 git）| [boris-cherny-tips.md](./boris-cherny-tips.md) §CLAUDE.md      | 該節的「無情迭代」需收斂到 gotcha（見 §2.5）；`#` hotkey 那套用法整體已被取代（該節未提）。已加時效註記 |
| CLAUDE.md 只放 gotcha          | harness §四 面向 1「持久化指令文件」與 §五 5.1 表格的 `CLAUDE.md`（專案級）列 | **未解衝突（半個）**：兩處都說 CLAUDE.md 寫「架構、build/test 指令、命名約定」，而架構與命名約定多半是 Claude 看 repo 就推得出來的——正是 §三 說該刪的類別。但 5.1 那列已經含「已知陷阱」，方向本來就對了一半；真正要重估的是「架構／命名約定」這兩項。留待 harness 下次 review |

一句話定位：harness 筆記回答「模型在什麼環境做事」，本篇回答「**這代模型該給它看什麼、不該給它看什麼**」——而且答案跟上一代相反的地方比相同的多。

---

## 六、來源與更新

### 主要來源

- **Thariq**（@trq212, Anthropic Claude Code）— _"The new rules of context engineering for Claude 5 models"_（2026-07-25）— [x.com/trq212/status/2080710971228918066](https://x.com/trq212/status/2080710971228918066)
- 文中外連（尚未逐篇讀）：
  - Anthropic — [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
  - Anthropic — [A harness for every task: dynamic workflows in Claude Code](https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code)
  - Anthropic — [A field guide to Claude Fable: finding your unknowns](https://claude.com/blog/a-field-guide-to-claude-fable-finding-your-unknowns)
  - Thariq — [如何 prompt 這代模型（前作）](https://x.com/trq212/status/2073100352921215386)

### 本機驗證紀錄（2026-08-02 實測）

文章多數宣稱可在當下的 Claude Code 直接對上，因此定性為**已出貨行為**而非宣傳：

| 宣稱                          | 驗證方式                                                                | 結果                |
| ----------------------------- | ----------------------------------------------------------------------- | ------------------- |
| 新版系統提示詞的置換句        | 該句逐字出現在當前 session 的系統提示詞裡；binary v2.1.220 也搜得到 4 次 | ✅ 觀測到           |
| **舊規則其實還在 binary 裡**  | 「default to writing no comments…」在 v2.1.220 仍搜得到 1 次            | ⚠️ 見下方說明        |
| deferred tool loading         | 系統提示詞含 deferred tools 清單＋`ToolSearch` 取用機制                 | ✅ 觀測到           |
| auto-memory 存在              | `claude --help` 的 `--bare` 說明把 auto-memory 列為可跳過的功能         | ✅ 觀測到           |
| auto-memory 位置（原文沒說）  | `~/.claude/projects/<專案>/memory/`；per-project 分槽，且在 git repo 外  | ✅ 觀測到           |
| `/doctor` 可修、CLI 版唯讀    | `claude --help` 的 `doctor` 說明文字                                    | ✅ 觀測到           |
| `/checkup` 是 `/doctor` 的別名 | binary v2.1.220：`name:"doctor",aliases:["checkup"]`                    | ✅ 觀測到           |
| 系統提示詞砍 80%、eval 未退步 | 無法驗證（需要新舊版本比對＋Anthropic 內部 eval）                       | ⚠️ 僅第一方說法     |

> ⚠️ **別把「置換」讀成「刪除」。** 新舊兩條規則**同時**存在於 v2.1.220 的 binary 裡。原文的宣稱本來就是**模型相依**的（它寫的是「for models like Claude Opus 5 and Claude Fable 5」），所以一個 binary 同時帶著兩套 prompt、按模型選用，是合理的解釋——但**我沒有追程式碼路徑去確認選用邏輯，這只是推測**。可以確定的是：**不能說舊規則已經從產品裡消失**，binary 會打臉。

### 本 repo 內部連結

- [harness-engineering.md](./harness-engineering.md) — 使用者視角的五面向
- [ihower-harness-engineering.md](./ihower-harness-engineering.md) — 開發者視角；§八 是本篇的上位框架
- [loop-engineering.md](./loop-engineering.md) — harness 上一層的自走 loop
- [boris-cherny-tips.md](./boris-cherny-tips.md) — Claude Code 實務技巧
- [prompt.md](./prompt.md) — 單次請求的措辭技巧
- [graph-engineering.md](./graph-engineering.md) — workflow 編排層把 context offloading 開到極致（本篇 §2.3 漸進揭露的另一端；offloading 本身在 harness 面向 1）
- [eval-engineering.md](./eval-engineering.md) — 本篇 §前言 說「你自己刪 context 之前該有自己的 eval」，那篇就是那組 eval 該長什麼樣（判官怎麼選、評什麼、案例哪來）

### 校對紀錄

- **2026-08-02**：初版。依 Thariq 2026-07-25 長文整理，重編為「樞紐（成本科目變了）→ 六組差分 → 分層配置 → 判別法」骨架；重疊處採薄委派連回 harness 筆記。原文沒有、本篇自行加入並已就地標示的內容：§四「能力假設 vs 偏好假設」判別法（偏好那半為新增）、§2.1 的 Never/Always 掃描動作、§2.5 的自動記憶位置、§三 堆疊的具體程度排序、§四 `/doctor` 唯讀 vs 可修的區分、§六 本機驗證表。同批處理的交叉修補：harness §七 補第一方實證、ihower §八 補 context 層案例、prompt.md §Few-Shot 加射程註記、boris-cherny-tips §CLAUDE.md 加時效註記。
  - 發布前覆核修正：§五 5.4→**5.5**（誤用了 harness 已作廢的舊編號）；§七 原案例誤標為「社群」（實為 Anthropic 來源，只是沒給比例）；§2.2 原寫「原文沒說清楚射程」，**實際上原文把範圍限定在 tool 使用**，本篇往 prompt.md 與 evaluator 的外推已改標為未驗證；補回原文明講卻漏掉的 CLAUDE.md 兩條（保持輕量＋簡述 repo 用途、verification skill 模式）；移除「原文叫你把偏好寫得更明確」這個原文沒有的說法。`/checkup` 查證後確認是 `/doctor` 的**別名**（非已移除）；auto-memory 的專案數原誤植，已改為不寫死數字。

- **2026-08-02（同日補記）**：§本 repo 內部連結新增 [graph-engineering.md](./graph-engineering.md)，標出分界——該篇談 workflow 編排層把 context offloading 開到極致（subagent 各自扛 context、主 context 只收結論），是本篇 §2.3 漸進揭露的另一端；**offloading 本身仍在 harness 面向 1**，本篇維持不重複的立場（§2.3 末的那則宣告不變）。

- **2026-08-04**：§本 repo 內部連結新增 [eval-engineering.md](./eval-engineering.md)。本篇 §前言 的反方第一條說「你自己刪 context 之前該有自己的 eval」——那篇就是那組 eval 該長什麼樣（判官怎麼選、評什麼、案例哪來、什麼時候可以憑它放手）。本篇內容未動

下次 review 觸發點：下一代模型發布（本篇的差分會再翻一次）、`/doctor` 行為變動、Anthropic 公布更細的 context engineering 指南、§2.2 few-shot 射程有人做出實測、harness 面向 1／5.1 的 CLAUDE.md 描述更新（§五 最後一列的未解衝突）。
