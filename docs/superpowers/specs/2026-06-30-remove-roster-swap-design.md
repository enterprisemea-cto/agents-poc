# Remove `swap` from roster editing

## Goal

The `build-team` roster-edit loop advertises a `swap <old> for <new>` command. Remove it so the
advertised commands are `accept` / `add` / `remove` plus natural-language edits. Restructuring a
roster via natural language ("split backend into API and worker") stays supported — only the
explicit `swap` verb goes away.

## Scope

Two files, deletions only. No new logic, no templates, hooks, or tests touched.

1. **`skills/build-team/SKILL.md`** (step 2, "Propose roster")
   Delete the bullet:
   ```
   - `swap <old> for <new>` — replace, preserving position
   ```
   Keep `accept`, `add <role>`, `remove <role>`, and the natural-language-edits bullet.

2. **`docs/specs/2026-06-29-agents-poc-design.md`** (line ~62)
   The design spec mirrors the command list: `accept` / `add` / `remove` / `swap` / natural-language.
   Drop `swap` from that list so the spec stays truthful.

3. **`docs/superpowers/plans/2026-06-29-agents-poc.md`** (line ~224)
   The plan doc mirrors the same command list: `accept`/`add`/`remove`/`swap`/natural-language.
   Drop `swap` here too.

## Out of scope

- Natural-language edits — kept deliberately.
- `team-template.md`, `team.md`, `dispatch` skill — they reference "roster" only as a section name,
  not the edit verbs. Untouched.

## Verification

After edits, repo-wide `grep -rn "swap" skills/ docs/superpowers/plans/ docs/specs/` returns no
matches outside this spec file. Specifically:
- `grep -n "swap" skills/build-team/SKILL.md` → no matches.
- `grep -n "swap" docs/specs/2026-06-29-agents-poc-design.md` → no matches.
- `grep -n "swap" docs/superpowers/plans/2026-06-29-agents-poc.md` → no matches.
- SKILL.md step 2 still lists `accept` / `add` / `remove` / natural-language.
