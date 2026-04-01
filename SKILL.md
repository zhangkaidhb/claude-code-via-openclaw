---
name: claude-code-via-openclaw
description: >
  Drive Claude Code (or other ACP coding agents) through OpenClaw for long-running
  coding tasks with background execution, progress monitoring, and result reporting.
  Use when: (1) delegating coding work to Claude Code/Codex/Cursor via acpx CLI,
  (2) running multi-step development tasks that need background execution,
  (3) setting up progress monitoring with automatic notifications,
  (4) managing named Claude Code sessions across multiple turns.
  NOT for: simple one-liner fixes (just edit), reading code (use read tool),
  thread-bound ACP harness requests in chat (use sessions_spawn with runtime:"acp").
---

# Claude Code via OpenClaw

Drive Claude Code through `acpx` CLI for background coding tasks with monitoring.

## Core Workflow

```
1. Spawn  → acpx claude prompt -s <session-name> "<task>" &
2. Monitor → cron job or manual process action=log
3. Report  → parse output, send to user, iterate
```

## Quick Start

### 1. Spawn a Claude Code Session

```bash
ACPX=~/.nvm/versions/node/v24.13.0/lib/node_modules/openclaw/dist/extensions/acpx/node_modules/.bin/acpx
cd <project-dir>

# --print mode: stable, non-interactive, one-shot
exec --background --timeout 3600 -- \
  $ACPX claude prompt -s <session-name> "<detailed task description>"
```

**Key flags:**
- `-s <name>` — Named session (persists across invocations, resume with `-s`)
- `--timeout` — Set generously (3600s for complex tasks, never <300s)
- `--background` — OpenClaw background execution (non-blocking)

### 2. Monitor Progress

```bash
# Check all running processes
process action=list

# Get latest output from a session
process action=log sessionId=<id> limit=50
```

### 3. Progress Monitoring (yieldMs)

#### ⚠️ 不要用 cron 监控

经过多次失败迭代，**cron 监控方案已被废弃**。原因：
- announce 模式消息不可靠（系统吞掉或用户看不到）
- agentTurn 调 message 工具发消息也不稳定
- exec completed 后无法保证第一时间 disable cron（上下文压缩、遗忘）

#### 使用 exec yieldMs

```bash
exec --background --timeout 3600 --yieldMs 300000 -- \
  $ACPX claude prompt -s <session-name> "<task>"
```

`yieldMs: 300000` 每 5 分钟自动唤醒，检查 exec 进程状态：
- 进程仍在运行 → 回到后台等待
- 进程已完成 → 返回结果，继续后续工作（验证、截图、报告）
- 不需要 cron，不需要手动清理

**优点：**
- 无需创建/删除 cron
- 无清理问题（进程结束自动停止检查）
- 与 exec 生命周期绑定，不会泄漏

## Critical Lessons Learned

### Use --print Mode, Not PTY
- PTY mode has ANSI control character bugs that corrupt input
- `--print` gives Claude Code the full task upfront — more stable
- PTY is only needed for interactive debugging, which rarely works through acpx

### Timeout Discipline
- **Never set timeout < 300s** for real coding tasks
- Complex refactors: 3600s (1 hour)
- New feature development: 3600-7200s
- If killed by SIGTERM, the task is lost — err on the side of too long

### Session Management
- Always use `-s <session-name>` for named sessions
- Same session name resumes context across multiple `prompt` calls
- Use descriptive names: `oc-<project>-<feature>` (e.g., `oc-git-uml-seq`)

### Monitoring is Hard
- OpenClaw cannot self-poll; it can only respond to messages/system events
- Use cron jobs for periodic monitoring, but accept they fire in isolated sessions
- Cron `agentTurn` in isolated sessions **cannot** access `process` tool
- Bind to `current` session to get process access: `"sessionTarget": "current"`
- **Always** disable monitoring cron when task completes

### Task Prompt Quality
- Be extremely specific about what to change, which files, which functions
- Include test commands and success criteria
- Show expected output examples when possible
- One task per prompt — don't stack unrelated requests
- **Each prompt must be self-contained** — include file paths, current code state, expected output
- **Include the exact function signatures** when asking Claude Code to modify code
- **State what NOT to do** — e.g., "do NOT remove X", "do NOT change Y"

### Iterative Fix Strategy
- **First attempt failing is normal** — the key is analyzing WHY it failed
- After failure: read the actual code/error, identify the root cause, give Claude Code more precise instructions
- **Do NOT retry with the same vague prompt** — each iteration must be more specific
- Common failure patterns:
  - "Too many details" → add filtering/criteria (e.g., "only cross-class calls")
  - "Wrong syntax" → test the syntax yourself first, then specify what works
  - "Test failures" → read the test error, tell Claude Code exactly which assertion failed
- **Analyze before delegating** — if you can identify the bug yourself, fix it directly instead of spawning Claude Code

### "Just Do It" Principle
- When user's intent is clear and direction is obvious, execute immediately
- Do NOT ask "要我现在做吗？" or "你想继续吗？" for clear tasks
- Ask only when there's genuine ambiguity or multiple viable approaches

### Reporting Results
- Always verify output before reporting to user
- Generate visual proof (screenshots, test output) when applicable

## Reference

- [acpx-cli.md](references/acpx-cli.md) — acpx CLI command reference
- [troubleshooting.md](references/troubleshooting.md) — Common issues and solutions
- [rendering.md](references/rendering.md) — Screenshots, Mermaid pitfalls, visual output

## Reference

- [acpx-cli.md](references/acpx-cli.md) — acpx CLI command reference
- [troubleshooting.md](references/troubleshooting.md) — Common issues and solutions
