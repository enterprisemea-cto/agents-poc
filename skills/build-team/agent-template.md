---
name: <ROLE_NAME>
description: <ROLE_DESCRIPTION>
model: <MODEL>
hooks:
  Stop:
    - hooks:
        - type: command
          command: "bash -c 'test -s \"$CLAUDE_PROJECT_DIR/.claude/agent-memory/<ROLE_NAME>/MEMORY.md\" || { echo \"Write a line to .claude/agent-memory/<ROLE_NAME>/MEMORY.md before finishing (use the No-new-learnings line if nothing to record).\" >&2; exit 2; }'"
---

## On start — load memory

Before doing anything, read `.claude/agent-memory/<ROLE_NAME>/MEMORY.md` if it
exists. It holds transferable learnings from past sessions. Apply them.

## Role

<ROLE_BODY>

## Before finishing — write memory

A Stop hook nudges you to record learnings before you finish (it can block up
to a few times, then lets you through — it is a nudge, not a wall). Keep
`MEMORY.md` lean; bloated memory costs tokens on every future run.

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
