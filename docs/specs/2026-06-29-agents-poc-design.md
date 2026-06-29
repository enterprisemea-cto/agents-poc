# Agents POC — per-project agents with memory + routing

**Date:** 2026-06-29
**Status:** Approved design, pre-plan

## Purpose

A shippable, installable plugin that teaches people how to build **their own
custom, per-project agents** that (a) carry **memory** across sessions and
(b) can be **routed and chained** for multi-step workflows.

It is a stripped-down distillation of three internal skills — `staff` (team
assembly), `persona` (per-agent customization + memory), and `dispatch`
(routing + reviewer chaining) — merged and de-coupled from the author's
machine so anyone can install and run it. The generated files are themselves
the lesson: a learner opens a generated agent and sees exactly how memory and
routing work, then edits it.

## What it is NOT

- Not a copy of `staff`/`persona`/`dispatch`. A fresh, third thing.
- Not dependent on global base agents (`~/.claude/agents/*`). Generated agents
  are **standalone** — the single biggest strip from the internal system.
- Not a code-analysis tool. Target projects are early-stage and
  **document-driven** (Markdown briefs/PRDs/notes), not built codebases.

## Deliverable

One installable plugin:

```
agents-poc/
  .claude-plugin/plugin.json       # plugin manifest
  skills/build-team/SKILL.md       # interview → agents + team.md + CLAUDE.md
  skills/dispatch/SKILL.md          # route task → role → chain reviewer
  README.md                         # the lesson — walks a real generated agent file
```

No shipped hook script — the memory nudge is inlined into each generated agent
(see Inline memory hook). Plugin ships two skills + a manifest + README.

## Components

### Skill 1 — `/build-team`

Merges `staff` + `persona`, stripped to standalone + doc-driven. Flow:

**1. Gather context — three sources (pull from all three).**

  1. **Conversation context** — prior turns of the current session (what the
     user discussed before invoking the skill). Authoritative for intent.
  2. **Doc scan** — auto-read project Markdown: `README*`, `docs/**/*.md`,
     root-level `*.md` (briefs, PRDs, plans, notes), `CLAUDE.md`. Bounded:
     ~8 files max, ~200 lines each. No source-code sampling.
  3. **User-pointed context** — explicitly ask: *"Anywhere else I should look?
     Paths, docs, folders, URLs — or 'no'."* Read what they point to,
     including dirs outside the project or a web URL.

**2. Propose roster.** Synthesize all three sources → draft 2–4 roles with a
one-line why-needed each. Nudge toward **≥2 roles** so dispatch has something
to route between and chain. Present roster; accept `accept` / `add` /
`remove` / `swap` / natural-language changes. Loop until accepted.

**3. Interview gaps only.** Scan + convo do the heavy lifting; ask 0–4
follow-ups one at a time only to fill what's missing (project stage, role
boundaries, must-haves). Respect "skip" / short answers → build thin.

**4. Generate standalone agent files.** For each accepted role, write
`.claude/agents/<role>.md`. Each file is self-contained (no global base
dependency) and contains:
  - **Role definition** — synthesized from gathered context.
  - **Load-memory-on-start block** — a single line: on invocation, read
    `.claude/agent-memory/<role>/MEMORY.md` if present. (NOT the persona
    Step 0 lookup — see Strips.)
  - **Write-memory-before-finish block** — append disciplined learnings
    before completing (see Memory below). Anti-bloat is prose guidance to the
    agent — NO `caveman:compress` skill call (see Strips).
  - **Inline Stop hook** — the memory-nudge logic lives directly in the agent
    frontmatter, not an external script (see Memory / hook below).

**5. Write `.claude/team.md`.** Roster table + routing notes + review
pairings. Routing notes map work types (verbs, doc topics, file kinds) →
roles. Review pairings default: one role reviews the others; single-role
projects have no pairing (chaining skipped).

**Tie-break needs explicit metadata.** Standalone roles have no inherent
seniority, so `/dispatch`'s "senior wins ties" has nothing to compare against.
`/build-team` therefore writes an explicit **priority order** into `team.md`'s
routing notes (the order roles were proposed, or a "primary"/"reviewer" tag).
`/dispatch` reads that; if genuinely tied with no priority signal, it asks
rather than guessing.

**6. Patch project `CLAUDE.md` — after confirmation.** Show the proposed
`## Team` section (delegation rule + roster table) and ask before writing.
Walk up to nearest `CLAUDE.md`; replace an existing `## Team` section or
append; create `.claude/CLAUDE.md` if none exists. Never modify content
outside the `## Team` section.

### Skill 2 — `/dispatch <task>`

Runtime router. Based on the internal `dispatch` skill, minus the
"no team = run /staff" coupling.

  1. Walk up to nearest `.claude/team.md`. If absent → tell user to run
     `/build-team`. Stop.
  2. Match task against routing notes → pick highest-scoring role (senior
     wins ties). Split into sequential dispatches if the task spans two roles.
  3. `Task(subagent_type=<role>, prompt=<task> + context pointer to team.md)`.
  4. **Chain reviewer** — if the dispatched role has a review pairing,
     auto-dispatch the reviewer with a summary of the implementer's output.
     This is the visible "chaining" payoff.

