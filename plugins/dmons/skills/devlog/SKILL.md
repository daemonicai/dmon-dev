---
name: devlog
description: Maintain a change's DEVLOG.md — the shared channel the Analyst/Architect, worker(s), reviewer, and supervisor talk through as they build an OpenSpec change: attributed thread posts, in-thread questions and answers, the per-block review loop, and the per-section supervisor review, grouped by section with a pinned ## NEXT. Use when the user runs /devlog, when any of the four roles needs to post a note/question/handoff, or after finishing a block or a section during OpenSpec apply.
---

# Devlog

`tasks.md` is a checklist — it records *what* and *whether done*. `DEVLOG.md` is the change's **shared
working channel** — the place the **Analyst/Architect**, the **worker(s)**, the **reviewer**, and the
**supervisor** talk to each other as they build: notes, decisions and their reasons, questions and
answers, and the whole review back-and-forth. Think of it as a chat thread scoped to one change.

It lives in the OpenSpec change directory, next to `tasks.md`:
`openspec/changes/<change-name>/DEVLOG.md`

## Who writes, and when

- **Analyst/Architect** — opens each section with its **base commit**, posts the block brief and
  decisions, answers questions, and rewrites `## NEXT`.
- **worker** (`worker` or a per-stack `worker-<stack>`) — posts what it implemented, questions when
  blocked or unsure, and the `→ @reviewer` handoff.
- **reviewer** — posts its verdict and findings **per block**, and re-audits in-thread until `Approve`.
- **supervisor** — posts its verdict and findings **per section**, once all that section's blocks have
  landed. Cross-block concerns only; the per-block review is the reviewer's.

Everyone **reads the thread first** to pick up context, then appends their own post.

## When to use

- The user runs `/devlog` (optionally with a change name or a note to record).
- Any role needs to post a note, question, answer, or handoff during a block.
- A block's review loop is under way — findings and fixes are posted here.
- A block just landed — capture what the checkbox can't hold before the next one.
- A **section** is opening (record its base commit) or closing (the supervisor's section review, and
  any remediation block that follows from it).

## Locating the change

1. If the user passed a change name, use `openspec/changes/<name>/`.
2. Otherwise run `openspec list --json` to find active changes.
   - Exactly one active change → use it.
   - More than one → ask which change to log against.
   - None → tell the user there's no active change and stop.

## File structure

The body is organised by **section**, mirroring the numbered `## N. <name>` headings in `tasks.md`. The
last section is **always** `## NEXT`. Inside a section, entries are **attributed thread posts**, each
referencing the **block** it concerns. A section opens with its **base commit** and closes with the
**supervisor's review**.

```markdown
# DEVLOG: <change-name>

<!-- One short line: what this change is. -->

## 3. <Section name from tasks.md>

- **[architect]** Base: `a1b2c3d` — submission form, validation, and the submit pipeline.
- **[architect]** Block 3.1–3.3: build the form + validation. Decision: debounce on submit, not keystroke — <why; alternative rejected>.
- **[worker-frontend]** Done 3.1–3.2. ❓ @architect — spec says 300ms, design says 500ms; which?
- **[architect]** @worker-frontend — 500ms, design wins (spec is stale, I'll flag it).
- **[worker-frontend]** 3.3 done, all green. → @reviewer
- **[reviewer]** Request changes: `Form.tsx:42` swallows the error. Otherwise good.
- **[worker-frontend]** Fixed 42. → @reviewer
- **[reviewer]** Approve.
- **[architect]** Block 3.4–3.5: wire submit to the API. <brief>
- **[worker-frontend]** 3.4–3.5 done, green. → @reviewer
- **[reviewer]** Approve.
- **[supervisor]** Section review of `a1b2c3d..HEAD` (3.1–3.5). Request changes: 3.1 defined `SubmitResult` with a `status` string, 3.4 returns a `code` enum and maps it at the call site (`Api.ts:88`) — two shapes for one contract. Also `Form.tsx:30` still holds the stub validator 3.2 superseded. Spec REQ-4 (retry on 503) isn't covered by any block.
- **[architect]** Remediation block from the above → @worker-frontend: unify on the enum, drop the stub, add the retry path + test. No new task numbers.
- **[worker-frontend]** Done — enum unified, stub removed, retry covered by `Api.test.ts:120`. → @reviewer
- **[reviewer]** Approve.
- **[supervisor]** Re-reviewed `a1b2c3d..HEAD`. Approve. Note for NEXT: `Api.ts` is doing both transport and mapping — worth splitting before section 5 builds on it.

## NEXT

- **Up next:** <the next block/section to tackle.>
- **Open questions:** <unresolved questions needing the Product Owner or a decision.>
- **Nits / deferred:** <small issues or cleanups consciously deferred.>
- **Carry-forward:** <anything in-flight the next session must know.>
```

