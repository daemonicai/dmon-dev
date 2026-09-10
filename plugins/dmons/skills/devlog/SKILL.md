---
name: devlog
description: Maintain a change's DEVLOG.md — the shared channel the Analyst/Architect, worker(s), reviewer, and supervisor talk through as they build an OpenSpec change: attributed thread posts, in-thread questions and answers, the per-section review loop, and the end-of-change supervisor review, grouped by section with a pinned ## NEXT. Use when the user runs /devlog, when any of the four roles needs to post a note/question/handoff, or after finishing a section or the change review during the implementation phase.
---

# Devlog

`tasks.md` is a checklist — it records *what* and *whether done*. `DEVLOG.md` is the change's **shared
working channel** — the place the **Analyst/Architect**, the **worker(s)**, the **reviewer**, and the
**supervisor** talk to each other as they build: notes, decisions and their reasons, questions and
answers, and the whole review back-and-forth. Think of it as a chat thread scoped to one change.

It lives in the OpenSpec change directory, next to `tasks.md`:
`openspec/changes/<change-name>/DEVLOG.md`

## Who writes, and when

- **Analyst/Architect** — opens each section with its **base commit**, posts the section brief and
  decisions, records gate exit lines, answers questions, opens the change review, and rewrites
  `## NEXT`.
- **worker** (`worker` or a per-stack `worker-<stack>`) — posts what it implemented as it works through
  the section, questions when blocked or unsure, and the `→ @reviewer` handoff.
- **reviewer** — posts its verdict and findings **per section**, and re-audits after each remediation
  pass until `Approve`.
- **supervisor** — posts its verdict and findings **once per change**, under `## Change review`, after
  every section has landed. Concerns *between* sections only; each section's own review is the
  reviewer's.

Everyone **reads the thread first** to pick up context, then appends their own post.

## When to use

- The user runs `/devlog` (optionally with a change name or a note to record).
- Any role needs to post a note, question, answer, or handoff during a section.
- A section's review loop is under way — findings and remediation passes are posted here.
- A section just landed — capture what the checkbox can't hold before the next one.
- The **change review** is opening (the architect's range post), under way (the supervisor's verdict),
  or being remediated.
- The run **stopped** and the stop needs recording so a cold session can resume it.

## Locating the change

1. If the user passed a change name, use `openspec/changes/<name>/`.
2. Otherwise run `openspec list --json` to find active changes.
   - Exactly one active change → use it.
   - More than one → ask which change to log against.
   - None → tell the user there's no active change and stop.

## File structure

The body is organised by **section**, mirroring the numbered `## N. <name>` headings in `tasks.md`. Once
every section has landed, a **`## Change review`** heading follows the last of them. The last heading is
**always** `## NEXT`. Inside a section, entries are **attributed thread posts**, referencing the tasks
they concern. A section opens with its **base commit** and closes when it is committed.

```markdown
# DEVLOG: <change-name>

<!-- One short line: what this change is. -->

## 3. <Section name from tasks.md>

- **[architect]** Base: `a1b2c3d` — submission form, validation, and the submit pipeline.
- **[architect]** Brief → @worker-frontend: 3.1–3.5, the form, validation, and submit wiring. Decision: debounce on submit, not keystroke — <why; alternative rejected>. Gates: `make gates-web`.
- **[worker-frontend]** 3.1–3.2 done. ❓ @architect — spec says 300ms, design says 500ms; which?
- **[architect]** @worker-frontend — 500ms, design wins (spec is stale, I'll flag it).
- **[worker-frontend]** 3.3–3.5 done. `WEB_BUILD_EXIT:0 WEB_TEST_EXIT:0`. → @reviewer
- **[reviewer]** Request changes: `Form.tsx:42` swallows the error; 3.1 defines `SubmitResult.status` as a string, 3.4 returns a `code` enum and maps it at `Api.ts:88` — one contract, two shapes. Spec REQ-4 (retry on 503) isn't covered.
- **[architect]** Remediation pass → @worker-frontend: the reviewer's three findings above. No new task numbers.
- **[worker-frontend]** Fixed 42, unified on the enum, retry covered by `Api.test.ts:120`. → @reviewer
- **[reviewer]** Approve.
- **[architect]** Gates: `GATES_WEB_EXIT:0`. Ticked 3.1–3.5, committed `e4f5a6b`.

## 4. <Section name from tasks.md>

- ...

## Change review

- **[architect]** Change review of `9f8e7d6..HEAD` — 4 sections, 4 commits.
- **[supervisor]** Request changes: §2 introduced `Retry` as a policy object, §4 re-implements it inline in `Upload.ts:51` — the same abstraction grown twice. `Form.tsx:30` still holds the stub validator §1 used before §3 replaced it.
- **[architect]** STOPPED — change review requests changes on committed work. Notified the Product Owner: remediate, park, or fix the spec?
- **[architect]** Product Owner: remediate. Remediation pass → @worker-frontend: the supervisor's two findings.
- **[worker-frontend]** Done — `Upload.ts` uses `Retry`, stub removed. → @reviewer
- **[reviewer]** Approve.
- **[architect]** Gates green; committed `fix(...)` `b7c8d9e`.
- **[supervisor]** Re-reviewed `9f8e7d6..HEAD`. Approve. Note for NEXT: `Api.ts` is doing both transport and mapping — worth splitting in the next change.

## NEXT

- **Up next:** <the next section, or the change review, or archiving.>
- **Open questions:** <unresolved questions needing the Product Owner or a decision.>
- **Nits / deferred:** <small issues or cleanups consciously deferred.>
- **Carry-forward:** <anything in-flight the next session must know — including any STOPPED state.>
```

