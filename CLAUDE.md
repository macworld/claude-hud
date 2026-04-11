# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Project Overview

Claude HUD is a Claude Code plugin that displays a real-time multi-line statusline. It shows context health, tool activity, agent status, and todo progress.

## Build Commands

```bash
npm ci               # Install dependencies
npm run build        # Build TypeScript to dist/

# Test with sample stdin data
echo '{"model":{"display_name":"Opus"},"context_window":{"current_usage":{"input_tokens":45000},"context_window_size":200000}}' | node dist/index.js
```

## Architecture

### Data Flow

```
Claude Code → stdin JSON → parse → render lines → stdout → Claude Code displays
           ↘ transcript_path → parse JSONL → tools/agents/todos
```

**Key insight**: The statusline is invoked every ~300ms by Claude Code. Each invocation:
1. Receives JSON via stdin (model, context, tokens - native accurate data)
2. Parses the transcript JSONL file for tools, agents, and todos
3. Renders multi-line output to stdout
4. Claude Code displays all lines

### Data Sources

**Native from stdin JSON** (accurate, no estimation):
- `model.display_name` - Current model
- `context_window.current_usage` - Token counts
- `context_window.context_window_size` - Max context
- `transcript_path` - Path to session transcript

**From transcript JSONL parsing**:
- `tool_use` blocks → tool name, input, start time
- `tool_result` blocks → completion, duration
- Running tools = `tool_use` without matching `tool_result`
- `TodoWrite` calls → todo list
- `Task` calls → agent info

**From config files**:
- MCP count from `~/.claude/settings.json` (mcpServers)
- Hooks count from `~/.claude/settings.json` (hooks)
- Rules count from CLAUDE.md files

**From Claude Code stdin rate limits**:
- `rate_limits.five_hour.used_percentage` - 5-hour subscriber usage percentage
- `rate_limits.five_hour.resets_at` - 5-hour reset timestamp
- `rate_limits.seven_day.used_percentage` - 7-day subscriber usage percentage
- `rate_limits.seven_day.resets_at` - 7-day reset timestamp

### File Structure

```
src/
├── index.ts           # Entry point
├── stdin.ts           # Parse Claude's JSON input
├── transcript.ts      # Parse transcript JSONL
├── config-reader.ts   # Read MCP/rules configs
├── config.ts          # Load/validate user config
├── git.ts             # Git status (branch, dirty, ahead/behind)
├── types.ts           # TypeScript interfaces
└── render/
    ├── index.ts       # Main render coordinator
    ├── session-line.ts   # Compact mode: single line with all info
    ├── tools-line.ts     # Tool activity (opt-in)
    ├── agents-line.ts    # Agent status (opt-in)
    ├── todos-line.ts     # Todo progress (opt-in)
    ├── colors.ts         # ANSI color helpers
    └── lines/
        ├── index.ts      # Barrel export
        ├── project.ts    # Line 1: model bracket + project + git
        ├── identity.ts   # Line 2a: context bar
        ├── usage.ts      # Line 2b: usage bar (combined with identity)
        └── environment.ts # Config counts (opt-in)
```

### Output Format (default expanded layout)

```
[Opus] │ my-project git:(main*)
Context █████░░░░░ 45% │ Usage ██░░░░░░░░ 25% (1h 30m / 5h)
```

Lines 1-2 always shown. Additional lines are opt-in via config:
- Tools line (`showTools`): ◐ Edit: auth.ts | ✓ Read ×3
- Agents line (`showAgents`): ◐ explore [haiku]: Finding auth code
- Todos line (`showTodos`): ▸ Fix authentication bug (2/5)
- Environment line (`showConfigCounts`): 2 CLAUDE.md | 4 rules

### Context Thresholds

| Threshold | Color | Action |
|-----------|-------|--------|
| <70% | Green | Normal |
| 70-85% | Yellow | Warning |
| >85% | Red | Show token breakdown |

## Plugin Configuration

The plugin manifest is in `.claude-plugin/plugin.json` (metadata only - name, description, version, author).

**StatusLine configuration** must be added to the user's `~/.claude/settings.json` via `/claude-hud:setup`.

