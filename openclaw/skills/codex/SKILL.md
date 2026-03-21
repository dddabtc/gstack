---
name: codex
description: >
  OpenAI Codex CLI wrapper — three modes. Code review: independent diff review via
  codex review with pass/fail gate. Challenge: adversarial mode that tries to break
  your code. Consult: ask codex anything with session continuity for follow-ups.
  The "200 IQ autistic developer" second opinion. Use when asked to "codex review",
  "codex challenge", "ask codex", "second opinion", or "consult codex".
  Based on gstack by Garry Tan, adapted for OpenClaw.
---

# /codex — Multi-AI Second Opinion

You are running the `/codex` skill. This wraps the OpenAI Codex CLI to get an independent,
brutally honest second opinion from a different AI system.

Codex is the "200 IQ autistic developer" — direct, terse, technically precise, challenges
assumptions, catches things you might miss. Present its output faithfully, not summarized.

## Step 0: Check codex binary

```bash
CODEX_BIN=$(which codex 2>/dev/null || echo "")
[ -z "$CODEX_BIN" ] && echo "NOT_FOUND" || echo "FOUND: $CODEX_BIN"
```

If `NOT_FOUND`: stop and tell the user:
"Codex CLI not found. Install it: `npm install -g @openai/codex` or see https://github.com/openai/codex"

## Step 0.5: Detect base branch

```bash
BASE=$(gh pr view --json baseRefName -q .baseRefName 2>/dev/null || \
       gh repo view --json defaultBranchRef -q .defaultBranchRef.name 2>/dev/null || \
       echo "main")
echo "Base branch: $BASE"
```

## Step 1: Detect mode

Parse the user's input to determine which mode to run:

1. `/codex review` or `/codex review <instructions>` — Review mode (Step 2A)
2. `/codex challenge` or `/codex challenge <focus>` — Challenge mode (Step 2B)
3. `/codex` with no arguments — Auto-detect:
   - Check for a diff: `git diff origin/$BASE --stat 2>/dev/null | tail -1`
   - If a diff exists, ask the user:
     - A) Review the diff (code review with pass/fail gate)
     - B) Challenge the diff (adversarial — try to break it)
     - C) Something else — I'll provide a prompt
   - If no diff, ask: "What would you like to ask Codex?"
4. `/codex <anything else>` — Consult mode (Step 2C)

## Step 2A: Review Mode

Run Codex code review against the current branch diff.

1. Create temp files:
```bash
TMPERR=$(mktemp /tmp/codex-err-XXXXXX.txt)
```

2. Run the review (5-minute timeout):
```bash
timeout 300 codex review --base $BASE -c 'model_reasoning_effort="xhigh"' --enable web_search_cached 2>"$TMPERR"
```

If the user provided custom instructions (e.g., `/codex review focus on security`):
```bash
timeout 300 codex review "focus on security" --base $BASE -c 'model_reasoning_effort="xhigh"' --enable web_search_cached 2>"$TMPERR"
```

3. Parse cost from stderr:
```bash
grep "tokens used" "$TMPERR" 2>/dev/null || echo "tokens: unknown"
```

4. Determine gate verdict: `[P1]` markers = FAIL, otherwise PASS.

5. Present the output:
```
CODEX SAYS (code review):
════════════════════════════════════════════════════════════
<full codex output, verbatim — do not truncate or summarize>
════════════════════════════════════════════════════════════
GATE: PASS/FAIL                    Tokens: N
```

6. Cross-model comparison: If `/review` was already run earlier in this conversation,
   compare the two sets of findings (agreement rate, unique findings per model).

7. Clean up:
```bash
rm -f "$TMPERR"
```

## Step 2B: Challenge (Adversarial) Mode

Codex tries to break your code — finding edge cases, race conditions, security holes.

1. Construct the adversarial prompt:

Default (no focus):
"Review the changes on this branch against the base branch. Run `git diff origin/$BASE` to see the diff. Your job is to find ways this code will fail in production. Think like an attacker and a chaos engineer. Find edge cases, race conditions, security holes, resource leaks, failure modes, and silent data corruption paths. Be adversarial. Be thorough. No compliments — just the problems."

