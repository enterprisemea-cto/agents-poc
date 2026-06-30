---
name: <ROLE_NAME>
description: <ROLE_DESCRIPTION>
model: <MODEL>
hooks:
  Stop:
    - hooks:
        - type: command
          command: "bash -c 'f=\"$CLAUDE_PROJECT_DIR/.claude/agent-memory/<ROLE_NAME>/MEMORY.md\"; grep -q \"^$(date +%F):\" \"$f\" 2>/dev/null || { echo \"Record this session before finishing: add a line dated $(date +%F) to .claude/agent-memory/<ROLE_NAME>/MEMORY.md (use the No-new-learnings line if nothing to record).\" >&2; exit 2; }'"
---

## On start — load memory

Before doing anything, read `.claude/agent-memory/<ROLE_NAME>/MEMORY.md` if it
exists. It holds transferable learnings from past sessions. Apply them.

## Role

<ROLE_BODY>

## Before finishing — write memory

<!-- ponytail: the Stop hook checks for an entry dated today (per-day granularity).
     It catches the first un-recorded run each day, not every session. True
     per-session enforcement would need a SubagentStart snapshot + marker file —
     not worth it for this POC. Upgrade path is in the design spec. -->

A Stop hook blocks you from finishing until
`.claude/agent-memory/<ROLE_NAME>/MEMORY.md` has an entry dated today. Write one
— a real learning or the No-new-learnings line — and the check passes (the model
can always satisfy it in one step, so it self-terminates; it is a backstop, not
an unbreakable wall). Keep `MEMORY.md` lean; bloated memory costs tokens on
every future run.

1. `mkdir -p .claude/agent-memory/<ROLE_NAME>`
2. Read existing `.claude/agent-memory/<ROLE_NAME>/MEMORY.md` first. If a similar
   entry exists, UPDATE it — do not append a duplicate.
3. Append new learnings in this exact 3-line shape:

   ```
   YYYY-MM-DD: <transferable rule, ≤15 words>.
     Why: <root cause or mechanism, ≤20 words>.
     Apply: <when this rule fires, ≤15 words>.
   ```

4. Save ONLY transferable patterns. SKIP task IDs, file:line citations, and
   anything derivable from the project docs or this agent file.
5. If the file grows past ~40 entries, merge duplicates before adding more.
6. Nothing new learned? Write exactly: `YYYY-MM-DD: No new learnings this session.`
