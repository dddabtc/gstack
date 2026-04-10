---
name: gstack-upgrade
description: >
  Information about gstack upgrade process. Since OpenClaw uses its own skill
  installation system, this skill explains how gstack skills are managed in
  OpenClaw. Use when asked to "upgrade gstack", "update gstack skills", or
  "get latest gstack version". Based on gstack by Garry Tan, adapted for OpenClaw.
metadata:
  {
    "openclaw":
      {
        "emoji": "🛠️",
        "requires": { "bins": ["bash"] }
      }
  }
---

# /gstack-upgrade — Skill Management in OpenClaw

In OpenClaw, gstack skills are installed as individual skill directories under
`~/.openclaw/workspace/skills/`. They don't use gstack's git-based upgrade system.

## How skills are managed

Skills live in: `~/.openclaw/workspace/skills/<skill-name>/SKILL.md`

To update a skill:
1. Get the latest version of the skill's SKILL.md
2. Replace the file in the skill directory

To check what's installed:
```bash
ls ~/.openclaw/workspace/skills/
```

To see a skill's content:
```bash
cat ~/.openclaw/workspace/skills/<name>/SKILL.md
```

## Updating from gstack upstream

If the user wants to pull the latest gstack skill updates:

1. Clone or pull the latest gstack repo:
```bash
cd /tmp && git clone --depth 1 https://github.com/garrytan/gstack.git gstack-latest
```

2. Check if OpenClaw-adapted versions exist:
```bash
ls /tmp/gstack-latest/openclaw/skills/
```

3. Copy updated skills to the workspace:
```bash
cp /tmp/gstack-latest/openclaw/skills/<name>/SKILL.md ~/.openclaw/workspace/skills/<name>/SKILL.md
```

4. Clean up:
```bash
rm -rf /tmp/gstack-latest
```

## Notes

- OpenClaw skills don't use gstack's preamble, telemetry, or auto-upgrade system
- Skills are adapted to use OpenClaw's built-in tools (browser, exec, web_search, etc.)
- Custom skills (camera, overlay-release, technical-cofounder, x-access) are independent of gstack
