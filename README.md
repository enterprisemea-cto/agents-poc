# agents-poc

A Claude Code plugin that teaches **per-project agents, persistent memory, and task routing** by generating the files you learn from. You install the plugin, run two commands, and inspect what was produced.

---

## What this is

This plugin gives Claude Code two slash commands:

- `/build-team` — interviews you about your project, then generates per-role agent files, a routing table (`team.md`), and a patch for `CLAUDE.md`.
- `/dispatch "<task>"` — reads that routing table and delegates the task to the right agent, then chains a reviewer if one is paired.

The generated files are the lesson. You can read them to understand how Claude Code agents work, edit them to customise behaviour, and run the skills again to add more roles.

---

## Install

This plugin is a local directory containing `.claude-plugin/plugin.json`. To add it to Claude Code:

1. Clone or copy this repository to your machine.
2. In Claude Code, open your plugin settings and add it as a **local plugin directory**, pointing at the root of this repository.
3. Restart Claude Code. The `/build-team` and `/dispatch` commands will appear.

---

## Quickstart (happy path)

```
/build-team
```

Claude interviews you, proposes a roster (2–4 roles), asks a few targeted questions, then writes:

- `.claude/agents/<role>.md` — one file per role
- `.claude/team.md` — routing table and review pairings
- A `## Team` section appended to your `CLAUDE.md`

**Then run `/clear`.**

> `/clear` is required. Agent files written during a session are not registered in Claude Code's agent registry until you clear and restart. Skipping this step means `/dispatch` will fail with "agent not found".

```
/dispatch "implement the widget creation endpoint"
```

`/dispatch` reads `team.md`, scores the task against the routing notes, picks the best-matched role, delegates via `Task(subagent_type=...)`, and — if a reviewer is paired — chains that reviewer automatically once the implementation completes.

---

## Anatomy of a generated agent

Below is the real `builder` agent generated for a sample Widget API project. Every line is production output from the template-fill step.

```markdown
---
name: builder
description: Implements features for the Widget API.
model: opus
hooks:
  Stop:
    - hooks:
        - type: command
          command: "bash -c 'test -s \"$CLAUDE_PROJECT_DIR/.claude/agent-memory/builder/MEMORY.md\" || { echo \"Write a line to .claude/agent-memory/builder/MEMORY.md before finishing (use the No-new-learnings line if nothing to record).\" >&2; exit 2; }'"
---

## On start — load memory

Before doing anything, read `.claude/agent-memory/builder/MEMORY.md` if it
exists. It holds transferable learnings from past sessions. Apply them.

## Role

You implement features for the Widget API. Write the minimal code that satisfies each task, then hand off to the reviewer.

## Before finishing — write memory

A Stop hook nudges you to record learnings before you finish (it can block up
to a few times, then lets you through — it is a nudge, not a wall). Keep
`MEMORY.md` lean; bloated memory costs tokens on every future run.

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
command: "bash -c 'test -s \"$CLAUDE_PROJECT_DIR/.claude/agent-memory/builder/MEMORY.md\" || { echo \"...\"; exit 2; }'"
```

- **`$CLAUDE_PROJECT_DIR`** — Claude Code sets this to the project root at runtime. The hook uses it to find the memory file regardless of what directory the agent is working in.
- **`test -s`** — tests that the file exists AND is non-empty (`-s` = size > 0). An empty file still fails the test, so the agent cannot satisfy the hook by writing a blank file.
- **`exit 2`** — exit code 2 signals Claude Code to block the session from ending and surface the message to the agent. The agent sees the stderr message and knows to write the memory entry.
- **Nudge, not a wall** — Claude Code respects the environment variable `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP` (defaults to 8). After that many blocks, the hook is bypassed and the session closes. This prevents an infinite loop if the agent ignores the nudge.

### `## On start — load memory`

Instructs the agent to read its own `MEMORY.md` at the top of every session before doing any work. This is a plain prose instruction — there is no hook enforcing it. The discipline lives in the agent definition, not the shell.

### `## Before finishing — write memory`

The three-line entry shape:

```
YYYY-MM-DD: <transferable rule, ≤15 words>.
  Why: <root cause or mechanism, ≤20 words>.
  Apply: <when this rule fires, ≤15 words>.
```

- **`YYYY-MM-DD:`** — datestamp so entries age visibly. Old entries can be merged or dropped.
- **`Why:`** — the mechanism, not the symptom. Forces the agent to understand, not just record.
- **`Apply:`** — the trigger condition. Without it, a rule is unactionable.

---

## Memory

Memory files live at:

```
.claude/agent-memory/<role>/MEMORY.md
```

Each agent owns its own file. The file is never written by `/build-team` — only the agent itself writes it, after real sessions produce real learnings.

**Discipline rules:**

- **Transferable only.** Record patterns that apply across future sessions. Skip task IDs, ticket numbers, file:line citations, and anything derivable from the project docs.
- **Update, don't duplicate.** Before appending, read the existing file. If a similar entry exists, update it in place.
- **No-new-learnings line.** If nothing was learned, write `YYYY-MM-DD: No new learnings this session.` This satisfies the hook and keeps the file honest.
- **Size ceiling.** Past ~40 entries, merge duplicates before adding more. Bloated memory costs tokens on every future run.

---

## Make your own

**Edit a generated agent** — open `.claude/agents/<role>.md` and change the `## Role` section, the model (`opus`/`sonnet`), or the hook message. Changes take effect after `/clear`.

**Hand-write an agent** — create `.claude/agents/my-role.md` with the same frontmatter structure. The minimum viable file is a `name:`, `model:`, and a `## Role` block.

**Add more roles** — run `/build-team` again. It detects existing agent files and asks before overwriting (`overwrite / version-suffix / skip`).
