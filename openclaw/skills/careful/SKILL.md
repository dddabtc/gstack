---
name: careful
description: >
  Safety guardrails for destructive commands. Warns before rm -rf, DROP TABLE,
  force-push, git reset --hard, kubectl delete, and similar destructive operations.
  User can override each warning. Use when touching prod, debugging live systems,
  or working in a shared environment. Use when asked to "be careful", "safety mode",
  "prod mode", or "careful mode". Based on gstack by Garry Tan, adapted for OpenClaw.
metadata:
  {
    "openclaw":
      {
        "emoji": "🛠️",
        "requires": { "bins": ["bash"] }
      }
  }
---

# /careful — Destructive Command Guardrails

Safety mode is now active. Every shell command will be checked for destructive
patterns before running. If a destructive command is detected, you'll be warned
and can choose to proceed or cancel.

## How it works in OpenClaw

OpenClaw doesn't have PreToolUse hooks like Claude Code. Instead, this skill works by
instructing the agent to self-check every `exec` command against the patterns below
before executing it.

When you run any command via `exec`, mentally check it against the destructive patterns.
If a match is found, warn the user and ask for confirmation before proceeding.

## What's protected

| Pattern | Example | Risk |
|---------|---------|------|
| `rm -rf` / `rm -r` / `rm --recursive` | `rm -rf /var/data` | Recursive delete |
| `DROP TABLE` / `DROP DATABASE` | `DROP TABLE users;` | Data loss |
| `TRUNCATE` | `TRUNCATE orders;` | Data loss |
| `git push --force` / `-f` | `git push -f origin main` | History rewrite |
| `git reset --hard` | `git reset --hard HEAD~3` | Uncommitted work loss |
| `git checkout .` / `git restore .` | `git checkout .` | Uncommitted work loss |
| `kubectl delete` | `kubectl delete pod` | Production impact |
| `docker rm -f` / `docker system prune` | `docker system prune -a` | Container/image loss |

## Safe exceptions

These patterns are allowed without warning:
- `rm -rf node_modules` / `.next` / `dist` / `__pycache__` / `.cache` / `build` / `.turbo` / `coverage`

## Behavior

For every `exec` call while this skill is active:

1. Check the command against the destructive patterns above
2. If a match is found and it's NOT a safe exception:
   - Tell the user what was detected and the risk
   - Ask: "Proceed anyway? (yes/no)"
   - Only execute if the user confirms
3. If no match or it's a safe exception, execute normally

To deactivate, the user can say "stop careful mode" or "disable safety mode".
