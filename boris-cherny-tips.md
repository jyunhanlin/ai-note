# Boris Cherny 的 Claude Code 使用技巧

> Boris Cherny 是 Claude Code 的創建者。以下技巧來源於他在 Threads 上分享的個人使用心得與 Claude Code 團隊的集體經驗。

---

## 個人設定

- 使用最強模型（Opus）並開啟**思考模式（Thinking）**：雖然每個 token 速度較慢，但因為需要較少的糾錯與引導，整體反而更快。
- 同時執行多個（約 5 個）Claude Code 實例，用數字標記 Tab，並開啟系統通知以知道何時需要回應。
- 使用 `/statusline` 在狀態列顯示 context 使用量與目前的 git 分支。
- 使用 `/terminal-setup` 設定終端機（如啟用 Shift+Enter 換行）。
- 使用語音輸入撰寫提示（比打字快約 3 倍，可輸入更豐富的內容）。
- 使用 Ghostty 作為終端機以獲得更好的輸出渲染效果。

---

## 平行作業（Parallel Sessions）

- **最大生產力提升**：使用 `git worktree` 建立 3-5 個獨立工作目錄，每個目錄執行獨立的 Claude session，平行處理不同任務，session 之間不會互相干擾。
- 設立一個專用的「分析」worktree，只用於查看日誌或執行 BigQuery 等唯讀任務。
- 可以設定 shell aliases（如 `za`、`zb`、`zc`）以一個按鍵快速切換不同 worktree。
- 一個 session 起草計劃，另一個 session 審查計劃。

---

## CLAUDE.md

- 將 `CLAUDE.md` 視為**活文件**，記錄專案規範、命名慣例、測試指令和常見錯誤。
- 每次 Claude 犯錯後，明確要求它更新 `CLAUDE.md`，讓錯誤不再重複。
  - 可以在每次糾錯後附加：「更新你的 CLAUDE.md，避免再犯同樣的錯誤。」
- 隨時間無情地迭代這個文件，直到錯誤率下降。
- 將 `CLAUDE.md` 提交到 git，讓整個團隊共同維護，持續添加新發現的錯誤。

---

## 計劃模式（Plan Mode）

- 複雜任務先按 **Shift+Tab 兩次**進入 Plan Mode，與 Claude 一起迭代計劃。
- 把精力投入在制定好的計劃上，之後讓 Claude 一次性完成實作（減少中途修正）。
- 計劃完成後再切換到執行模式（例如自動接受編輯）。

---

## 斜線指令（Slash Commands）

- 將常用工作流程建立成可重用的斜線指令，儲存在 `.claude/commands/` 並納入版本控制（與團隊共用）。
- 指令可以包含 inline bash 腳本來預先計算 context，讓每次執行更精準。
- 範例用途：`/techdebt`（移除重複程式碼）、`/commit-push-pr`（整合多個步驟）、`/simplify`（精簡程式碼）。

---

## 子代理（Subagents）

- 明確將子任務分配給子代理，以保持主代理的 context 乾淨、聚焦。
- 使用新的 instance 執行驗證任務，避免主 context 膨脹。
- 常用子代理範例：
  - **code-simplifier**：Claude 完成任務後自動簡化程式碼。
  - **verify-app**：端到端測試，包含詳細的驗證指令（截圖、模擬器建置、測試執行）。

---

## Hooks（自動化鉤子）

- **PostToolUse**：最實用的 hook，在 Claude 每次編輯文件後自動執行格式化工具（如 prettier、swiftformat），避免 CI 失敗。
- **SessionStart**：自動載入最近的 commits 和 sprint context，讓每個 session 都有完整背景。
- 進階用法：透過 hook 將敏感的權限檢查路由到更強的模型（如用於安全掃描）。

---

## 權限管理

- 使用 `/permissions` 預先允許常用的安全指令，減少不必要的中斷，同時保留對高風險操作的確認機制。
- 只對需要修改文件或設定的指令保留審核要求。

---

## 讓 Claude 端到端完成任務

- 連接 Slack MCP、GitHub、Sentry、BigQuery 等工具，讓 Claude 擁有完整的工作流程 context。
- 廣泛委派任務，例如：「去修復失敗的 CI 測試」，而不是微管理每個步驟。
- 給 Claude 一個**驗證迴圈**，讓它自行檢查輸出結果，可帶來 2-3 倍的品質提升。

---

## 提示技巧

- 要求 Claude 解釋其修改的理由，挑戰它的決策。
- 當結果平庸時，要求重新撰寫，而不是接受低品質輸出。
- 要求 Claude 扮演審閱者角色，審查自己或他人的程式碼。

---

## 資料分析

- 在 Claude Code session 中直接使用資料庫 CLI（如 `bq`）進行內聯指標分析。
- 建立專用的分析 worktree，只用於查看資料，不進行程式碼修改。

---

## 學習程式碼

- 啟用解釋輸出模式，讓 Claude 說明它讀到的程式碼。
- 要求以 HTML 視覺化呈現，或使用 ASCII 圖表理解不熟悉的架構。

---

## 排程與長時間任務

- `/loop`：在本地執行週期性任務，最長可達三天。
- `/schedule`：設定在機器關閉後仍能持續執行的雲端工作。

---

## 核心理念

> 「Claude Code 沒有唯一的正確使用方式。我們刻意將它設計成可以任意使用、自訂和擴展的工具。每個 Claude Code 團隊成員的使用方式都截然不同——你應該實驗，找到適合自己的方式。」
>
> — Boris Cherny

---

*資料來源：Boris Cherny 的 Threads 貼文（[@boris_cherny](https://www.threads.com/@boris_cherny)），包含個人設定分享與團隊技巧整理。*
