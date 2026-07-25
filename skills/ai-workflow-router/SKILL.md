---
name: ai-workflow-router
description: 依任務複雜度（1-5 級）自動選擇並執行對應的 AI 協作開發流程（直接實作 / to-spec / grill-with-docs+to-spec+to-tickets / wayfinder），並在實作與 code review 階段安排子代理與跨代理交接，code review 一律呼叫 codex-plugin-cc（可自訂模型與推理強度）。當使用者要開始一項新的程式開發任務——新功能、修改、重構、除錯——且尚未指定要用哪套規劃流程時，主動使用此技能詢問複雜度並依結果執行。任何「我要做/改/加 XXX」的請求都應該先觸發這個技能問一次分級，不要自己用預設流程硬做。
---

# AI Workflow Router

依任務複雜度分五級，把 grilling / spec / ticket 拆分的「規劃投入」和任務大小成正比，避免小任務被過度規劃、大任務被規劃不足。核心來源：Matt Pocock skills v1.1（grill-with-docs、wayfinder、to-spec、to-tickets、implement、code-review）+ codex-plugin-cc 做 code review。

## Step 0：一定要先問複雜度

任務開始前，先用簡短的一句話描述每一級，讓使用者選 1-5：

```
這個任務的複雜度大概是？
1 極小 — 單行/單一小修正，需求已經很明確
2 小型 — 單一函式或模組，你自己心裡已經有答案
3 中型 — 跨 2-5 個檔案的單一功能，需要對齊一下細節
4 大型 — 完整功能、跨前後端或多層，範圍清楚但工作量大
5 巨大/模糊 — 範圍本身還不確定，可能要先做研究才知道要做什麼
```

不要自己猜測後直接開工——分級決定後面該不該花 token 做 grilling/wayfinder，猜錯會造成無效工作（小任務被過度規劃）或規劃不足（大任務直接跳實作，後面重工）。如果使用者的描述已經清楚到能自己判斷等級（例如「幫我修這個 typo」），可以省略提問直接標註「判斷為 Level 1，如果不對請糾正」再往下走。

## Step 1：依等級執行對應流程

### Level 1 — 極小
不呼叫任何規劃技能。直接理解需求、修改、跑對應的 test/lint。

**Code review：** `/codex:review --model gpt-5.6-terra --effort low`，只抓明顯錯誤，不用長篇報告。

---

### Level 2 — 小型
呼叫 `/to-spec`（不訪談，只是把已經講清楚的需求統整成一份簡短 spec），跳過拆票，直接進 TDD 實作。

**子代理／交接：** explore 階段可以丟給 subagent，讓它去讀相關程式碼，只把摘要帶回主 context。

**Code review：** `/codex:review --model gpt-5.6-terra --effort medium`。

---

### Level 3 — 中型
1. `/grill-with-docs`（不是舊版 `/grill-me`——它會順便維護 CONTEXT.md 詞彙表和 ADR，長期報酬率更高），5-15 題左右對齊即可
2. `/to-spec`
3. `/to-tickets` 拆成 2-5 張垂直切片票，標出彼此的阻擋關係

**子代理／交接：** 若拆出的票彼此沒有阻擋關係，可以分別開 session／subagent 認領。認領規則：開工前先把票 assign 給自己，沒被 assign 的才算「可認領」，避免兩個 agent 撞同一張票。

**Code review：** 每張票各自 `/clear` 後 `/codex:review --model gpt-5.6-terra --effort high`——review 一定要在乾淨 context 做，同一個 session 邊做邊審，審查者會跟著實作者一起變笨。

---

### Level 4 — 大型
1. `/grill-with-docs`（完整版，範圍大就別省這步）
2. `/to-spec`
3. `/to-tickets`，每張票標註：
   - 阻擋關係
   - AFK（可離人跑）或 HITL（需要人決策）
   - 垂直切片，第一刀要能看到 schema + service + 前端最小畫面

**子代理／交接（這一級是重點）：**
- 沒有阻擋關係的票，平行分派給多個 implement agent／subagent 同時做（類似 Sand Castle 的 planner → 多個 sandbox → merger 架構）
- 認領協議同 Level 3：先 assign 才算 claim
- **交接時給 implement agent 的資訊要完整**，不能只丟 diff 或一句話摘要：
  1. 這張票的完整內容
  2. 專案的 coding standard / CONTEXT.md
  3. 這張票依賴的前置票做了什麼決策（從 spec 的 Decisions 區塊取，不用整份 spec 重讀）
- 若多個 agent 的分支要合併，用一個獨立的 merge agent 處理衝突，而不是叫某個 implement agent 順便兼職

**Code review：** 每張票各自 `/codex:review --model gpt-5.6-terra --effort high`。全部票完成後，人工跑一次整合 QA（這步不能自動化掉），發現的問題直接補成新票，不用整個流程重跑。

---

