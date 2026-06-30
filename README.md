# agents-poc

A Claude Code plugin that teaches **per-project agents, persistent memory, and task routing** — by generating the files you learn from. Install it, run two commands, read what was produced.

---

## What this is

Two slash commands:

- `/agents-poc:build-team` — interviews you, then generates per-role agent files, a routing table (`team.md`), and a `CLAUDE.md` patch.
- `/agents-poc:dispatch "<task>"` — reads the routing table, delegates to the best-matched agent, then chains a reviewer if one is paired.

Both are description-matched, so plain-language requests trigger them too — no slash command required.

The generated files are the lesson: read them to learn how agents work, edit them to change behaviour, re-run to add roles.

---

## Install

```
/plugin marketplace add enterprisemea-cto/agents-poc
/plugin install agents-poc@agents-poc
```

- `marketplace add` takes an `<owner>/<repo>` slug or the full `https://github.com/enterprisemea-cto/agents-poc` URL.
- Install argument is `<plugin>@<marketplace>` — both are `agents-poc`.
- Restart Claude Code when prompted so the skills load.

**Developing the plugin?** Point Claude Code at a local clone:

```
git clone https://github.com/enterprisemea-cto/agents-poc
claude --plugin-dir /path/to/agents-poc
```

Or register the path as a marketplace: `/plugin marketplace add /path/to/agents-poc`.

---

## Quickstart

```
/agents-poc:build-team
```

Claude interviews you, proposes a 2–4 role roster, asks a few targeted questions, then writes:

- `.claude/agents/<role>.md` — one per role
- `.claude/team.md` — routing table and review pairings
- a `## Team` section in `CLAUDE.md` (after you confirm)

**Then run `/clear`** — required. Agent files written mid-session aren't registered until you clear and restart; skip it and `dispatch` fails with "agent not found".

```
/agents-poc:dispatch "implement the widget creation endpoint"
```

`dispatch` reads `team.md`, scores the task against the routing notes, picks the best role, delegates via `Task(subagent_type=...)` (Claude Code's sub-task mechanism), and chains the paired reviewer once the implementation completes.

---

## Anatomy of a generated agent

The real `builder` agent from a sample Widget API project — production output from the template-fill step:

```markdown
---
name: builder
description: Implements features for the Widget API.
model: opus
hooks:
  Stop:
    - hooks:
        - type: command
          command: "bash -c 'f=\"$CLAUDE_PROJECT_DIR/.claude/agent-memory/builder/MEMORY.md\"; grep -q \"^$(date +%F):\" \"$f\" 2>/dev/null || { echo \"Record this session before finishing: add a line dated $(date +%F) to .claude/agent-memory/builder/MEMORY.md (use the No-new-learnings line if nothing to record).\" >&2; exit 2; }'"
---

## On start — load memory

Before doing anything, read `.claude/agent-memory/builder/MEMORY.md` if it
exists. It holds transferable learnings from past sessions. Apply them.

## Role

You implement features for the Widget API. Write the minimal code that satisfies each task, then hand off to the reviewer.

## Before finishing — write memory

A Stop hook blocks you from finishing until
`.claude/agent-memory/builder/MEMORY.md` has an entry dated today. Write one —
a real learning or the No-new-learnings line — and the check passes (the model
can always satisfy it in one step, so it self-terminates; it is a backstop, not
an unbreakable wall). Keep `MEMORY.md` lean; bloated memory costs tokens on
every future run.

1. `mkdir -p .claude/agent-memory/builder`
2. Read existing `.claude/agent-memory/builder/MEMORY.md` first. If a similar
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
```

### The inline Stop hook

```yaml
command: "bash -c 'f=\"$CLAUDE_PROJECT_DIR/.claude/agent-memory/builder/MEMORY.md\"; grep -q \"^$(date +%F):\" \"$f\" 2>/dev/null || { echo \"...\"; exit 2; }'"
```

- **`$CLAUDE_PROJECT_DIR`** — set to the project root at runtime; locates the memory file regardless of the agent's working directory.
- **`grep -q "^$(date +%F):"`** — checks whether `MEMORY.md` has a line dated today (`date +%F` = `YYYY-MM-DD`). Missing → the hook fires. Any dated line satisfies it, including the `No new learnings this session.` line.
- **`exit 2`** — signals Claude Code to block session end and surface the stderr message to the agent.
- **Backstop, not a wall** — self-satisfiable: one dated line and the next check passes (exit 0), so flow ends in one retry. The nudge is **per-day** — it catches the first un-recorded run each day, not every session. That ceiling is deliberate; true per-session enforcement would need a `SubagentStart` snapshot (see the design spec).

### `## On start — load memory`

Prose instruction to read `MEMORY.md` before any work. No hook enforces it — the discipline lives in the agent definition, not the shell.

### `## Before finishing — write memory`

The three-line entry shape:

```
YYYY-MM-DD: <transferable rule, ≤15 words>.
  Why: <root cause or mechanism, ≤20 words>.
  Apply: <when this rule fires, ≤15 words>.
```

- **`YYYY-MM-DD:`** — datestamp so entries age visibly and can be merged or dropped.
- **`Why:`** — the mechanism, not the symptom. Forces understanding, not just recording.
- **`Apply:`** — the trigger condition. Without it a rule is unactionable.

---

## Memory

Files live at `.claude/agent-memory/<role>/MEMORY.md` — one per agent. Never written by `build-team`; only the agent itself writes it, after real sessions produce real learnings.

**Discipline:**

- **Transferable only.** Skip task IDs, ticket numbers, file:line citations, and anything derivable from the project docs.
- **Update, don't duplicate.** Read the file first; update a similar entry in place.
- **No-new-learnings line.** Nothing learned? Write `YYYY-MM-DD: No new learnings this session.` — satisfies the hook, keeps the file honest.
- **Size ceiling.** Past ~40 entries, merge duplicates before adding more.

---

## Make your own

- **Edit an agent** — change the `## Role`, the model (`opus`/`sonnet`), or the hook message in `.claude/agents/<role>.md`. Takes effect after `/clear`.
- **Hand-write one** — create `.claude/agents/my-role.md` with the same frontmatter. Minimum viable: `name:`, `model:`, and a `## Role` block.
- **Add roles** — re-run `/agents-poc:build-team`. It detects existing files and asks before overwriting (`overwrite / version-suffix / skip`).
