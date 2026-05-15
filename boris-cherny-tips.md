# Boris Cherny 的 Claude Code 使用技巧

> Boris Cherny 是 Claude Code 的創建者。以下技巧來源於他在 Threads 上分享的個人使用心得與 Claude Code 團隊的集體經驗。

> 「Claude Code 沒有唯一的正確使用方式。我們刻意將它設計成可以任意使用、自訂和擴展的工具。每個 Claude Code 團隊成員的使用方式都截然不同——你應該實驗，找到適合自己的方式。」
>
> — Boris Cherny

---

## 目錄

1. [環境與個人設定](#環境與個人設定)
2. [核心工作流程](#核心工作流程)
3. [客製化與自動化](#客製化與自動化)
4. [進階與隱藏功能](#進階與隱藏功能)
5. [應用情境](#應用情境)

---

## 環境與個人設定

### 模型與終端機
- 使用最強模型（Opus）並開啟**思考模式（Thinking）**：雖然每個 token 速度較慢，但因為需要較少的糾錯與引導，整體反而更快。
- 使用 Ghostty 作為終端機以獲得更好的輸出渲染效果。
- 使用 `/terminal-setup` 設定終端機（如啟用 Shift+Enter 換行）。
- 使用 `/statusline` 在狀態列顯示 context 使用量與目前的 git 分支。
- 使用語音輸入撰寫提示（比打字快約 3 倍，可輸入更豐富的內容）。

### 多 Session 管理
- 同時執行多個（約 5 個）Claude Code 實例，用數字標記 Tab，並開啟系統通知以知道何時需要回應。
- 使用 `git worktree` 建立 3-5 個獨立工作目錄，每個目錄執行獨立的 Claude session，平行處理不同任務，session 之間不會互相干擾（**最大生產力提升**）。
- 設定 shell aliases（如 `za`、`zb`、`zc`）以一個按鍵快速切換不同 worktree。
- 設立一個專用的「分析」worktree，只用於查看日誌或執行 BigQuery 等唯讀任務，不進行程式碼修改。
- 一個 session 起草計劃，另一個 session 審查計劃。

### 權限管理
- 使用 `/permissions` 預先允許常用的安全指令，減少不必要的中斷，同時保留對高風險操作的確認機制。
- 只對需要修改文件或設定的指令保留審核要求。

---

## 核心工作流程

### 計劃模式（Plan Mode）
- 複雜任務先按 **Shift+Tab 兩次**進入 Plan Mode，與 Claude 一起迭代計劃。
- 把精力投入在制定好的計劃上，之後讓 Claude 一次性完成實作（減少中途修正）。
- 計劃完成後再切換到執行模式（例如自動接受編輯）。

### 端到端任務委派
- 連接 Slack MCP、GitHub、Sentry、BigQuery 等工具，讓 Claude 擁有完整的工作流程 context。
- 廣泛委派任務，例如：「去修復失敗的 CI 測試」，而不是微管理每個步驟。
- 給 Claude 一個**驗證迴圈**，讓它自行檢查輸出結果，可帶來 2-3 倍的品質提升。

### 提示技巧
- 要求 Claude 解釋其修改的理由，挑戰它的決策。
- 當結果平庸時，要求重新撰寫，而不是接受低品質輸出。
- 要求 Claude 扮演審閱者角色，審查自己或他人的程式碼。

---

## 客製化與自動化

### CLAUDE.md
- 將 `CLAUDE.md` 視為**活文件**，記錄專案規範、命名慣例、測試指令和常見錯誤。
- 每次 Claude 犯錯後，明確要求它更新 `CLAUDE.md`，讓錯誤不再重複。
  - 可在每次糾錯後附加：「更新你的 CLAUDE.md，避免再犯同樣的錯誤。」
- 隨時間無情地迭代這個文件，直到錯誤率下降。
- 將 `CLAUDE.md` 提交到 git，讓整個團隊共同維護，持續添加新發現的錯誤。

### 斜線指令（Slash Commands）
- 將常用工作流程建立成可重用的斜線指令，儲存在 `.claude/commands/` 並納入版本控制（與團隊共用）。
- 指令可以包含 inline bash 腳本來預先計算 context，讓每次執行更精準。
- 範例用途：`/techdebt`（移除重複程式碼）、`/commit-push-pr`（整合多個步驟）、`/simplify`（精簡程式碼）。

### 子代理與自訂 Agent
- 明確將子任務分配給子代理，以保持主代理的 context 乾淨、聚焦。
- 使用新的 instance 執行驗證任務，避免主 context 膨脹。
- 在 `.claude/agents/` 定義具有受限工具集和自訂 system prompt 的專用 agent（透過 `--agent` 旗標使用），例如唯讀審查 agent 或領域特定 agent。
- 常用子代理範例：
  - **code-simplifier**：Claude 完成任務後自動簡化程式碼。
  - **verify-app**：端到端測試，包含詳細的驗證指令（截圖、模擬器建置、測試執行）。

### Hooks（自動化鉤子）
- **PostToolUse**：最實用的 hook，在 Claude 每次編輯文件後自動執行格式化工具（如 prettier、swiftformat），避免 CI 失敗。
- **SessionStart**：自動載入最近的 commits 和 sprint context，讓每個 session 都有完整背景。
- **PreToolUse**：記錄所有 bash 指令，方便稽核與除錯。
- **PermissionRequest**：將權限確認請求路由到 WhatsApp 或 Slack，讓你遠端批准高風險操作。
- **PostCompact**：在 Claude 壓縮 context window 之後觸發，可用於重新注入關鍵指令，避免重要資訊在壓縮後遺失。
- 進階用法：透過 hook 將敏感的權限檢查路由到更強的模型（如用於安全掃描）。

---

## 進階與隱藏功能

### 努力程度控制（Effort Levels）
- 預設努力程度已調整為 **xhigh**（介於 high 和 max 之間的新等級），在推理深度與延遲之間取得平衡。
- `/effort max`：讓 Claude 進行更長時間的推理，用盡所需 token，適合複雜的除錯、架構決策或需要深度思考的棘手問題。
- 可透過 effort 設定、task budgets 或要求 Claude 簡潔回答來管理 token 使用量。

### 排程與長時間任務
- `/loop`：在本地執行週期性任務，最長可達三天。Boris 實際運行的排程範例：每 5 分鐘處理一次 code review、每 30 分鐘推送 Slack 反饋、每小時清理過期 PR。
- `/schedule`：設定在機器關閉後仍能持續執行的雲端工作。
- **`/goal`**（2026 年 5 月新增）：為任務設定目標條件，讓 Claude 自動執行直到達成。底層機制等同於 `/loop until <條件>`——有人在 Threads 問 Boris 能否新增 `/goal` 功能，他回覆「用 `/loop until` 就行了」，隨後 Claude Code 2.1.139 於 18 小時內正式推出 `/goal` 指令。（[來源](https://www.threads.com/@aabyzov/post/DYOc5I1CDvH/may-boris-cherny-at-anthropic-gets-asked-about-goal-for-claude-code-replies)）

### 跨裝置與遠端控制（2026 年 3 月分享）
- **行動裝置 App**：Claude Code 有 iOS / Android 應用程式，可直接在手機上審查 PR、撰寫程式碼，不需要開電腦。
- **`--teleport`**：將雲端 session 傳送至本地終端機繼續執行。
- **`/remote-control`**（或 `claude remote-control`）：從手機或網頁控制本地執行中的 session。Boris 在設定中對所有 session 預設啟用 remote-control。
- **Cowork Dispatch**：Claude Desktop App 的安全遠端控制，可透過 MCP 和瀏覽器處理郵件、管理檔案等非編碼任務，Boris 每天用它遠端處理 Slack 和電子郵件。
- **Cowork + Opus 4.7 自動完成現實世界任務**（2026 年 5 月分享）：將個人偏好（如航班艙等、飯店需求）寫進 Cowork instructions，讓 Opus 4.7 自動開啟瀏覽器、瀏覽多個網站並完成預訂。Boris 分享：過去 Cowork 訂機票表現平平，但搭配 Opus 4.7 後首次成功一次性完成——在他繼續用 Claude Code 工作的同時，Cowork 幫他訂了 8 趟班機和 5 間飯店。（[來源](https://www.threads.com/@boris_cherny/post/DYOBHmqmiR5/i-needed-to-book-flights-for-a-bunch-of-upcoming-travel-as-always-i-used-claude)）

### 前端與瀏覽器整合
- **Chrome 擴充功能**：讓 Claude 直接在瀏覽器中驗證前端輸出，可迭代修正網頁外觀與功能。
- **Desktop App 內建 Web Server**：Desktop 版本會自動啟動 web server，並用內建瀏覽器進行測試，簡化前端開發流程。

### Session 與大規模操作
- **`/branch`**：建立對話分支，或用 `claude --resume <session-id> --fork-session` 從 CLI 複製已有 session，方便嘗試不同方向。
- **`/btw`**：在 Claude 執行任務的過程中插入一個快速的旁側問題，不中斷主任務、不污染對話歷史，答案以浮層顯示。
- **`/batch`**：將大規模程式碼遷移分散到數百甚至數千個 worktree agent 上並行執行，適合跨整個 codebase 的大量變更。

### 其他旗標
- **`--bare`**：跳過自動載入本地設定，讓 SDK 啟動速度提升最多 10 倍；當你明確指定 system prompt 和 MCP 時使用。
- **`--add-dir`**：授予 Claude 存取主工作目錄以外額外 repository 的權限，適合跨多個 repo 的任務。

---

## 應用情境

### 資料分析
- 在 Claude Code session 中直接使用資料庫 CLI（如 `bq`）進行內聯指標分析。
- 搭配專用的分析 worktree（見「多 Session 管理」），只用於查看資料。

### 學習程式碼
- 啟用解釋輸出模式，讓 Claude 說明它讀到的程式碼。
- 要求以 HTML 視覺化呈現，或使用 ASCII 圖表理解不熟悉的架構。

---

*資料來源：Boris Cherny 的 Threads 貼文（[@boris_cherny](https://www.threads.com/@boris_cherny)），包含個人設定分享與團隊技巧整理。*
