# Per-Day Memory Nudge + Dispatch Namespacing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the generated-agent memory Stop-hook genuinely nudge each day (not just on the first run ever), remove the fabricated loop-safety claim, and namespace the dispatch command.

**Architecture:** Pure-Markdown plugin. The hook check changes from `test -s FILE` (file non-empty) to `grep -q "^$(date +%F):" FILE` (an entry dated today exists). This aligns the check with the already-date-stamped memory entry format. Docs are reworded to match and to drop the unverifiable `stop_hook_active` claim.

**Tech Stack:** Markdown, bash hook commands, grep/date coreutils. No code, no test framework — verification is grep-based.

## Global Constraints

- Plugin must stay portable: use `$CLAUDE_PROJECT_DIR`, never `${CLAUDE_PLUGIN_ROOT}` in generated-agent hooks.
- No machine paths (`/home/`, `/Users/`), no `bypassPermissions`, no author-plugin deps (caveman/ponytail/context-mode tokens) in shipped files (`skills/**`, `.claude-plugin/**`, `README.md`).
- `stop_hook_active` is NOT a documented field — must not appear as a claimed mechanism.
- The hook command stays a single-line bash string valid inside YAML frontmatter (escaped `\"` for inner quotes).

## Canonical new strings (single source of truth — reused across tasks)

**Hook command** (template form, `<ROLE_NAME>` substituted per agent):

```
bash -c 'f="$CLAUDE_PROJECT_DIR/.claude/agent-memory/<ROLE_NAME>/MEMORY.md"; grep -q "^$(date +%F):" "$f" 2>/dev/null || { echo "Record this session before finishing: add a line dated $(date +%F) to .claude/agent-memory/<ROLE_NAME>/MEMORY.md (use the No-new-learnings line if nothing to record)." >&2; exit 2; }'
```

In YAML frontmatter the inner double-quotes are backslash-escaped and the whole value is wrapped in double-quotes, exactly as the current `test -s` command is.

**Body memory paragraph** (template form):

```
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
```

---

### Task 1: Template — hook + body

**Files:**
- Modify: `skills/build-team/agent-template.md`

- [ ] **Step 1:** Replace the `command:` line (line 9) with the canonical hook command (template form, keeping YAML escaping and the `<ROLE_NAME>` placeholder).
- [ ] **Step 2:** Replace the body paragraph under `## Before finishing — write memory` (current lines 22-25, the "nudges you to record... nudge, not a wall" sentence) with the canonical body memory paragraph (template form). Leave the numbered list (1-6) below it unchanged.
- [ ] **Step 3:** Verify:

```bash
grep -n 'date +%F' skills/build-team/agent-template.md          # hook + body reference today
grep -c 'test -s' skills/build-team/agent-template.md           # expect 0
grep -c 'nudge, not a wall\|block up to a few\|lets you through' skills/build-team/agent-template.md  # expect 0
grep -c '<ROLE_NAME>' skills/build-team/agent-template.md       # still present (placeholder intact)
```

Expected: line numbers for `date +%F`; `0`; `0`; non-zero.

- [ ] **Step 4: Commit**

```bash
git add skills/build-team/agent-template.md
git commit -m "fix: per-day memory nudge in agent template; drop false loop claim"
```

---

### Task 2: dispatch SKILL — namespace the command

**Files:**
- Modify: `skills/dispatch/SKILL.md:41`

- [ ] **Step 1:** On line 41, change `re-run \`/dispatch\`` to `re-run \`/agents-poc:dispatch\``. Leave `/clear` bare (built-in).
- [ ] **Step 2:** Verify:

```bash
grep -n '/dispatch' skills/dispatch/SKILL.md   # every hit must be /agents-poc:dispatch
```

Expected: only `/agents-poc:dispatch` occurrences, no bare `/dispatch`.

- [ ] **Step 3: Commit**

```bash
git add skills/dispatch/SKILL.md
git commit -m "fix: namespace dispatch command to avoid global skill collision"
```

---

### Task 3: README — hook example, explanation, body, drop stop_hook_active

**Files:**
- Modify: `README.md` (lines 81, 95-96, 119, 123, 127)