## Conventions

- **Attribute every post.** Prefix with the author's role: `[architect]`, `[worker]` /
  `[worker-<stack>]`, `[reviewer]`, `[supervisor]`.
- **Reference the block.** Say which tasks the post concerns (`3.1–3.3`). A supervisor post references
  the whole section and its commit range.
- **Open every section with its base commit.** The first post under a `## N.` heading is
  `**[architect]** Base: <sha> — <what this section delivers>`. This is the supervisor's review scope
  (`git diff <sha>..HEAD`), not decoration — without it the section review has no boundary.
- **Questions are @-addressed and answered in-thread.** `❓ @architect — …?`; the addressee replies as
  their own post. Handoffs read `→ @reviewer`. A question that outlives the block rolls up into
  `## NEXT` → Open questions.
- **Both review loops live here.** Per block: reviewer posts findings, worker fixes and responds,
  reviewer re-audits, until `Approve`. Per section: supervisor posts findings, the architect carves a
  remediation block that runs the block loop again, supervisor re-audits. Workers hand off to
  `@reviewer` only — the architect invokes the supervisor.
- **Remediation blocks carry no task numbers.** Every box in the section is already ticked, so the
  DEVLOG *is* the record of what the supervisor found and what was done about it. Say what changed.

## Rules

- **`## NEXT` is always the final section.** Every write leaves it at the bottom, fully rewritten to
  reflect the current state. Never leave a stale NEXT.
- **Section headings match `tasks.md`.** Same numbers and names (e.g. `## 4. Message Submission Page`).
  Add a heading on the first post under it, followed immediately by the section's `Base:` post; append
  later posts beneath.
- **Append-only; chatter persists.** Never rewrite or delete prior posts — only `## NEXT` is refreshed.
  The back-and-forth stays in history: it is committed with the block and archived with the change.
- **Record reasons, not just actions.** A decision without its rationale is a wasted post. Capture why a
  path was chosen and what was rejected.
- **Be terse.** Short posts. This is a working channel, not prose — don't restate what `tasks.md`
  already says (the checkbox state); add the detail the checkbox can't hold.
- **Create on first use.** If `DEVLOG.md` doesn't exist, create it with the title, a one-line summary,
  the first section, and a `## NEXT`.

## Procedure

1. Locate the change directory (see above).
2. Read `tasks.md` for the current section names and completion state, and read the existing
   `DEVLOG.md` thread.
3. Determine what to post:
   - From explicit user notes if given.
   - Otherwise from the work/role at hand — a section base, a brief, a result, a question, an answer, a
     block review verdict, or a section review verdict.
4. Edit `DEVLOG.md`:
   - Append your attributed post under the relevant `## N.` section, referencing the block. If the
     heading is new, create it and post the section's `Base: <sha>` first (`git rev-parse --short HEAD`).
   - Rewrite `## NEXT` to reflect the next block/section and any carried-forward items or open
     questions — including architectural notes the supervisor parked rather than blocked on.
5. Confirm to the user in one or two lines what was posted.
