---
name: dispatch
description: Route a task to the team / dispatch work to an agent / hand this to the team
---

## 1. Find team.md

Walk up from the current working directory, checking for `.claude/team.md` at each level until reaching the filesystem root. Read the first one found in full — you need all four sections (`## Roster`, `## Priority order`, `## Routing notes`, `## Review pairings`).

If no `.claude/team.md` is found anywhere in the path: tell the user "No team.md found — run `/build-team` first to define your agent team", then STOP. Do not proceed and do not attempt the task inline.

## 2. Match task to role

Read `## Routing notes` and score the incoming task against it. Routing notes map work types (verbs, document topics, file kinds) to roles — count how many signals in the task match each role.

Pick the highest-scoring role. If two or more roles tie, use `## Priority order` to break the tie: the role listed earlier wins.

If the task is genuinely tied with no signal from `## Priority order` — meaning both roles appear at equal priority — ask the user to clarify which role should handle it rather than guessing.

If the task clearly spans two genuinely distinct competencies (for example: implement a feature AND write its documentation, or backend logic AND frontend rendering), split it into sequential dispatches — one per role, in logical order.

## 3. Dispatch

State the routing decision in exactly one line before calling the agent:

> Routing to `<role>` — <reason drawn from Routing notes>.

Then call:

```
Task(subagent_type="<agent-name>", prompt="<the full task description>

---
Context: see .claude/team.md for your role definition, routing notes, and review pairings. Your agent name is <agent-name> and your role is <role>.")
```

Look up `<agent-name>` from the `Agent` column in `## Roster` for the matched role.

The subagent has no conversation history. Include in the prompt: the complete task, any file paths or constraints already known, and any decisions made in this conversation that affect the work.

If the Task call fails with "agent not found": tell the user "This agent was created this session and isn't registered yet — run `/clear` once, then re-run `/dispatch`." Do not retry automatically.

## 4. Chain reviewer

After the dispatched task completes, read `## Review pairings`.

If the role that just ran has a paired reviewer listed there, dispatch that reviewer:

```
Task(subagent_type="<reviewer-agent>", prompt="Review the output of <role> for this task: <one-line task summary>. Files changed / key output: <brief summary of what the implementer produced>.")
```

Look up `<reviewer-agent>` from `## Roster` using the reviewer role named in `## Review pairings`.

If no pairing exists for the dispatched role, skip silently — do not mention it.

## Rules

- When a `.claude/team.md` exists, never do the work inline. Always dispatch.
- Pass enough context in every prompt — the subagent has no conversation history and cannot infer what you know.
- Single dispatch for atomic tasks. Split only when the task genuinely requires two distinct competencies; do not split for style.
- After every implementation dispatch, check for a reviewer pairing and chain it if one exists.
