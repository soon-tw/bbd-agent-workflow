# bbd-agent-workflow

個人 / 團隊用的 AI Agent 效率提升技能集合（Agent Skills），採開放的 `SKILL.md` 格式，
可安裝到 Claude Code、Codex CLI、Cursor、GitHub Copilot、Gemini CLI 等支援 Agent Skills 標準的代理。

## 目錄結構

```
bbd-agent-workflow/
└── skills/
    └── ai-workflow-router/
        └── SKILL.md
```

每個技能一個資料夾，資料夾名稱就是技能名稱，內含一份 `SKILL.md`（必要）與可選的
`references/`、`scripts/`、`assets/` 等輔助資源。之後新增技能，比照這個結構在
`skills/` 底下新增資料夾即可。

## 目前收錄的技能

| 技能 | 說明 |
|---|---|
| [`ai-workflow-router`](skills/ai-workflow-router/SKILL.md) | 依任務複雜度（1-5 級）自動選擇對應的 AI 協作開發流程（直接實作 / to-spec / grill-with-docs+to-spec+to-tickets / wayfinder），並安排子代理分工與跨代理交接；code review 統一呼叫 codex-plugin-cc（預設 `gpt-5.6-terra` + high effort）。 |

## 安裝方式

依你使用的代理，把 `skills/` 底下對應的技能資料夾複製或建 symlink 到代理的技能路徑：

| 代理 | 路徑 |
|---|---|
| Claude Code | `.claude/skills/<技能名>/`（專案）或 `~/.claude/skills/`（個人全域） |
| Codex CLI | `.agents/skills/<技能名>/` 或 `~/.codex/skills/` |
| Cursor | `.agents/skills/<技能名>/` |
| GitHub Copilot (VS Code) | `.github/skills/<技能名>/` |
| Gemini CLI | `.gemini/skills/<技能名>/`，或 `.agents/skills/` 當備援路徑 |

多代理團隊建議統一放在 `.agents/skills/`，這是多數代理都會讀取的共用備援路徑，
一份檔案全部代理通用。

## 授權

MIT（依你的實際需求調整）