## Conventions

- **Attribute every post.** Prefix with the author's role: `[architect]`, `[worker]` /
  `[worker-<stack>]`, `[reviewer]`, `[supervisor]`.
- **Reference the tasks.** Say which tasks a post concerns (`3.1–3.3`). A supervisor post references
  sections (`§2`, `§4`) and the change's commit range.
- **Open every section with its base commit.** The first post under a `## N.` heading is
  `**[architect]** Base: <sha> — <what this section delivers>`. It is the reviewer's scope for that
  section, and the first section's base is the supervisor's scope for the whole change — without it,
  neither review has a boundary.
- **Questions are @-addressed and answered in-thread.** `❓ @architect — …?`; the addressee replies as
  their own post. Handoffs read `→ @reviewer`. A question that outlives the section rolls up into
  `## NEXT` → Open questions.
- **Both review loops live here.** Per section: the reviewer posts findings, the architect briefs a
  remediation pass, a worker fixes, the reviewer re-audits — at most two passes. Per change: the
  supervisor posts findings under `## Change review`, the architect puts them to the Product Owner, and
  any remediation pass they choose runs the section loop again before the supervisor re-audits. Workers
  hand off to `@reviewer` only — the architect invokes the supervisor.
- **Remediation passes carry no task numbers.** The DEVLOG *is* the record of what was found and what
  was done about it. Say what changed.
- **Record gate exit lines, not verdicts.** `GATES_EXIT:0`, quoted — the supervisor reads these rather
  than re-running anything.
- **Record stops so they can be resumed cold.** `**[architect]** STOPPED at N.M — <what blocked, the
  question>`, with `## NEXT` rewritten to match. The session may die before the answer arrives.

## Rules

- **`## NEXT` is always the final section.** Every write leaves it at the bottom, fully rewritten to
  reflect the current state. Never leave a stale NEXT.
- **`## Change review` sits between the last `## N.` section and `## NEXT`**, and is created only once
  every section has landed.
- **Section headings match `tasks.md`.** Same numbers and names (e.g. `## 4. Message Submission Page`).
  Add a heading on the first post under it, followed immediately by the section's `Base:` post; append
  later posts beneath.
- **Append-only; chatter persists.** Never rewrite or delete prior posts — only `## NEXT` is refreshed.
  The back-and-forth stays in history: it is committed with each section and archived with the change.
- **Record reasons, not just actions.** A decision without its rationale is a wasted post. Capture why a
  path was chosen and what was rejected.
- **Be terse.** Short posts. This is a working channel, not prose — don't restate what `tasks.md`
  already says (the checkbox state); add the detail the checkbox can't hold.
- **Create on first use.** If `DEVLOG.md` doesn't exist, create it with the title, a one-line summary,
  the first section, and a `## NEXT`.
- **A DEVLOG written under an older workflow stays as written.** A change started under 0.5.x has
  per-block posts and per-section supervisor reviews; continue it in the current conventions from the
  next post on, and never retro-fit the old ones.

## Procedure

1. Locate the change directory (see above).
2. Read `tasks.md` for the current section names and completion state, and read the existing
   `DEVLOG.md` thread.
3. Determine what to post:
   - From explicit user notes if given.
   - Otherwise from the work/role at hand — a section base, a brief, a result, a question, an answer, a
     review verdict, gate exit lines, a stop, or the change review.
4. Edit `DEVLOG.md`:
   - Append your attributed post under the relevant `## N.` section (or `## Change review`). If a
     section heading is new, create it and post the section's `Base: <sha>` first
     (`git rev-parse --short HEAD`).
   - Rewrite `## NEXT` to reflect the next section or step and any carried-forward items or open
     questions — including architectural notes the reviewer or supervisor parked rather than blocked on.
5. Confirm to the user in one or two lines what was posted.
