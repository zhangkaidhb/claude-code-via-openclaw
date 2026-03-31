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

### 3. Set Up Auto-Monitoring (Cron)

Create a cron job that checks progress every 5 minutes and notifies the user:

```bash
cron action=add --job '{
  "name": "CC Monitor: <session-name>",
  "schedule": {"kind": "every", "everyMs": 300000},
  "payload": {
    "kind": "agentTurn",
    "message": "Check Claude Code progress. Use process action=list to find acpx/claude processes. If running, use process action=log to get latest output. Report substantive progress only. If completed, report final result. If no processes found, reply NO_REPLY.",
    "timeoutSeconds": 120
  },
  "sessionTarget": "current",
  "delivery": {"mode": "announce", "channel": "<channel>", "to": "<target>"}
}'
```

**Disable when done:**
```bash
cron action=update --jobId <id> --patch '{"enabled": false}'
```

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
- If the first attempt fails, analyze the failure and give Claude Code more precise instructions

### Reporting Results
- Always verify output before reporting to user
- Generate visual proof (screenshots, test output) when applicable
- For Mermaid/UML diagrams: use Playwright to render and screenshot
- Send screenshots as documents to avoid Telegram compression

## Screenshots with Playwright

```bash
# Install once (not in skill - user should have this)
# npm install playwright npx playwright install chromium

# Render HTML to screenshot
node -e "
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch({headless: true, args: ['--no-sandbox']});
  const page = await browser.newPage();
  await page.goto('file:///path/to/file.html', {waitUntil: 'networkidle'});
  await page.screenshot({path: '/tmp/output.png', fullPage: true});
  await browser.close();
})();
"
```

**Tips:**
- Use `fullPage: true` for long content
- Set `viewport` size before goto for specific dimensions
- Use `clip` to crop to a specific element area

## Reference

- [acpx-cli.md](references/acpx-cli.md) — acpx CLI command reference
- [troubleshooting.md](references/troubleshooting.md) — Common issues and solutions
