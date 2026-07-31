---
name: ai-workflow-router
version: 0.2.0
description: 依任務複雜度（0-5 級）自動選擇並執行對應的 AI 協作開發流程（預設 AI agent 直處理 / 直接實作 / to-spec / grill-with-docs+to-spec+to-tickets / wayfinder），每個流程與作動任務都要回報進度條、目前位置、下一步和建議技能；沒有可追蹤進度時提供建議工作流。並在實作與 code review 階段安排子代理與跨代理交接，code review 一律呼叫 codex-plugin-cc（可自訂模型與推理強度）。當使用者要開始一項新的程式開發任務——新功能、修改、重構、除錯——且尚未指定要用哪套規劃流程時，主動使用此技能詢問複雜度並依結果執行。任何「我要做/改/加 XXX」的請求都應該先觸發這個技能問一次分級，不要自己用預設流程硬做。
---

# AI Workflow Router

目前版本：v0.2.0

依任務複雜度分六級，把 grilling / spec / ticket 拆分的「規劃投入」和任務大小成正比，避免小任務被過度規劃、大任務被規劃不足。Level 0 不使用技能，由預設 AI agent 直接處理。核心來源：Matt Pocock skills v1.1（grill-with-docs、wayfinder、to-spec、to-tickets、implement、code-review）+ codex-plugin-cc 做 code review。

## 全流程進度回報契約（每次作動都必須遵守）

每個流程、每張票、每個子代理任務都要在「開始、狀態變更、完成或阻塞」時回報進度；統一使用以下 compact template，不要重複建立其他進度模板：

```text
【Level X · <流程名稱>】 `[████░░░░░░]` 40%（2/5）　`執行中`
🔄 <目前步驟／任務>　｜　下一步：<下一個可執行動作>
技能：`<skill-a>`、`<skill-b>`　｜　明細：<僅多票／子代理時填寫>
```

規則：

- 已知步驟數時使用確定進度：`完成數 / 總數`；百分比只在必要任務完成並驗證後才遞增。
- 未知範圍或動態拆票時使用 `進度：[----------] N/A（待盤點）`，不要捏造百分比；盤點完成後立刻建立步驟清單並改用確定進度。
- 每個任務使用狀態圖例：`✅ 完成`、`🔄 執行中`、`⏳ 待處理`、`⛔ 阻塞`、`↪ 略過（附原因）`。未附證據不得標記完成。
- 子代理或平行任務要逐項列出狀態；不能用一個總百分比掩蓋其中的阻塞或失敗。
- 範圍變更時重新計算總數，並明確寫出「範圍由 A 變為 B」；阻塞時保留目前百分比，不得顯示 100%。
- 完成回報必須包含驗證證據（測試、lint、review 或使用者確認），所有必要步驟驗證通過後才顯示 `100%`。
- Level 0 不使用技能；若需要回報進度，`建議技能` 填寫 `無（Level 0 不使用技能）`。

### 無進度資料時

沒有活躍工作流或無法計算進度時，仍用上述模板：第一行填 `【尚未建立】 [----------] N/A（待盤點）`，目前填 `⏳ 尚未有可追蹤任務`，並提供建議 Level 0-5 與下一步；Level 0 的技能欄填 `無`。

除 Level 0 外，建議技能至少從下列規則選一組：需求模糊用 `wayfinder`／`grill-with-docs`；需求明確整理用 `to-spec`；拆分工作用 `to-tickets`；實作用 `implement`／`tdd`；除錯用 `hunt`／`diagnosing-bugs`；驗證與審查用 `verification-loop`／`code-review`。

## Step 0：一定要先問複雜度

任務開始前，先用簡短的一句話描述每一級，讓使用者選 0-5：

```
這個任務的複雜度大概是？
0 預設處理 — 不使用技能，由預設 AI agent 直接處理
1 極小 — 單行/單一小修正，需求已經很明確
2 小型 — 單一函式或模組，你自己心裡已經有答案
3 中型 — 跨 2-5 個檔案的單一功能，需要對齊一下細節
4 大型 — 完整功能、跨前後端或多層，範圍清楚但工作量大
5 巨大/模糊 — 範圍本身還不確定，可能要先做研究才知道要做什麼
```

開始執行前先用上述模板輸出「尚未建立」狀態，將目前填為 `🔄 判斷任務複雜度（0-5）`，下一步填「請使用者選擇等級，或依需求明確度標註建議等級」。

不要自己猜測後直接開工——分級決定後面該不該使用技能或花 token 做 grilling/wayfinder，猜錯會造成無效工作（小任務被過度規劃）或規劃不足（大任務直接跳實作，後面重工）。如果使用者的描述已經清楚到能自己判斷等級（例如「幫我修這個 typo」），可以省略提問直接標註「判斷為 Level 1，如果不對請糾正」再往下走。分級確定後，立刻建立該等級的步驟總數並更新進度條，不要繼續使用 N/A。

## Step 1：依等級執行對應流程

### Level 0 — 預設 AI agent 直處理

不使用任何技能，也不呼叫規劃流程、子代理或 code review；停止本 router 的流程編排，直接由預設 AI agent 依使用者需求處理，並只做必要的基本驗證。

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

## 標準流程節點與回報方式

分級完成後，先把必要節點列成任務清單，再依序更新總進度。下列是預設節點；若任務不需要某節點，標記 `↪ 略過` 並附原因，不要悄悄刪除分母：

| 等級 | 預設流程節點 |
| --- | --- |
| Level 0 | 分級 → 預設 AI agent 直接處理 → 必要驗證 |
| Level 1 | 分級 → 實作 → 驗證 |
| Level 2 | 分級 → `to-spec` → 實作／TDD → 驗證／review |
| Level 3 | 分級 → `grill-with-docs` → `to-spec` → `to-tickets` → 實作 → review／整合 QA |
| Level 4 | 分級 → `grill-with-docs` → `to-spec` → `to-tickets` → 平行實作 → 各票 review → 整合 QA |
| Level 5 | 分級 → `wayfinder` → 決策完成 → `to-spec` → `to-tickets` → 實作 → review／整合 QA |

每個節點開始與結束都使用上述模板；只有多票或子代理時才填寫模板中的 `明細` 欄，不另建進度模板。

任一節點開始前，先回報「目前」；節點完成後，回報驗證證據與新的進度條。若工作流尚未完成但暫時沒有可執行動作，使用 `⛔ 阻塞`，同時說明阻塞原因、需要誰決策，以及解除後的下一步。

## Code Review 設定

預設使用 `gpt-5.6-terra` + high effort；Level 1 可降為 low，Level 2 可降為 medium，其餘維持 high。Level 5 的核心或高風險票加 `--adversarial`。

## 省 token 原則（每一級都要遵守）

1. 不要用力過猛——低等級硬套 grilling/wayfinder 是無效工作。
2. Review 永遠在乾淨 context 做（`/clear` 之後再叫 codex review）。
3. 探索一律丟給 subagent，只把摘要帶回主 context。
4. 對齊過的 spec/PRD 不用每個動作前都整份重讀一次。
5. 規劃文件用完即關閉，不刻意保留造成 doc rot（除非使用者要求留存)。
