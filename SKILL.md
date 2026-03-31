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

#### Lifecycle

```
启动任务 → 创建/启用监控 cron → 任务完成 → 禁用监控 cron → 报告结果
```

**必须严格遵循此生命周期。** 禁止在无活跃任务时保持 cron 运行——会发送无意义的"无活跃任务"消息。

#### 创建监控 Cron

```bash
cron action=add --job '{
  "name": "CC Monitor: <session-name>",
  "schedule": {"kind": "every", "everyMs": 300000},
  "payload": {
    "kind": "agentTurn",
    "message": "检查 Claude Code 进度：\n1. 用 process action=list 找 acpx/claude 进程\n2. 有活跃进程 → 用 process action=log 看最新输出，汇报进展\n3. 无活跃进程 → 报告\"无活跃任务\"\n\n每次必须发一条消息，禁止不发消息。保持简短（3行以内）。",
    "timeoutSeconds": 60
  },
  "sessionTarget": "current",
  "delivery": {"mode": "announce", "channel": "<channel>", "to": "<target>"}
}'
```

#### 禁用监控 Cron（任务完成后）

```bash
cron action=update --jobId <id> --patch '{"enabled": false}'
```

#### Cron Prompt 设计原则

经过多次失败迭代总结出的规则：

| 规则 | 原因 |
|------|------|
| **禁止写 NO_REPLY** | 系统会吞掉 NO_REPLY，用户看到的是 `⚠️ ⏰ Cron failed` |
| **必须写"禁止不发消息"** | 否则 agent 会自行判断"没有变化"而跳过通知 |
| **prompt 要极简（~50 词）** | current session 会累积历史，长 prompt 浪费 token |
| **timeoutSeconds: 60** | 监控只需查进程+发消息，不需要太长 |
| **sessionTarget: "current"** | isolated session 无法访问 process 工具 |
| **每次任务用新 cron** | 复用旧 cron 会继承上次 session 的历史上下文 |

#### 典型工作流

```
1. exec --background 启动 Claude Code → 获得 sessionId
2. cron action=add 创建监控（everyMs: 300000, sessionTarget: current）
3. 等待 exec completed 系统消息 或 cron 通知
4. 验证结果，报告给用户
5. cron action=update enabled=false 关闭监控
6. 如果下次任务复用同一 session，创建新 cron（不要复用旧的）
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
