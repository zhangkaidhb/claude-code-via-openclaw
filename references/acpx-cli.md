# acpx CLI Reference

## Installation

```bash
# acpx is bundled with OpenClaw
ACPX=~/.nvm/versions/node/v24.13.0/lib/node_modules/openclaw/dist/extensions/acpx/node_modules/.bin/acpx
```

## Commands

### `acpx claude prompt`

Send a prompt to Claude Code.

```bash
# One-shot prompt (--print mode)
$ACPX claude prompt -s <session> "<task>"

# With specific model
$ACPX claude prompt -s <session> --model claude-sonnet-4-20250514 "<task>"

# Resume existing session
$ACPX claude prompt -s <session> "continue from where you left off"
```

**Flags:**
- `-s, --session <name>` — Session name (required for persistence)
- `--model <model>` — Override default model
- `--cwd <dir>` — Working directory (default: current dir)

### Session Management

```bash
# List sessions
$ACPX claude list

# Delete session
$ACPX claude delete -s <session>
```

### Other Agents

```bash
# Codex (OpenAI)
$ACPX codex prompt -s <session> "<task>"

# General ACP
$ACPX prompt --agent <agent-id> -s <session> "<task>"
```

## Integration with OpenClaw exec

```bash
# Background execution in OpenClaw
exec --background --timeout 3600 -- \
  $ACPX claude prompt -s my-session "implement feature X"

# The session ID from exec output is used for monitoring
process action=log sessionId=<exec-session-id> limit=50
```

## Output Format

acpx outputs Claude Code's response as streaming text. Key patterns:
- `[tool]` lines indicate tool calls
- `[plan]` / `[done]` markers indicate task planning/completion
- Exit code 0 = success, 1 = error/failure