**Registration gotcha (must document).** Agent files written to
`.claude/agents/` mid-session are not in the registry until `/clear`. So the
first `Task(subagent_type=<role>)` fails with "agent not found" right after
`/build-team`. `/build-team`'s closing handoff AND `/dispatch` AND the README
must tell the user to `/clear` once before dispatching. Without this the very
first run fails mysteriously.

### Inline memory hook (no external script)

The memory nudge is an **inline Stop hook in each generated agent's
frontmatter** — no shipped script, no path variable. `${CLAUDE_PLUGIN_ROOT}`
does NOT expand inside a project agent file (it's scoped to the plugin's own
bundled files, not files the plugin writes into a user's repo), so the
original "ship a script, reference via `${CLAUDE_PLUGIN_ROOT}`" approach would
silently no-op. Inlining also serves the teaching goal: the learner reads the
entire hook logic in the agent file.

Shape (per generated agent, `<role>` substituted):
```yaml
hooks:
  Stop:
    - hooks:
        - type: command
          command: "bash -c 'test -s \"$CLAUDE_PROJECT_DIR/.claude/agent-memory/<role>/MEMORY.md\" || { echo \"Write MEMORY.md before finishing\" >&2; exit 2; }'"
```

- Uses `$CLAUDE_PROJECT_DIR` (documented hook anchor), not a plugin variable.
- A `Stop` matcher in a custom subagent's frontmatter is auto-converted to
  `SubagentStop` at runtime — generated project agents are "custom" subagents
  and ARE allowed hooks (the plugin-subagent restriction does not apply here).
- **Nudge, not enforcement.** Claude Code overrides a blocking Stop hook after
  8 consecutive blocks (`CLAUDE_CODE_STOP_HOOK_BLOCK_CAP`). Describe this as a
  strong nudge with an 8-try ceiling — NOT "blocks until written."
- **First-run safety.** Writing the `YYYY-MM-DD: No new learnings.` line
  satisfies the `test -s` check (file becomes non-empty), so a brand-new agent
  with nothing to record can finish. The memory-write block must instruct the
  agent to write that line when there's nothing new.
- Detection is **presence/non-empty**, not "written this session." For a
  teaching POC that's the right simplicity — avoids transcript/sentinel
  coupling. (A `SubagentStart` sentinel for true per-session proof is possible
  but deliberately out of scope.)

## Memory model

- Location: `.claude/agent-memory/<role>/MEMORY.md`, one dir per role.
- Entry shape (disciplined, 3 lines):
  ```
  YYYY-MM-DD: <transferable rule, ≤15 words>.
    Why: <root cause or mechanism, ≤20 words>.
    Apply: <when this rule fires, ≤15 words>.
  ```
- Anti-bloat rules: read existing MEMORY first and UPDATE duplicates rather
  than append; skip task IDs / file:line citations / facts derivable from the
  docs or the agent file; compress when the file grows large.
- Memory dirs are owned by the agents — `/build-team` never overwrites them.

## Explicit strips (from the internal system)

These are not optional cleanups — they are correctness/portability fixes. If
the implementer starts from `standalone-agent-template.md`, several break the
POC on a fresh machine and must be cut explicitly:

- **Standalone agents** — drop the `~/.claude/agents/*` base-extending model.
- **No code signal-scan** — drop entry-point/route/schema/`.env` sampling;
  scan Markdown docs only.
- **No dispatch→staff coupling** — dispatch points at whatever `team.md` exists.
- **No machine-specific paths and no `${CLAUDE_PLUGIN_ROOT}`** — memory hook is
  inlined against `$CLAUDE_PROJECT_DIR` (the plugin variable does not expand in
  a written project agent file).
- **Cut the persona Step 0 lookup** (`standalone-agent-template.md`) — it walks
  up to `.claude/personas/`, a dir the POC drops. Dead, confusing code for a
  learner. Replace with the one-line memory-load.
- **Cut the `caveman:compress` skill call** (`standalone-agent-template.md`) —
  hard dependency on the author's caveman plugin, absent on fresh install.
  Anti-bloat becomes prose guidance.
- **Drop `permissionMode: bypassPermissions`** from the template — shipping it
  to strangers silently bypasses all permission prompts. Use the default flow.
- Drop review-pairing complexity beyond "one role reviews the rest."

## README — the lesson (required structure)

The pedagogy claim lives or dies here. README MUST walk a learner through one
**real generated agent file**, pointing at and explaining: the memory-load
line, the memory-write block, and the inline Stop hook. Plus: the install
step, the `/clear`-to-register step, and the `/build-team` → `/clear` →
`/dispatch` happy path. "Files are the lesson" is only true if the README
annotates an actual file.

## Success criteria

A learner can: install the plugin → run `/build-team` in an early-stage,
doc-driven project → get standalone agents + `team.md` + a confirmed CLAUDE.md
patch → **`/clear`** (registers the new agents) → run `/dispatch "<task>"` and
watch it route to a role and chain a reviewer → run again next session and see
agents read/write `MEMORY.md`. Then they open a generated agent file and
understand the whole pattern well enough to edit it or build a team by hand.

## Open / deferred

- Plugin manifest format details (`plugin.json` fields) — resolve at plan time.
- True per-session memory proof (`SubagentStart` sentinel) — deliberately OUT
  of scope; POC uses a presence/non-empty check.
- This POC directory is not a git repo. Spec is committed to `docs/specs/`
  only if the user opts to `git init`; otherwise it lives as a plain file.