The setup command adds an auto-updating command that finds the latest installed version at runtime.

Note: `statusLine` is NOT a valid plugin.json field. It must be configured in settings.json after plugin installation. Updates are automatic - no need to re-run setup.

## Dependencies

- **Runtime**: Node.js 18+ or Bun
- **Build**: TypeScript 5, ES2022 target, NodeNext modules

---

# macworld 贡献者上下文

> 以下内容仅用于 macworld fork 的开发协作，不提交到上游。

## 仓库关系

- **origin**: `macworld/claude-hud`（fork）
- **upstream**: `jarrodwatts/claude-hud`（上游）
- 主分支 `main` 保持与 upstream 同步
- 功能分支 `feat/*` 通过 PR 提交到 upstream

```bash
# 同步上游
git fetch upstream && git rebase upstream/main

# 提 PR
git push origin feat/xxx -u
gh pr create --repo jarrodwatts/claude-hud --head macworld:feat/xxx --base main
```

## 本地预览

插件运行缓存在 `~/.claude/plugins/cache/claude-hud/claude-hud/0.0.12/`。同步改动：

```bash
rsync -a --exclude='.git' --exclude='node_modules' ./ ~/.claude/plugins/cache/claude-hud/claude-hud/0.0.12/
# 需新建 Claude Code 会话才能看到 statusLine 变化
```

## 代码规范补充

- `dist/` 不提交 — 上游 CI bot 自动编译
- 新增用户可见文案需加 i18n（`src/i18n/`，使用 `t()` 函数）
- Commit 使用 Conventional Commits（`feat / fix / docs`）

## 当前活跃 PR

### #420 — `feat/configurable-max-width`
- 新增 `maxWidth` config，终端宽度检测失败时的可配 fallback
- 设计：检测成功时 maxWidth 被忽略（是 fallback 不是 override）
- Closes #385, #404
- 改动：`src/config.ts`, `src/render/index.ts` + 测试

### #421 — `feat/usage-compact`
- 新增 `display.usageCompact`，紧凑用量：`5h: 25% (3h 45m)` 替代 `Usage 5h 25% (resets in 3h 45m)`
- 额外修复了 `commands/configure.md` 的 Usage Style Mapping 表格（文档与代码不一致）
- Closes #411
- 改动：`src/config.ts`, `src/render/session-line.ts`, `src/render/lines/usage.ts`, `commands/configure.md`

## Issue #416 评论记录

在 [#416](https://github.com/jarrodwatts/claude-hud/issues/416) 上分享了终端宽度检测研究：

| 方法 | tmux | zellij | 无 TTY |
|------|------|--------|--------|
| `tput cols` | 80（错，默认值） | 正确 | 80（错） |
| `tmux display-message` | 正确 | N/A | N/A |

- 建议检测链：`tmux display-message || tput cols || echo 0`
- 根本方案：Claude Code 应传正确 COLUMNS（[anthropics/claude-code#5430](https://github.com/anthropics/claude-code/issues/5430)）

## 本地 HUD 环境

### statusLine 命令（`~/.claude/settings.json`）

注入了 `COLUMNS=$(tmux display-message -p '#{pane_width}')` 让宽度检测在 tmux 下正确工作。tmux `window-size latest` + `aggressive-resize on` 策略使 pane 宽度跟随最近活跃 client 自动调整。

### HUD 配置（`~/.claude/plugins/claude-hud/config.json`）

```json
{
  "lineLayout": "compact",
  "gitStatus": { "enabled": false },
  "display": {
    "showModel": false,
    "showProject": false,
    "showContextBar": false,
    "showUsage": true,
    "usageBarEnabled": false,
    "usageCompact": true,
    "showSessionName": true,
    "showSpeed": true,
    "showSessionTokens": true,
    "showDuration": true,
    "sevenDayThreshold": 0
  }
}
```

## 已知问题

- `tests/core.test.js` 中 `countConfigs cache: miss on nested rules additions` 是上游 flaky test
- 插件缓存会被插件更新覆盖，需重新 rsync 同步
- 上游 0.0.11 → 0.0.12 无 release tag，版本在 `.claude-plugin/plugin.json` 管理
