---
name: build-team
description: Run this skill to build agents / set up a team / onboard agents / create custom agents for this project — interviews the user, generates per-role agent files, writes team.md, and patches CLAUDE.md.
---

## 1. Gather context

Pull from all three sources before proposing anything.

**a) Conversation history** — read the current session turns. They are authoritative for intent: the user may have already described the project, named roles, or stated constraints. Extract everything relevant before asking questions.

**b) Doc scan** — read up to 8 files, up to ~200 lines each, Markdown only (never sample source code):
- `README.md`, `README*` at the project root
- `docs/**/*.md` (up to 5 files, most recently modified first)
- `CLAUDE.md` — explicit priority (richest context source)
- Root-level `*.md` files (CHANGELOG, CONTRIBUTING, etc.)
- `.claude/CLAUDE.md` (fallback if no project root CLAUDE.md exists)

From each file extract: project purpose, domain, tech stack mentions, team/role language, any existing agent or automation references.

**c) One explicit ask** — after reading, ask exactly once:

> "Anywhere else I should look? Paths, docs, folders, URLs — or 'no'."

Read everything the user points to. If they answer "no", proceed immediately.

---

## 2. Propose roster

Synthesize the three context sources into 2–4 roles. Aim for at least 2 roles so the dispatch skill has meaningful routing and can chain reviews.

For each proposed role provide:
- **Agent name** (kebab slug, e.g. `backend-engineer`)
- **One-line why** — what work this role owns and why it is distinct from the others

Present the full roster. Accept and act on these edits (loop until the user types `accept` or equivalent):
- `accept` — proceed to step 3
- `add <role>` — append a new role with a one-line rationale you draft
- `remove <role>` — drop it and re-number
- `swap <old> for <new>` — replace, preserving position
- Natural-language edits (e.g. "split backend into API and worker") — interpret and show the updated roster

Do not proceed past this step until the user explicitly accepts.

---

## 3. Interview gaps only

Ask 0–4 targeted follow-up questions to fill genuinely missing information — things that would make the role definitions vague or incorrect. Examples: project stage (greenfield vs. maintenance), hard boundaries between roles, must-have tool restrictions, output format expectations.

Rules for this step:
- One question at a time. Wait for the answer before asking the next.
- Only ask if the answer is not already in the gathered context.
- If the user says "skip" or gives a short answer, accept it and proceed.
- Stop after 4 questions regardless of what remains unclear — document the assumption in the relevant agent's role body.

---

## 4. Generate agents

For each accepted role, in order:

**Read** `skills/build-team/agent-template.md` (the sibling file in this skill directory). It contains the full template with frontmatter, memory hooks, and memory write instructions. Do not modify its structure — only substitute placeholders.

**Substitute every occurrence** of these placeholders throughout the file:

| Placeholder | Value |
|---|---|
| `<ROLE_NAME>` | Kebab slug (e.g. `backend-engineer`). Appears in: frontmatter `name:`, Stop-hook command (twice — directory path and echo message), memory load section (once), memory write section (twice). Replace ALL occurrences. |
| `<ROLE_DESCRIPTION>` | A single sentence describing what this agent does in the context of this specific project. |
| `<MODEL>` | `opus` for implementation, architecture, and strategy roles. `sonnet` for review, QA, copy, and triage roles. |
| `<ROLE_BODY>` | 3–8 bullet points synthesized from the gathered context and interview answers: primary responsibilities, key constraints, tool or file scope, output format, anything domain-specific to this project. Write in imperative voice ("Implement", "Review", "Draft"). |

**Collision check** — before writing, check whether `.claude/agents/<role>.md` already exists. If it does:
- Ask: "`.claude/agents/<role>.md` exists. Overwrite / version-suffix (saves as `<role>-v2.md`) / skip? Default: skip."
- Apply the user's answer. Default to skip if they do not respond.

**Write** the filled file to `.claude/agents/<role>.md`.

Repeat for every role in the accepted roster before moving to step 5.

---

## 5. Write team.md

Read `skills/build-team/team-template.md` (the sibling file in this skill directory). Fill every placeholder:

| Placeholder | Value |
|---|---|
| `<PROJECT_NAME>` | Project name from the gathered context, or the repository/directory name if not stated. |
| `<BUILD_DATE>` | Today's date in `YYYY-MM-DD` format. |
| `<SOURCE_LINE>` | Concise description of what was used, e.g. `"conversation + README.md + docs/architecture.md"`. |
| `<ROSTER_ROWS>` | One Markdown table row per role: `\| Role display name \| agent-name \| One sentence on when to use this agent \|`. |
| `<PRIORITY_ORDER>` | Numbered list of agent names in the order the roles were accepted (first = highest priority). The `/dispatch` skill uses this list to break ties when a task matches multiple roles. |
| `<ROUTING_NOTES>` | Bullet list mapping work types to roles. Use concrete language: verbs (implement, review, write, triage), doc or file types (API spec, test file, changelog), and domain topics (authentication, billing, UI). Example: `- API endpoint changes → backend-engineer`. |
| `<REVIEW_PAIRINGS>` | For each implementer role, name its reviewer. Convention: the first/primary role listed reviews the work of the others. If only one role exists in the project, write: `No pairings (single role)`. |

Write the filled file to `.claude/team.md`.

---

## 6. Patch CLAUDE.md — after confirmation

Compose a `## Team` section with this structure:

```
## Team

Delegate work to agents. Use `/dispatch "<task>"` — it reads `.claude/team.md` and routes to the right agent.

| Role | Agent | When to use |
|------|-------|-------------|
<one row per role, same content as team.md roster>
```

Show this proposed section to the user and ask: "Add this to CLAUDE.md? (yes/no)"

On **yes**:
1. Walk up from the project root to find the nearest `CLAUDE.md` (for mono-repo / nested-project cases, check parent directories). Check `.claude/CLAUDE.md` and then the project root `CLAUDE.md`.
2. If a `## Team` section already exists (heading through the next `##` or end of file), replace exactly that section with the new one.
3. If no `## Team` section exists, append the new section at the end of the file.
4. If no `CLAUDE.md` exists anywhere, create `.claude/CLAUDE.md` containing only the `## Team` section.
5. Never modify any content outside the `## Team` section.

On **no**: skip silently and proceed to handoff.

---

## 7. Handoff

Print a summary:

- List every agent file written (`.claude/agents/<role>.md`) and confirm `team.md` location.
- State any roles that were skipped due to collision (if any).
- Print this instruction verbatim:

> **Run `/clear` to register the new agents, THEN `/dispatch "<task>"` to use them.**
> The `/clear` step is REQUIRED — agent files written during a session are not in the registry until you clear and restart.

---

## Rules

- One question at a time. Never ask two questions in the same message.
- Never write files outside the current project directory.
- Never overwrite `.claude/agent-memory/` directories or any file inside them — those are owned by the agents that created them.
- Never reference or write to `~/.claude/agents`, `.claude/personas/`, or any global agent path.
- Never emit machine-specific paths or environment variable expansions in any generated file.
- Markdown-only doc scan: read `.md` files for context, never sample source code files to infer role definitions.
