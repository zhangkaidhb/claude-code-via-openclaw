# Troubleshooting

## Claude Code Killed by SIGTERM

**Symptom:** Session exits with no output or partial output.

**Cause:** Timeout too short or system resource limits.

**Fix:** Increase `--timeout` to at least 3600s. Check with `ulimit -a`.

## PTY Input Corruption

**Symptom:** Commands sent to PTY session produce garbled output or are ignored.

**Cause:** ANSI escape sequences interfere with input parsing.

**Fix:** Use `--print` mode instead of PTY. Never use `pty: true` for acpx.

## sessions_spawn ACP Doesn't Work in Telegram

**Symptom:** `sessions_spawn` with `runtime: "acp"` fails or returns empty.

**Cause:** Telegram doesn't support ACP thread spawning.

**Fix:** Use `acpx claude prompt` via `exec` instead of `sessions_spawn`.

## Cron Monitoring Can't Access process Tool

**Symptom:** Cron `agentTurn` in isolated session can't use `process action=list`.

**Cause:** Isolated sessions have limited tool access.

**Fix:** Use `"sessionTarget": "current"` to bind the cron to the main session.

## Telegram Image Compression

**Symptom:** Screenshots look blurry when sent to Telegram.

**Fix:** Send as document: `message action=send asDocument=true`. Or use higher DPI.

## Mermaid Diagram Too Small in Screenshot

**Symptom:** Diagram occupies tiny portion of screenshot.

**Fix:** Get SVG bounding box, crop to content area, use deviceScaleFactor 3x+.
