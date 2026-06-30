# Agents POC Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an installable Claude Code plugin that teaches people to create per-project standalone agents with memory, plus routing/chaining via dispatch.

**Architecture:** A plugin shipping two skills (`build-team`, `dispatch`) + two file templates + a README. `build-team` interviews/scans docs → writes standalone agent files (each with an inline memory Stop-hook) + `team.md` + a confirmed CLAUDE.md patch. `dispatch` routes a task to a role and chains a reviewer. No external scripts, no global-agent dependency.

**Tech Stack:** Markdown (skills, templates, README), JSON (plugin manifest), bash (inline hook one-liner, verification). No build system, no test framework — verification is shell assertions.

## Global Constraints

- **Standalone only** — generated agents never reference `~/.claude/agents` or `.claude/personas/`.
- **No machine-specific paths and no `${CLAUDE_PLUGIN_ROOT}`** in any generated/project file — the memory hook is inlined against `$CLAUDE_PROJECT_DIR`.
- **No author-plugin dependencies** — no `caveman:compress` (or any `caveman`/`ponytail`/`context-mode`) calls in shipped or generated files.
- **No `permissionMode: bypassPermissions`** in templates or generated agents.
- **Memory = nudge, not enforcement** — copy says "nudge"; Claude Code's `stop_hook_active` loop protection prevents an infinite block. Never "blocks until written."
- **Memory entry shape** verbatim: 3 lines — `YYYY-MM-DD: <rule ≤15w>.` / `  Why: <≤20w>.` / `  Apply: <≤15w>.`
- Spec of record: `docs/specs/2026-06-29-agents-poc-design.md`.

### Forbidden-token gate (run after every file-authoring task)

```bash
# Must return ZERO matches across shipped skills/templates:
grep -rnE 'CLAUDE_PLUGIN_ROOT|bypassPermissions|caveman|ponytail|context-mode|\.claude/personas|~/\.claude/agents' \
  skills/ README.md .claude-plugin/ 2>/dev/null
```
Expected: no output (exit 1 from grep = no matches = pass).

---

### Task 0: Repo + plugin scaffold

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create dirs: `skills/build-team/`, `skills/dispatch/`

**Interfaces:**
- Produces: a valid plugin manifest discoverable by Claude Code; skills auto-load from `skills/*/SKILL.md`.

- [ ] **Step 1: Init repo**

```bash
cd /home/tomic/Sync/clients/enterpriseam.com/34_agents-poc
git init
printf '%s\n' '.DS_Store' 'scratch/' > .gitignore
mkdir -p skills/build-team skills/dispatch .claude-plugin
```

- [ ] **Step 2: Write `.claude-plugin/plugin.json`**

```json
{
  "name": "agents-poc",
  "version": "0.1.0",
  "description": "Teach yourself to build per-project agents with memory, routing, and reviewer chaining.",
  "author": { "name": "Mario" }
}
```

- [ ] **Step 3: Verify manifest is valid JSON**

Run: `jq -e . .claude-plugin/plugin.json`
Expected: pretty-printed JSON, exit 0. If `claude plugin validate` exists in this CLI, also run it and expect pass.

- [ ] **Step 4: Commit**

```bash
git add -A && git commit -m "chore: scaffold agents-poc plugin"
```

---

### Task 1: Standalone agent template (the crown jewel)

**Files:**
- Create: `skills/build-team/agent-template.md`

**Interfaces:**
- Consumes: nothing.
- Produces: the template `build-team` fills per role. Placeholders (literal, `<UPPERCASE>`): `<ROLE_NAME>`, `<ROLE_DESCRIPTION>`, `<MODEL>`, `<ROLE_BODY>`. `build-team` substitutes every occurrence — including the two inside the hook command and the memory blocks.

- [ ] **Step 1: Write the template**

````markdown
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
````

- [ ] **Step 2: Verify hook command is valid bash**

Extract and dry-parse the inline command (substitute a sample role):

Run:
```bash
ROLE=demo
bash -n -c 'test -s "$CLAUDE_PROJECT_DIR/.claude/agent-memory/'"$ROLE"'/MEMORY.md" || { echo "x" >&2; exit 2; }'
echo "parse-exit:$?"
```
Expected: `parse-exit:0` (syntax valid; `bash -n` does not execute).

