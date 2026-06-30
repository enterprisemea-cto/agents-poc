# Design: per-day memory nudge + dispatch namespacing fix

Date: 2026-06-30
Status: approved

## Problem

Two defects in the shipped plugin.

**#1 — Memory Stop-hook is a no-op after the first run.** Generated agents
carry an inline `Stop` hook whose check is `test -s MEMORY.md` ("file
non-empty?"). Once the agent writes anything once, the file is non-empty
forever, so the nudge never fires again — including sessions where the agent
learned something but skipped recording. The README sells it as "nudges you to
record learnings each session." That is false: it nudges once, on the first
run, then goes silent for the file's lifetime. Intent is agents that actually
record memory over time.

**#1b — Unverifiable loop-safety claim.** The template body claims the hook
"can block up to a few times, then lets you through." No documented mechanism
backs this. `stop_hook_active` is not a documented Stop-hook field (confirmed
against the Claude Code hooks reference, 2026-06-30). The wording must become
true or be removed.

**#2 — Command namespacing drift.** `skills/dispatch/SKILL.md` tells the user
to run bare `/clear` then `/dispatch`. Bare `/dispatch` collides with an
unrelated global `dispatch` skill. Every other doc uses the namespaced
`/agents-poc:dispatch`.

## Mechanism facts (verified against Claude Code hooks docs)

- Agent-file YAML frontmatter `hooks:` blocks ARE honored.
- A `Stop` hook in an agent file auto-converts to `SubagentStop` and fires
  when that subagent finishes.
- Exit 2 blocks the finish and feeds stderr back to the model.
- `$CLAUDE_PROJECT_DIR` is set and points at the project root.
- `stop_hook_active` is NOT a documented field — do not rely on it.

## Decision: #1 — date-stamp check (Option B1)

The memory entry format already date-stamps every entry (`YYYY-MM-DD:`), and
the "nothing learned" path (rule 6) still writes a dated line. So the nudge
checks for a line dated **today**:

```bash
grep -q "^$(date +%F):" "$CLAUDE_PROJECT_DIR/.claude/agent-memory/<ROLE_NAME>/MEMORY.md" \
  || { echo "<nudge text>" >&2; exit 2; }
```

The check is satisfied iff the agent wrote any entry dated today (a real
learning or the explicit "No new learnings this session." line). It aligns
the hook exactly with the documented entry format.

**Why not true per-session (Option B2):** B2 needs a `SubagentStart` snapshot
+ marker file + stale-marker cleanup — real machinery and failure surface for
marginal gain. The accepted ceiling of B1: enforcement is **per-day**, not
per-session. The first forgetful run each day is blocked; a second same-day
run can finish unrecorded (the body instructions still drive recording — the
hook is a backstop, not the only driver). Acceptable for a POC where agents
run roughly once a day. This ceiling is recorded as a `ponytail:`-style note
near the hook so the upgrade path (B2) is visible.

### Self-termination (replaces the false loop claim)

The nudge is self-satisfiable: the model writes one today-dated line and the
next check passes (exit 0). Normal flow terminates in one retry. No fabricated
"blocks a few times" limit is claimed. The body text states plainly: the hook
blocks finishing until a line dated today exists, which the model satisfies by
writing the line.

## Decision: #2 — namespace the command

`skills/dispatch/SKILL.md` → `/dispatch` becomes `/agents-poc:dispatch`
(matching README and build-team). `/clear` is a built-in and stays bare.

## Files changed

| File | Change |
|---|---|
| `skills/build-team/agent-template.md` | hook command → date-stamp check; body: replace false loop claim with self-termination wording; add per-day ceiling note |
| `README.md` | hook explanation → "nudges each day" + accurate self-termination |
| `skills/dispatch/SKILL.md` | bare `/dispatch` → `/agents-poc:dispatch` |
| `scratch/sample-project/.../agents/builder.md` | regenerate hook + body to match template (dogfood parity) |
| `scratch/sample-project/.../agents/reviewer.md` | same |

`scratch/` is gitignored; sample agents are updated for honest end-to-end
evidence but are not part of the shipped diff.

## Out of scope

- Option B2 (true per-session). Documented as the upgrade path only.
- Manifest polish (#3 keywords/homepage/license) — separate, non-behavioral.
- Any new features (routing weights, multi-reviewer, memory search).

## Verification

- `grep -rn 'test -s' skills/` returns nothing.
- `grep -rn 'blocks up to a few times\|lets you through' skills/ README.md`
  returns nothing.
- `grep -rn '/dispatch\b' skills/ README.md` shows only `/agents-poc:dispatch`.
- Template hook contains `date +%F`.
- A sample agent finishing without a today-dated memory line is blocked
  (exit 2); after writing one, it passes (manual reasoning check — the grep
  matches the documented `YYYY-MM-DD:` line shape).
