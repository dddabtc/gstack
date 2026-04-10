---
name: guard
description: >
  Full safety mode: destructive command warnings + directory-scoped edits.
  Combines /careful (warns before rm -rf, DROP TABLE, force-push, etc.) with
  /freeze (blocks edits outside a specified directory). Use for maximum safety
  when touching prod or debugging live systems. Use when asked to "guard mode",
  "full safety", "lock it down", or "maximum safety".
  Based on gstack by Garry Tan, adapted for OpenClaw.
metadata:
  {
    "openclaw":
      {
        "emoji": "🛠️",
        "requires": { "bins": ["bash"] }
      }
  }
---

# /guard — Full Safety Mode

Activates both destructive command warnings and directory-scoped edit restrictions.
This is the combination of `/careful` + `/freeze` in a single command.

## Setup

Ask the user which directory to restrict edits to. Once the user provides a path:

1. Resolve it to an absolute path:
```bash
FREEZE_DIR=$(cd "<user-provided-path>" 2>/dev/null && pwd)
echo "$FREEZE_DIR"
```

2. Ensure trailing slash and save to the freeze state file:
```bash
FREEZE_DIR="${FREEZE_DIR%/}/"
STATE_DIR="$HOME/.openclaw/workspace/.freeze-state"
mkdir -p "$STATE_DIR"
echo "$FREEZE_DIR" > "$STATE_DIR/freeze-dir.txt"
echo "Freeze boundary set: $FREEZE_DIR"
```

Tell the user:
- "Guard mode active. Two protections are now running:"
- "1. Destructive command warnings — rm -rf, DROP TABLE, force-push, etc. will warn before executing (you can override)"
- "2. Edit boundary — file edits restricted to `<path>/`. Edits outside this directory are blocked."
- "To remove the edit boundary, run `/unfreeze`. To deactivate everything, say 'stop guard mode'."

## How it works in OpenClaw

OpenClaw doesn't have PreToolUse hooks. Instead, this skill works by instructing the
agent to self-enforce both protections:

1. Before any `exec` command: check against the destructive patterns from `/careful`
   (rm -rf, DROP TABLE, force-push, git reset --hard, kubectl delete, docker rm -f, etc.)
   If matched and not a safe exception, warn the user and ask for confirmation.

2. Before any `edit` or `write`: read the freeze state file and check if the target
   file is within the freeze boundary. If outside, refuse the edit and explain why.

## What's protected

See `/careful` for the full list of destructive command patterns and safe exceptions.
See `/freeze` for how edit boundary enforcement works.
