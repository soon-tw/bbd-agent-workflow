# bbd-agent-workflow

個人 / 團隊用的 AI Agent 效率提升技能集合（Agent Skills），採開放的 `SKILL.md` 格式，
可安裝到 Claude Code、Codex CLI、Cursor、GitHub Copilot、Gemini CLI 等支援 Agent Skills 標準的代理。

## 快速使用

```bash
npx skills add soon-tw/bbd-agent-workflow
```

## 目錄結構

```text
bbd-agent-workflow/
└── skills/
    └── ai-workflow-router/
        ├── SKILL.md
        └── references/
            ├── context-management.md
            └── prompt-templates.md
```

每個技能一個資料夾，資料夾名稱就是技能名稱，內含一份 `SKILL.md`（必要）與可選的
`references/`、`scripts/`、`assets/` 等輔助資源。之後新增技能，比照這個結構在
`skills/` 底下新增資料夾即可。

## 目前收錄的技能

| 技能 | 說明 |
| --- | --- |
| [`ai-workflow-router`](skills/ai-workflow-router/SKILL.md) | v0.5.0；依任務複雜度、上下文壓力與目前 AI agent 機制，建議最小工作流、context mode、技能搭配及提示詞。區分人為觸發的流程技能與 agent 可觸發的工作紀律，並透過按需載入的 references 支援 context 管理；不會自動啟動流程。 |

## 使用方式

在需要規劃、選擇技能、評估協作方式或管理長任務 context 時使用 `ai-workflow-router`。它會同時評估任務複雜度、上下文壓力，以及當前 agent 是否支援 skills、子代理、平行／背景任務、artifacts、獨立 review 與驗證工具，再提供最小流程與可貼用提示詞。主技能保持精簡，只有需要時才載入 `references/` 中的詳細策略。

## 跨 AI 代理安裝方式

依你使用的代理，把 `skills/` 底下對應的技能資料夾複製或建 symlink 到代理的技能路徑：

| 代理 | 路徑 |
| --- | --- |
| Claude Code | `.claude/skills/<技能名>/`（專案）或 `~/.claude/skills/<技能名>/`（個人全域） |
| Codex CLI | `.agents/skills/<技能名>/` 或 `~/.codex/skills/<技能名>/` |
| Cursor | `.agents/skills/<技能名>/` |
| GitHub Copilot (VS Code) | `.github/skills/<技能名>/` |
| Gemini CLI | `.gemini/skills/<技能名>/`，或 `.agents/skills/<技能名>/` 當備援路徑 |

快速安裝：

```bash
npx skills add soon-tw/bbd-agent-workflow -a claude-code -a codex -a cursor
```

多代理團隊建議統一使用 `.agents/skills/`；更新後重啟代理即可載入最新技能。

## 授權

MIT（依你的實際需求調整）
