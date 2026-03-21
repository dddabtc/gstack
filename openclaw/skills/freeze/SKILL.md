---
name: freeze
description: >
  Restrict file edits to a specific directory for the session. Blocks edits
  outside the allowed path. Use when debugging to prevent accidentally
  "fixing" unrelated code, or when you want to scope changes to one module.
  Use when asked to "freeze", "restrict edits", "only edit this folder",
  or "lock down edits". Based on gstack by Garry Tan, adapted for OpenClaw.
---

# /freeze — Restrict Edits to a Directory

Lock file edits to a specific directory. Any `edit` or `write` operation targeting
a file outside the allowed path will be blocked (not just warned).

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

Tell the user: "Edits are now restricted to `<path>/`. Any edit or write
outside this directory will be blocked. To change the boundary, run `/freeze`
again. To remove it, run `/unfreeze`."

## How it works in OpenClaw

OpenClaw doesn't have PreToolUse hooks. Instead, this skill works by instructing the
agent to self-check every `edit` and `write` call:

1. Before any `edit` or `write`, read the freeze state:
   ```bash
   cat "$HOME/.openclaw/workspace/.freeze-state/freeze-dir.txt" 2>/dev/null
   ```
2. If a freeze boundary exists, check if the target file path starts with the freeze directory
3. If outside the boundary: refuse the edit and tell the user why
4. If inside the boundary or no freeze is set: proceed normally

## Notes

- The trailing `/` on the freeze directory prevents `/src` from matching `/src-old`
- Freeze applies to edit and write operations only — read and exec are unaffected
- This prevents accidental edits, not a security boundary — shell commands can still modify files outside the boundary
- To deactivate, run `/unfreeze`