With focus (e.g., "security"):
"Review the changes on this branch against the base branch. Run `git diff origin/$BASE` to see the diff. Focus specifically on SECURITY. Your job is to find every way an attacker could exploit this code."

2. Run codex exec with JSONL output (5-minute timeout):
```bash
timeout 300 codex exec "<prompt>" -s read-only -c 'model_reasoning_effort="xhigh"' --enable web_search_cached --json 2>/dev/null | python3 -c "
import sys, json
for line in sys.stdin:
    line = line.strip()
    if not line: continue
    try:
        obj = json.loads(line)
        t = obj.get('type','')
        if t == 'item.completed' and 'item' in obj:
            item = obj['item']
            itype = item.get('type','')
            text = item.get('text','')
            if itype == 'reasoning' and text:
                print(f'[codex thinking] {text}')
                print()
            elif itype == 'agent_message' and text:
                print(text)
            elif itype == 'command_execution':
                cmd = item.get('command','')
                if cmd: print(f'[codex ran] {cmd}')
        elif t == 'turn.completed':
            usage = obj.get('usage',{})
            tokens = usage.get('input_tokens',0) + usage.get('output_tokens',0)
            if tokens: print(f'\ntokens used: {tokens}')
    except: pass
"
```

3. Present the full output:
```
CODEX SAYS (adversarial challenge):
════════════════════════════════════════════════════════════
<full output, verbatim>
════════════════════════════════════════════════════════════
Tokens: N
```

## Step 2C: Consult Mode

Ask Codex anything about the codebase. Supports session continuity.

1. Check for existing session:
```bash
cat .context/codex-session-id 2>/dev/null || echo "NO_SESSION"
```

If a session exists, ask: continue or start fresh?

2. Run codex exec with JSONL output (5-minute timeout):

For a new session:
```bash
timeout 300 codex exec "<prompt>" -s read-only -c 'model_reasoning_effort="xhigh"' --enable web_search_cached --json 2>/dev/null | python3 -c "
import sys, json
for line in sys.stdin:
    line = line.strip()
    if not line: continue
    try:
        obj = json.loads(line)
        t = obj.get('type','')
        if t == 'thread.started':
            tid = obj.get('thread_id','')
            if tid: print(f'SESSION_ID:{tid}')
        elif t == 'item.completed' and 'item' in obj:
            item = obj['item']
            itype = item.get('type','')
            text = item.get('text','')
            if itype == 'reasoning' and text:
                print(f'[codex thinking] {text}')
                print()
            elif itype == 'agent_message' and text:
                print(text)
            elif itype == 'command_execution':
                cmd = item.get('command','')
                if cmd: print(f'[codex ran] {cmd}')
        elif t == 'turn.completed':
            usage = obj.get('usage',{})
            tokens = usage.get('input_tokens',0) + usage.get('output_tokens',0)
            if tokens: print(f'\ntokens used: {tokens}')
    except: pass
"
```

For a resumed session:
```bash
timeout 300 codex exec resume <session-id> "<prompt>" -s read-only -c 'model_reasoning_effort="xhigh"' --enable web_search_cached --json 2>/dev/null | python3 -c "<same parser>"
```

3. Save session ID from `SESSION_ID:` line:
```bash
mkdir -p .context
echo "<session-id>" > .context/codex-session-id
```

4. Present the output:
```
CODEX SAYS (consult):
════════════════════════════════════════════════════════════
<full output, verbatim>
════════════════════════════════════════════════════════════
Tokens: N
Session saved — run /codex again to continue this conversation.
```

5. Flag any disagreements: "Note: I disagree on X because Y."

## Model & Reasoning

- No model is hardcoded — codex uses its current default
- All modes use `xhigh` reasoning effort
- All commands use `--enable web_search_cached`
- If the user specifies a model, pass `-m` through to codex

## Error Handling

- Binary not found: Stop with install instructions
- Auth error: "Run `codex login` in your terminal to authenticate"
- Timeout: "Codex timed out after 5 minutes. Try a smaller scope."
- Empty response: "Codex returned no response. Check stderr for errors."
- Session resume failure: Delete session file and start fresh

## Important Rules

- Never modify files. This skill is read-only.
- Present output verbatim. Do not truncate or summarize.
- Add synthesis after, not instead of, the full output.
- 5-minute timeout on all codex calls.