- [ ] **Step 1:** Line 81 (example hook in the builder agent block): replace the `test -s ...` command with the canonical hook command, `<ROLE_NAME>` → `builder`.
- [ ] **Step 2:** Lines 95-96 (body example under `## Before finishing — write memory`): replace with the canonical body memory paragraph, `<ROLE_NAME>` → `builder` (include the ponytail HTML comment).
- [ ] **Step 3:** Line 119 (`### The inline Stop hook` code sample): update the `test -s` command to the canonical hook command shape (`builder`), keeping the `echo \"...\"` elision.
- [ ] **Step 4:** Line 123 (the `**`test -s`**` bullet): rewrite to explain the date check:

```
- **`grep -q "^$(date +%F):"`** — checks whether `MEMORY.md` already has an entry dated today (`date +%F` = `YYYY-MM-DD`, matching the entry format). If today's line is missing, the hook fires. Writing any dated line — including the explicit `No new learnings this session.` line — satisfies it.
```

- [ ] **Step 5:** Line 127 (the `**Nudge, not a wall**` / `stop_hook_active` bullet): replace with:

```
- **Backstop, not a wall** — the check is self-satisfiable: the model writes one line dated today and the next check passes (exit 0), so normal flow ends in one retry. The nudge is **per-day** — it catches the first un-recorded run each day, not every session within a day. That ceiling is deliberate; true per-session enforcement would need a `SubagentStart` snapshot (see the design spec).
```

- [ ] **Step 6:** Verify:

```bash
grep -c 'test -s' README.md                       # expect 0
grep -c 'stop_hook_active' README.md              # expect 0
grep -c 'nudge, not a wall\|block up to a few\|lets you through' README.md  # expect 0
grep -c 'date +%F' README.md                      # expect >=3 (line 81, body, bullet)
```

- [ ] **Step 7: Commit**

```bash
git add README.md
git commit -m "docs: README hook explanation matches per-day check; drop fabricated stop_hook_active"
```

---

### Task 4: Dogfood sample agents — parity

**Files:**
- Modify: `scratch/sample-project/.claude/agents/builder.md`
- Modify: `scratch/sample-project/.claude/agents/reviewer.md`

(`scratch/` is gitignored; these are updated for honest end-to-end evidence, not shipped. No commit needed — they stay out of git.)

- [ ] **Step 1:** In each file, replace the `test -s` hook `command:` with the canonical hook command, `<ROLE_NAME>` → `builder` / `reviewer` respectively.
- [ ] **Step 2:** In each file, replace the "nudges you to record... nudge, not a wall" body paragraph with the canonical body memory paragraph, role substituted.
- [ ] **Step 3:** Verify:

```bash
grep -L 'date +%F' scratch/sample-project/.claude/agents/builder.md scratch/sample-project/.claude/agents/reviewer.md   # expect no output (both contain it)
grep -c 'test -s' scratch/sample-project/.claude/agents/builder.md scratch/sample-project/.claude/agents/reviewer.md     # expect 0 each
```

---

### Task 5: Whole-repo verification

- [ ] **Step 1:** Run the full gate:

```bash
grep -rn 'test -s' skills/ README.md                                   # expect nothing
grep -rn 'stop_hook_active' skills/ README.md docs/                    # expect nothing (docs spec may reference it as "removed"; check context)
grep -rn 'nudge, not a wall\|block up to a few\|lets you through' skills/ README.md  # expect nothing
grep -rn '/dispatch' skills/ README.md | grep -v '/agents-poc:dispatch'  # expect nothing
grep -rn 'CLAUDE_PLUGIN_ROOT\|/home/\|/Users/\|bypassPermissions' skills/ .claude-plugin/ README.md  # expect nothing
grep -rn 'date +%F' skills/build-team/agent-template.md README.md      # present in both
```

- [ ] **Step 2:** Confirm the hook command is still a single valid YAML-frontmatter line in `agent-template.md` (one `command:` key, value wrapped in `"`, inner quotes escaped `\"`).

## Self-Review

- **Spec coverage:** #1 hook fix → Task 1 (template), Task 3 (README example), Task 4 (samples). #1b false-loop-claim → Tasks 1, 3 (body + bullet + stop_hook_active). #2 namespacing → Task 2. All spec "Files changed" rows mapped. ✅
- **Placeholder scan:** Canonical strings are concrete; `<ROLE_NAME>` is an intentional template placeholder, substituted per task. No TBD/TODO. ✅
- **Consistency:** One canonical hook command + one canonical body paragraph reused everywhere; `date +%F` and the YAML-escape shape match across template, README, samples. ✅