### Level 5 — 巨大／模糊
1. `/wayfinder`，先把「迷霧」清出來，而不是硬上 to-spec
2. **常見誤用（一定要避免）：** 不要對地圖上單一張任務直接 `/implement`。正確做法是持續對地圖跑 wayfinder，讓它把地圖上的任務逐一解決、決策記錄進「Decisions so far」，直到迷霧清空（wayfinder 自己會判斷路徑已清晰），才整包轉 `/to-spec`
3. 迷霧清空後：`/to-spec` → `/to-tickets`，其餘同 Level 4

**子代理／交接：**
- 地圖上的 research 類型票天生適合丟給 subagent 或背景 agent 離線跑，因為只是去找答案回報，不需要人即時互動
- grilling/prototype 類的決策票必須 HITL，不能讓 agent 自問自答——這類票的「交接」對象是使用者本人，不是另一個 agent
- 進入 to-tickets 之後的並行策略同 Level 4

**Code review：** 同 Level 4，但對核心路徑或高風險的票，加開一次 `/codex:review --model gpt-5.6-terra --effort high --adversarial`，用對抗性審查多找一輪問題。

## Code Review 模型與推理強度對照

**專案預設（`.codex/config.toml`）：**
```toml
model = "gpt-5.6-terra"
model_reasoning_effort = "high"
```
`gpt-5.6-terra` 是 GPT-5.6 家族裡的中階均衡款（性能大約對標 5.5、價格比旗艦 Sol 低約一半），搭配 high effort 作為所有等級的統一預設。

| 等級 | 預設模型/effort | 可選降級（省成本用） |
|---|---|---|
| 1 | gpt-5.6-terra / high | 可降 `--effort low`，明顯錯誤等級不需要高推理 |
| 2 | gpt-5.6-terra / high | 可降 `--effort medium` |
| 3 | gpt-5.6-terra / high | — |
| 4 | gpt-5.6-terra / high | — |
| 5（一般票） | gpt-5.6-terra / high | — |
| 5（核心/高風險票） | gpt-5.6-terra / high + `--adversarial` | 不建議降級 |

低等級任務若想更省，可在該次呼叫加 `--effort low` 或 `--effort medium` 覆蓋專案預設；不特別指定就一律套用 high。若某次任務特別在意品質，也可以臨時指定 `--model gpt-5.6-sol` 換旗艦模型，但成本會明顯上升，開始 review 前先問使用者是否要覆蓋。

## 跨 AI 代理安裝方式

SKILL.md 是開放格式，同一份檔案（含資料夾）可以原封不動用在 Claude Code、Codex CLI、Cursor、GitHub Copilot、Gemini CLI 等支援 Agent Skills 標準的代理上，差別只在放置路徑：

| 代理 | 放置路徑 | 備註 |
|---|---|---|
| Claude Code | `.claude/skills/ai-workflow-router/`（專案）或 `~/.claude/skills/`（個人全域） | 裝完在 CLI 執行 `/reload-plugins` 或重開 session |
| Codex CLI | `.agents/skills/ai-workflow-router/`（專案）或 `~/.codex/skills/`（個人） | 也可放 `.codex/skills/`，效果相同 |
| Cursor | `.agents/skills/ai-workflow-router/` | Cursor 的 skills 是外掛系統的其中一種原件，走 marketplace 或直接放資料夾都可以 |
| GitHub Copilot (VS Code) | `.github/skills/ai-workflow-router/`（會進版控） | VS Code 也認 `.claude/skills/` 和 `.agents/skills/`，可用 `chat.agentSkillsLocations` 設定額外搜尋路徑 |
| Gemini CLI | `.gemini/skills/ai-workflow-router/`，或放 `.agents/skills/` 當備援路徑 | |

**最省事的做法：** 如果團隊裡用的代理不只一種，直接放在 `.agents/skills/ai-workflow-router/`——這是多數代理都會讀取的共用備援路徑，一份檔案全部代理通用，不用複製多份。也可以用 symlink 把同一份實體檔案連到各代理專屬路徑（例如 `ln -s ../../.agents/skills/ai-workflow-router .claude/skills/ai-workflow-router`），確保只有一個真本、改一次全部同步。

若有現成的 CLI 工具（如 `npx skills add`），也可以直接指定目標代理安裝，例如：
```bash
npx skills add <你的repo或路徑> -a claude-code -a codex -a cursor
```

裝完之後大多數代理是「首次啟動自動偵測」，不用額外操作；Claude Code 則建議手動跑一次 `/reload-plugins` 確保吃到最新版本。

## 省 token 原則（每一級都要遵守）

1. 不要用力過猛——低等級硬套 grilling/wayfinder 是無效工作。
2. Review 永遠在乾淨 context 做（`/clear` 之後再叫 codex review）。
3. 探索一律丟給 subagent，只把摘要帶回主 context。
4. 對齊過的 spec/PRD 不用每個動作前都整份重讀一次。
5. 規劃文件用完即關閉，不刻意保留造成 doc rot（除非使用者要求留存)。
6. Code review 統一預設 `gpt-5.6-terra` + high effort（見上方對照表），Terra 本身已是均衡款，不必每次都手動調整；只有明顯瑣碎的 Level 1-2 任務才建議主動降到 low/medium 省成本，其餘等級沿用預設即可，不用逐次思考要不要降級。
