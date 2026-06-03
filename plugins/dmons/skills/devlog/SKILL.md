---
name: devlog
description: Maintain a DEVLOG.md narrative alongside an OpenSpec change's tasks.md checklist. Records implementation details, decisions and their reasons, grouped by task group, with a pinned ## NEXT section carrying forward the next work and any open questions. Use when the user runs /devlog, asks to update/write the devlog, or after completing a task group during OpenSpec apply.
---

# Devlog

`tasks.md` is a checklist — it records *what* and *whether done*. `DEVLOG.md` is the companion narrative — it records *how* and *why*: implementation details, choices made and the reasons behind them, things tried and abandoned, and what to do next.

`DEVLOG.md` lives in the OpenSpec change directory, next to `tasks.md`:
`openspec/changes/<change-name>/DEVLOG.md`

## When to use

- The user runs `/devlog` (optionally with a change name or notes to record).
- The user asks to update, write, or check the devlog.
- A task group in `tasks.md` has just been completed during implementation — append an entry before moving to the next group.

## Locating the change

1. If the user passed a change name, use `openspec/changes/<name>/`.
2. Otherwise run `openspec list --json` to find active changes.
   - Exactly one active change → use it.
   - More than one → ask the user which change to log against.
   - None → tell the user there's no active change and stop.

## File structure

The body is organised by **task group**, mirroring the numbered `## N. <name>` headings in `tasks.md`. The last section is **always** `## NEXT`.

```markdown
# DEVLOG: <change-name>

<!-- One short line: what this change is. -->

## 1. <Task group name from tasks.md>

- <What was implemented and any notable detail.>
- **Decision:** <choice made> — <why; alternatives rejected and the reason.>
- <Anything that deviated from design.md or the spec, and why.>

## 2. <Next task group name>

- ...

## NEXT

- **Up next:** <the next task or task group to tackle.>
- **Open questions:** <unresolved questions needing an answer.>
- **Nits / deferred:** <small issues, cleanups, or actions consciously deferred.>
- **Carry-forward:** <anything in-flight that the next session must know.>
```

## Rules

- **`## NEXT` is always the final section.** Every write must leave it at the bottom, fully rewritten to reflect the current state. Never leave a stale NEXT.
- **Group headings match `tasks.md`.** Use the same numbers and names (e.g. `## 4. Message Submission Page`). When logging a group for the first time, add its heading; on later writes, append bullets under the existing heading.
- **Record reasons, not just actions.** A decision without its rationale is a wasted entry. Capture why a path was chosen and what was rejected.
- **Be concise.** Short bullets. This is a working memory aid, not prose. Don't restate what `tasks.md` already says (the checkbox state); add the detail the checkbox can't hold.
- **Preserve history.** Append to existing group sections; don't rewrite or delete prior entries (except `## NEXT`, which is always refreshed).
- **Create on first use.** If `DEVLOG.md` doesn't exist, create it with the title, a one-line summary, the first group section, and a `## NEXT`.

## Procedure

1. Locate the change directory (see above).
2. Read `tasks.md` to know the current group names and completion state, and read the existing `DEVLOG.md` if present.
3. Determine what to record:
   - From explicit user notes if given.
   - Otherwise from the work done in this session since the last entry.
4. Edit `DEVLOG.md`:
   - Append bullets under the relevant `## N.` group heading (creating it if new).
   - Rewrite `## NEXT` to reflect the next task/group and any carried-forward items.
5. Confirm to the user what was logged in one or two lines.
