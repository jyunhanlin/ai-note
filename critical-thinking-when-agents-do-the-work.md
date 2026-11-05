# Critical thinking when agents do the work

> Addy Osmani，2026-11-03，[原文（X）](https://x.com/addyosmani/status/2117512471930167377)

### Why critical thinking got harder

```
  以前   問題  →  你在這段空間裡思考、判斷  →  答案
  現在   問題  →（幾秒）→  答案
```

作者立場
- 他很喜歡 agentic workflow，不是要你少用
- 目標：在什麼都能交給 agent 的世界裡，保持 critical thinking

以前 vs 現在
- Critical thinking 的老定義：帶著好奇和懷疑看新資訊，聽不同觀點，形成自己的意見；現在仍然成立
- Past -> cognitive and aesthetic processing 發生在問題與答案之間
- Now -> 這一層已經被 AI 處理掉，剩下幾秒鐘

為什麼變難
- Model 越好，明顯的錯越少
  - 大部分輸出寫得好、大致正確，不會讓你想再看一眼
  - 難的不是錯很多，而是錯很少、很難察覺
- 就算同時跑很多 agent，看 output 的眼睛還是只有一雙
- Agentic workflow 很舒服：快、順、常常是對的 → 警覺心自然放鬆

所以
- 日常裡沒有任何東西會逼你做 critical thinking，你必須自己把它放進日常

### Offloading, deferral and surrender

Spectrum

```
  top     Offloading   委派給 AI，結果還是你負責      ← 該待的地方
   │
   │      Deferral     AI 的 output 變成你的新 input
   │
   ▼      Surrender    AI 的 output 悄悄變成你自己的想法
  bottom

  怎麼滑下去：信任（或懶）增加，每一步都很小、看起來都合理
  仔細讀 → 用掃的 → 只看骨架 → 被問到某一行，答不出來
```

Cognitive offloading
- 委派任務給 AI
- Ownership 還是在於自己
  - 作者整天都這樣用，這是該待的地方

Deferral
- AI 的 output 就是自己的 new input
  - 危險在於它不再被當成假設，你直接在上面繼續蓋

Cognitive Surrender
- AI 的 output 悄悄變成你自己的想法, 不覺得需要再檢查
  - 這個詞來自 Wharton 的 Steven Shaw 和 Gideon Nave

為什麼要知道自己在光譜哪裡
- 往下滑的過程看不見：每一步都很小、看起來都合理
- 測試：連 AI 幫忙都解釋不出來，就已經交出 ownership

Debt
Cognitive debt
- surrender 累積出來的：腦中該有的系統模型不存在
  - 平常看不見，要改系統時才發現沒人懂

Comprehension debt
- cognitive debt 在 code 上的版本（Jeremy Twei 提出）
  - codebase 看起來很乾淨、測試也過，但沒人懂

不用守住全部
- 大型 codebase 本來就不可能全記在腦中；這種 debt 比想像中容易補回來
  - Claude Code 團隊的做法：需要時請 Claude 解釋那一塊，現場重建理解
- 前提是知道自己缺什麼（"You just need to know what you don't have"）
- 錯了代價低的事可以整個交出去；後果由別人承擔的，推理自己來

### The questions I ask myself

Did I formulate a hypothesis before asking
- 因為讀過寫得好的回答後, 可能就不會有其他想法
- 完全沒有頭緒的時候, 可以先做 blind spot pass
  - blind spot pass是一種提問方式：把你知道的告訴 AI，請它找出你沒想到要問的事。 它不回答你的問題，而是幫你找出盲點。

Am I asking it to help me think or to offload thinking from me
- 往 surrender 滑的過程自己察覺不到；以為自己在思考，其實是在 offload，就會累積 comprehension debt

Have I heard the counter argument from a different angle
- 避免被自己的答案帶走
  - 立場很硬時，問法本身就帶著立場，AI 會順著回答，你只聽到自己的回音


What would have to be true for this to be wrong
- AI 說它對、說它有把握，都不是證據；問「對嗎」也很容易得到「對」
  - 改問「在什麼情況下它會錯」，就能找出一個可以實際跑的檢查，再去跑

Would I bet on this in front of the person who has to maintain it
- 理解 AI 的產出, 並對該產出負責
  - 測試通過只代表能跑；後果由維護的人承擔，你得答得出「為什麼這樣寫」

What am I no longer practicing
- AI 替你做久了，有些技能會在你沒注意時退化；這題讓「放掉哪些」變成你的選擇，而不是意外


### Verification as a habit

```
  ┌─→ agent 改 code
  │     ↓
  │   跑驗證：e2e test / property test / 實際操作 app
  │     ↓
  │   讀結果
  │     ↓
  └── 沒過就再試            ← 這幾輪你都還沒看任何東西
        ↓
      過了 → 你看證據，不是讀每一行
```

瓶頸移動了
- 瓶頸已經從寫 code 移到「確定自己是對的」
- 給 agent 驗證的手段，它就能自己改 → 跑 → 讀結果 → 再試
  - Boris Cherny："give Claude a way to verify its work"
  - 你的工作從「讀每一行」變成「建 loop、看證據」

驗證也會 surrender
- 「有證據支持我的結論嗎？」不知不覺變成「check 綠了嗎？」
  - agent 寫 test 很便宜，你會看到大量綠燈
  - test 跟 feature 犯同一個錯，test 照樣會過
  - 一個綠燈只證明一件事，很容易以為它涵蓋全部
- 綠了之後問：它證明了什麼？沒證明什麼？

事前定義 "verified"
- 事後才定義，任何結果你都能合理化
- 終點要具體：test 通過、截圖、log 或 trace
  - 讓「完成」是事實，不是「看起來好了」
- agent 回報完成時，先看它跑的指令和輸出，再看 summary
  - summary 是 agent 的說法，指令和輸出才是證據

把檢查放進 loop
- 讓 subagent 在新的 context 審查自己的成果，每次都跑，不靠你記得
  - 新 context 沒帶著原本的假設，比較看得出問題（我的理解）
- 最後問一句 "Did I miss anything?"，常常會多冒出兩三件事

什麼時候要深入
- 判斷什麼時候該深入，本身就是一種技能
- "If you can't say what the checks would miss, ask for a real review rather than a quick approval."
  - 說不出 check 漏了什麼，代表你不知道它的範圍，就不能只快速 approve

不只是 code
- agent 整理了 50 通客戶訪談或競品分析 → 抽查幾份原始資料
- 建議所依賴的關鍵數字 → 自己確認算法

結論
- agent 讓驗證變得很便宜；難的是知道好的驗證長什麼樣子


### Actually paying attention to what's generated

Review 變成蓋章
- 例子：600 行 PR，看一下結構、變數名合理、test 過了 → approve
  - 但中間可能藏著 transaction 邊界的順序被改了，或某個 edge case 的 default 被翻轉
  - 你有 review，但只找到你本來就知道會找的東西
- 寫 code 和 review 是不同的技能；review 到某個點就變成 rubber stamping

怎麼讀
- 讀 diff 時，當作不是 AI 寫的
- 再看 diff 之外：agent 替你做了哪些決定
  - 長 session 裡它會做很多小決定：default 值、什麼不做、兩個需求衝突時改寫了哪一個
  - summary 只會說「一切順利」，這些決定都看不到 → 請它列出做過的決定和假設

考自己
- 隨機挑幾行，問自己這行在做什麼
- 文件：關掉後，憑記憶寫出主要論點
- 長 session：請 Claude 寫一份變更報告，底部附測驗，規則是「全對才 merge」
- 要理解到能辯護，因為總有一天會有人問

讓 agent 守住你的注意力
- 作者的開源專案 PR 很多：用 agent 分出哪些違反 guideline、哪些需要他親自看
- 他很少讓 agent 自己下判斷；agent 只負責縮小範圍，他再想是不是解對問題、用對方法

Orchestration Tax

```
  agent 產出的量  ====================
  你敢署名的量    ========
                          └──────────┘
                       orchestration tax
```

- agent 產出的量，和你敢署名的量之間的落差
- 完成的工作堆得比你看得快 → 再開一個 session 也沒用
  - 先處理落差：把 check 收緊，讓需要你看的變少；不然就放慢
- "When you're producing more than you can stand behind, you're no longer reviewing. You're hoping."

### Quality in, quality out

```
  以前   問題  →  你（在中間思考）  →  答案
  現在   你（寫輸入）  →  agent  →  你（審輸出）
```

心智模型的位置變了
- 你的心智模型還在，只是從中間移到前後兩端：前面寫任務描述和 context，後面審 output
- 所以好的 input 決定 output 的品質；作者把寫好 prompt 當成一種 critical thinking 的技能

Intent debt
- 以前需求講不清楚，會有人追問，你才發現「我其實沒想清楚」
- agent 不會停下來問：它自己補上你漏掉的部分，交回看起來完成的東西，但背後的想法還是模糊的
- Intent debt：目標、限制、理由沒有寫出來，藏在你沒親手寫的 artifact 裡
  - 別種 debt 別人能幫你補，這種只能你自己補，因為 intent 必須來自你
- Prompt 反映你對問題的理解；寫不清楚，可能是你自己還沒想清楚

怎麼寫輸入
- 先用自己的話寫 brief：讀者是誰、什麼不做、什麼叫完成
- 修正兩輪還不對 → 停止微調，重寫 prompt
- 讓 agent 幫忙：不清楚時，請它訪談你，問它不懂的地方
  - Andy Clark 區分 delegating 和 collaborating：協作時，prompt 讓 output 更好，output 讓下一個 prompt 更好，也讓你更懂這個問題
  - 成功的訊號：關掉 session 時，你自己的心智模型更清楚了

小心 input 鏈
- 每個 agent 只接上一個 agent 的 summary → 到第三個 agent，已經沒有人讀過原始文件

自己的輸出標準
- AI 的輸出都格式漂亮、語氣篤定，「看起來精緻」已經不是有用的訊號
- 不願意掛自己名字的，就不輸出
- 不合標準時，把原因寫進 shared config、hook 或 test，避免再發生
  - 這樣品味才會累積；寫下為什麼沒過，本身就是好的練習

### "But the models will be better at this than me"

```
  work            → 可以交給 agent
  auditing        → 大部分可以交給 agent
  accountability  → 交不出去，一定是你
```

反對意見
- 一兩年後 model 就能看出我看得出的問題，為什麼還要維持這個技能？

作者的回應
- 偵測問題確實會越來越交給 agent，他自己也這樣主張
- 但總要有人為上線的東西負責；不管 AI 幫了多少，負責的一定是人

為什麼還要自己 review
- 要信任 agent 多少，本身就是一個判斷
- 保持校準的方法就是 review，因為 model 會犯的錯一直在變
- Paradox of supervision（Anthropic 對自家工程師與研究員的研究）
  - "supervising Claude requires the very coding skills that may atrophy from AI overuse"
  - 監督 AI 需要的技能，正是過度用 AI 會退化的技能

Better model 解決不了的問題
- 你在做什麼、為什麼值得做、影響誰、願意犧牲什麼
- agent 可以幫你想，但答案是你的

結論
- "You may offload the work and much of the auditing, but not the accountability."

### Practices for the everyday

```
  每週   不用 agent 寫一點 code，或做一件「忘了會很慘」的事
  每月   寫下自己不再做的事，決定哪些可以放掉
  隨時   遇到看不懂的東西就去學，不只查一行解釋
```

原則
- 不需要全部嚴格照做，挑適合自己 workflow 的就好

每週：校準練習
- 不用 agent 寫一點 code，或做一件忘了怎麼做會很慘的事
- 作者叫它 calibration exercise，但不是道德要求
  - 用來確認自己的能力還在（我的理解）

每月：盤點
- 寫下自己不再做的事，決定哪些可以接受不做
  - 這就是 "What am I no longer practicing" 的定期版
- 問自己：如果 AI 不存在，你會有多迷失？

隨時：看不懂就去學
- 遇到沒看過的詞，可以查一行解釋直接貼上；作者寧願花點時間研究，看懂它放在整體的哪裡

把驗證排進時程
- 好好檢查 AI 的產出是真的工作；假裝它不花時間，團隊最後就不會做
- 它不是免費的，要排成正式的工作項目

技能會消失
- 不用的技能會消失，而且比你想的快
- 不願意放掉的，就持續練

### What to bring with you

```
  問之前      Be intentional / Have a hypothesis / Craft your prompt
  拿到結果後  A green check means one thing / Look for evidence / Read the decisions
  交付前      Set a bar
  長期        Keep yourself in the game
  隨時        Be honest about your place on the spectrum
```

問之前
- Be intentional：agent 可以考你、訪談你、給你看證據，但前提是你先設定好
  - → Actually paying attention（考自己）、Quality in（請 agent 訪談你）、Verification（證據）
- Have a hypothesis：問之前先有假設，猜錯也沒關係
  - → Did I formulate a hypothesis before asking
- Craft your prompt：你寫 prompt 的方式，反映你對問題懂多少
  - → Quality in, quality out（intent debt）

拿到結果後
- A green check means one thing：它檢查了什麼？沒檢查什麼？
  - → Verification（驗證也會 surrender）
- Look for evidence：先看它跑了什麼、回了什麼，再看 summary
  - → Verification（事前定義 verified）
- Read to know what decisions it made：它替你決定了什麼？
  - → Actually paying attention（看 diff 之外的決定）

交付前
- Set a bar, and stick to it：不願意掛自己名字的，就不交出去
  - → Would I bet on this in front of the person who has to maintain it、Quality in（自己的輸出標準）

長期
- Keep yourself in the game：重要的技能要親手練
  - → Practices for the everyday、What am I no longer practicing

隨時
- Be honest about your place on the spectrum：連 AI 幫忙都解釋不出來，就已經 surrender 了
  - → Offloading, deferral and surrender

一個可以馬上做的練習
- 問之前，寫下你預期會發生什麼
- agent 做完後，寫下一件它沒檢查的事
- 作者想知道：這樣做之後，你讀 output 的方式有沒有改變？

全篇收尾
- Think independently. Ask better questions. Take responsibility for the outcome.