- [ ] **Step 3: Verify frontmatter shape**

Run: `grep -c 'CLAUDE_PROJECT_DIR' skills/build-team/agent-template.md`
Expected: `1` (the hook command — the only place a runtime path var is used). Also confirm `exit 2` and `test -s` are present.

- [ ] **Step 4: Forbidden-token gate** — run the Global Constraints gate. Expected: no matches.

- [ ] **Step 5: Commit**

```bash
git add skills/build-team/agent-template.md && git commit -m "feat: standalone agent template with inline memory hook"
```

---

### Task 2: team.md template

**Files:**
- Create: `skills/build-team/team-template.md`

**Interfaces:**
- Produces: the manifest `build-team` fills. Placeholders: `<PROJECT_NAME>`, `<BUILD_DATE>`, `<SOURCE_LINE>`, `<ROSTER_ROWS>`, `<PRIORITY_ORDER>`, `<ROUTING_NOTES>`, `<REVIEW_PAIRINGS>`. `dispatch` reads `<PRIORITY_ORDER>` for tie-breaks.

- [ ] **Step 1: Write the template**

```markdown
# Team — <PROJECT_NAME>

Built by /build-team on <BUILD_DATE>. Source: <SOURCE_LINE>.

## Roster

| Role | Agent | When to use |
|------|-------|-------------|
<ROSTER_ROWS>

## Priority order

Highest-priority role first. `/dispatch` uses this to break ties.

<PRIORITY_ORDER>

## Routing notes

<ROUTING_NOTES>

## Review pairings

After an implementer finishes, /dispatch chains its reviewer:

<REVIEW_PAIRINGS>
```

- [ ] **Step 2: Verify all placeholders present**

Run:
```bash
for p in PROJECT_NAME BUILD_DATE SOURCE_LINE ROSTER_ROWS PRIORITY_ORDER ROUTING_NOTES REVIEW_PAIRINGS; do
  grep -q "<$p>" skills/build-team/team-template.md || echo "MISSING:$p"; done
echo done
```
Expected: only `done` (no `MISSING:` lines).

- [ ] **Step 3: Commit**

```bash
git add skills/build-team/team-template.md && git commit -m "feat: team.md template"
```

---

### Task 3: `build-team` SKILL.md

**Files:**
- Create: `skills/build-team/SKILL.md`

**Interfaces:**
- Consumes: `agent-template.md`, `team-template.md` (sibling files, referenced by relative path).
- Produces: the `/build-team` flow. Writes `.claude/agents/<role>.md`, `.claude/team.md`, and a confirmed `## Team` patch in the nearest `CLAUDE.md`.

- [ ] **Step 1: Author SKILL.md** with YAML frontmatter `name: build-team` and a `description:` that triggers on "build agents / set up a team / onboard agents / create custom agents". Body MUST contain these sections, in order, with concrete instructions (no placeholders, no TODOs):

  1. **Gather context (3 sources)** — (a) prior conversation turns; (b) doc scan: read `README*`, `docs/**/*.md`, root `*.md`, `CLAUDE.md`, capped ~8 files × ~200 lines, **Markdown only, no source-code sampling**; (c) explicitly ask once: *"Anywhere else I should look? Paths, docs, folders, URLs — or 'no'."* and read what's pointed to.
  2. **Propose roster** — synthesize all three → 2–4 roles, one-line why each. Nudge toward ≥2 so dispatch has something to route/chain. Present; accept `accept`/`add`/`remove`/`swap`/natural-language; loop until accepted.
  3. **Interview gaps only** — 0–4 follow-ups, one at a time, only for missing info (stage, role boundaries). Respect "skip"/short answers.
  4. **Generate agents** — for each role: read `agent-template.md`, substitute `<ROLE_NAME>` (slug), `<ROLE_DESCRIPTION>`, `<MODEL>` (`opus` default; `sonnet` for review/QA/copy roles), `<ROLE_BODY>` (synthesized). Substitute EVERY occurrence incl. the two in the hook command. Check collision: if `.claude/agents/<role>.md` exists, ask overwrite/suffix/skip (default skip). Write to `.claude/agents/<role>.md`.
  5. **Write team.md** — read `team-template.md`, fill: roster rows, `<PRIORITY_ORDER>` = roles in proposal order, routing notes (work-type → role), review pairings (first/primary role reviews the rest; single-role → none).
  6. **Patch CLAUDE.md — after confirmation** — show the proposed `## Team` section (delegation rule + roster table); ask yes/no; on yes, replace existing `## Team` section in nearest `CLAUDE.md` or append, or create `.claude/CLAUDE.md`. Never touch content outside `## Team`.
  7. **Handoff** — print: "Agents written. Run `/clear` to register them, then `/dispatch \"<task>\"`." State the `/clear` step is REQUIRED (mid-session agent files aren't registered until clear).

  Rules block at the end: one question at a time; never write outside the project; memory dirs (`.claude/agent-memory/`) are owned by agents — never overwrite.

- [ ] **Step 2: Verify frontmatter**

Run: `head -5 skills/build-team/SKILL.md | grep -E '^name:|^description:'`
Expected: both `name: build-team` and a non-empty `description:` line.

- [ ] **Step 3: Verify required sections present**

Run:
```bash
for s in "Gather context" "Propose roster" "Interview" "Generate agents" "team.md" "CLAUDE.md" "clear"; do
  grep -qi "$s" skills/build-team/SKILL.md || echo "MISSING:$s"; done; echo done
```
Expected: only `done`.

- [ ] **Step 4: Forbidden-token gate.** Expected: no matches.

- [ ] **Step 5: Commit**

```bash
git add skills/build-team/SKILL.md && git commit -m "feat: build-team skill"
```

---

### Task 4: `dispatch` SKILL.md

**Files:**
- Create: `skills/dispatch/SKILL.md`

**Interfaces:**
- Consumes: `.claude/team.md` (roster, priority order, routing notes, review pairings).
- Produces: `/dispatch <task>` routing + reviewer chaining.

- [ ] **Step 1: Author SKILL.md** with frontmatter `name: dispatch` and a triggering `description:` ("route a task to a team agent / dispatch work to the team"). Body sections:

  1. **Find team.md** — walk up to nearest `.claude/team.md`. If absent: tell user to run `/build-team`, then stop. (No `/staff` reference.)
  2. **Match task to role** — score against routing notes; highest wins. Ties broken by `## Priority order` (earlier = wins). If genuinely tied with no priority signal, ask. If task spans two roles, split into sequential dispatches.
  3. **Dispatch** — state the routing decision in one line, then `Task(subagent_type="<role>", prompt="<task>\n\n---\nContext: see .claude/team.md for your role and pairings.")`. If it fails "agent not found", remind the user to `/clear` first.
  4. **Chain reviewer** — after the task, read `## Review pairings`; if the role has a reviewer, dispatch it with a summary of the implementer's output. No pairing → skip.

  Rules: never do the work inline when a team exists; pass enough context (the subagent has no history).

- [ ] **Step 2: Verify frontmatter + sections**

Run:
```bash
head -5 skills/dispatch/SKILL.md | grep -qE '^name: dispatch' && \
for s in "team.md" "routing" "Priority" "Chain" "clear"; do grep -qi "$s" skills/dispatch/SKILL.md || echo "MISSING:$s"; done; echo done
```
Expected: only `done`.

- [ ] **Step 3: Forbidden-token gate** + confirm no `/staff` coupling:

Run: `grep -in 'staff' skills/dispatch/SKILL.md`
Expected: no output (POC dispatch must not reference `/staff`).

- [ ] **Step 4: Commit**

```bash
git add skills/dispatch/SKILL.md && git commit -m "feat: dispatch skill with reviewer chaining"
```

---

### Task 5: README — the lesson

**Files:**
- Create: `README.md`

**Interfaces:**
- Consumes: all of the above (it explains them).

- [ ] **Step 1: Write README** with these required sections:
  1. **What this is** — per-project agents + memory + routing, learn by reading the files.
  2. **Install** — how to add the plugin to Claude Code.
  3. **Quickstart (happy path)** — `/build-team` → **`/clear` (required, explain why)** → `/dispatch "<task>"`.
  4. **Anatomy of a generated agent** — paste a REAL example generated agent file (run build-team mentally or use the Task 6 output) and annotate, line by line: the inline Stop hook (what `$CLAUDE_PROJECT_DIR`, `test -s`, `exit 2` do; that it's a nudge, not a hard wall — loop protection via `stop_hook_active`), the on-start memory-load, the before-finish memory-write block + the 3-line entry shape.
  5. **Memory** — where it lives (`.claude/agent-memory/<role>/MEMORY.md`), the discipline rules.
  6. **Make your own** — edit a generated agent or hand-write one; run again to add roles.

- [ ] **Step 2: Verify the anatomy section references a real hook**

Run: `grep -q 'CLAUDE_PROJECT_DIR' README.md && grep -qi 'clear' README.md && echo ok`
Expected: `ok`.

- [ ] **Step 3: Forbidden-token gate.** Expected: no matches (README explains the inline hook; must not itself recommend `CLAUDE_PLUGIN_ROOT` etc.).

- [ ] **Step 4: Commit**

```bash
git add README.md && git commit -m "docs: README with annotated generated-agent walkthrough"
```

---

### Task 6: End-to-end integration dry-run

**Files:**
- Create (scratch, gitignored): `scratch/sample-project/` with a fake `docs/brief.md`.

This task proves the loop works on a fresh-feeling project and produces the real agent file the README annotates.

- [ ] **Step 1: Build a sample doc-driven project**

```bash
mkdir -p scratch/sample-project/docs
cat > scratch/sample-project/docs/brief.md <<'EOF'
# Widget API
Early-stage. Build a small REST API for managing widgets. Needs implementation
and code review. No code written yet.
EOF
```

- [ ] **Step 2: Manually simulate `build-team` substitution** to produce one real agent file (the deterministic, testable part — the interview is interactive, but template-fill is not):

Fill `agent-template.md` for role `builder` (model `opus`, body "You implement features for the Widget API."), writing to `scratch/sample-project/.claude/agents/builder.md`. Do the same for `reviewer` (model `sonnet`).

- [ ] **Step 3: Verify generated agent frontmatter parses**

Run:
```bash
cd scratch/sample-project
awk '/^---$/{c++; next} c==1{print}' .claude/agents/builder.md | head -20
grep -c '<ROLE_NAME>\|<ROLE_BODY>\|<MODEL>\|<ROLE_DESCRIPTION>' .claude/agents/builder.md
```
Expected: frontmatter prints; placeholder count is `0` (every placeholder substituted).

- [ ] **Step 4: Verify the inlined hook fires correctly (nudge logic)**

Run:
```bash
cd scratch/sample-project
export CLAUDE_PROJECT_DIR="$PWD"
# Extract the hook command from the agent file and run it with memory absent:
bash -c 'test -s "$CLAUDE_PROJECT_DIR/.claude/agent-memory/builder/MEMORY.md" || { echo "nudge fired" >&2; exit 2; }'; echo "exit:$?"
# Now create memory and confirm it passes:
mkdir -p .claude/agent-memory/builder
echo '2026-06-29: No new learnings this session.' > .claude/agent-memory/builder/MEMORY.md
bash -c 'test -s "$CLAUDE_PROJECT_DIR/.claude/agent-memory/builder/MEMORY.md" || { echo x >&2; exit 2; }'; echo "exit:$?"
```
Expected: first block prints `nudge fired` + `exit:2`; second prints `exit:0`. This proves: missing memory → nudge; the `No new learnings` line satisfies it.

- [ ] **Step 5: Feed the real `builder.md` into README's anatomy section** (replace any placeholder example with this verified file).

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "test: e2e dry-run of template fill + memory hook; wire real example into README"
```

---

## Self-Review (completed during authoring)

- **Spec coverage:** 3-source gather (T3), doc-only scan (T3), standalone agents (T1), inline hook vs `${CLAUDE_PLUGIN_ROOT}` (T1), nudge-not-enforce + first-run safety (T1), strips of persona Step 0 / caveman / bypassPermissions (gate, all tasks), `/clear` gotcha (T3/T4/T5), priority-order tie-break (T2/T3/T4), reviewer chaining (T4), README annotates a real file (T5/T6), CLAUDE.md patch after confirmation (T3). All mapped.
- **Placeholder scan:** template `<UPPERCASE>` placeholders are intentional artifacts, not plan gaps; every other step has concrete content/commands.
- **Type consistency:** placeholder names (`<ROLE_NAME>` etc.) identical across T1/T3 and the hook; `<PRIORITY_ORDER>` consistent T2→T3→T4.
