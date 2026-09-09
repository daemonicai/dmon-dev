# dmons 0.6 — the apply loop, without blocks

**Status:** proposed. Nothing implemented.

**Motivation:** a workshop run in the week to 2026-09-09 showed the loop grinding — most of the
wall-clock going on round trips rather than on work. Two things changed underneath it since the
shape was set: the `worker` and `reviewer` now run **Sonnet 5** (1M context, up from Sonnet 4.5's
200K, and materially more capable), and `block` has always been a dmons invention with no
counterpart in OpenSpec, which structures `tasks.md` as `## N.` sections and nothing finer.

Note what is *not* the argument: context capacity. A section's worth of tasks never came close to
200K either. Blocks bought **review granularity, commit granularity, and blast radius** — a worker
going wrong at task 3 was caught before it reached task 12. The claim in this redesign is that
Sonnet 5 goes wrong rarely enough that the tight leash now costs more than it saves.

## 1. The shape of it

**Every agent keeps its lens and moves up one level.** Nothing is merged, nothing is deleted.

| Agent | 0.5.1 scope | 0.6 scope | Model |
|---|---|---|---|
| `worker` | block | **section** | sonnet (unchanged) |
| `reviewer` | block | **section** | sonnet → **opus** |
| `supervisor` | section | **whole change** | opus (unchanged) |

| | 0.5.1 | 0.6.0 |
|---|---|---|
| Unit of work | block (a run of tasks within a section) | **the section** |
| Nested loops | two | **one**, plus a terminal step |
| Invocations per section | ~4–6 per block × blocks | **2–3 total** |
| Agent files | 3 (+ per-stack workers) | 3 (+ per-stack workers) |

`block` is retired as a term. The section-container term (*section* vs *group*) that scaffold picks
per repo is unaffected.

## 2. The loop

```
OUTER — for each ## N. section of tasks.md, in order
  ├─ architect posts the section's base commit to the DEVLOG
  ├─ brief worker (sonnet) → worker implements the whole section, self-tests
  ├─ reviewer (opus) audits the section diff
  │    Request changes → remediation pass → re-audit (max 2 rounds, then it's the PO's call)
  │    Approve ↓
  └─ gates pass → architect ticks the section's boxes → architect commits

FINAL — once every section has landed
  └─ supervisor (opus) audits the whole change
       Approve → done
       Request changes → remediation pass → re-audit (max 2 rounds, then it's the PO's call)
```

## 3. Why the agents don't merge

The obvious-looking simplification — one auditor invoked at two scopes, `supervisor.md` deleted — was
considered and rejected. Strip the boilerplate both agent files share (frontmatter, authoritative
context, DEVLOG mechanics, boundaries) and what remains in each is **the checklist, which is the bulk
of the file**. The two checklists barely overlap:

- **reviewer** — correctness, binding decisions, OpenSpec scope, language idiom, domain hazards.
- **supervisor** — does the unit satisfy its spec, cross-unit drift, duplicated abstraction, dead
  scaffolding, naming and layering, gate coverage, integrated test coverage, decision erosion.

Both files already spend a section fencing themselves off from each other — the reviewer's *"Stay
diff-local"*, the supervisor's *"You are not the reviewer — do not repeat its work"* and *"if you
find yourself listing style nits, you have the wrong lens"*. Merging them does not remove that
problem, it **internalises** it: one agent holding both checklists, self-policing which lens it wears
from a briefing parameter. The separation stops being structural and becomes a prompt instruction the
agent can drift out of, and the Architect's brief acquires a mode flag it does not currently need.

Keeping both also means the supervisor's checklist is **re-scoped, not rewritten**. Every one of its
checks reads identically one level up: "block" becomes "section", "section" becomes "change".

## 4. What the re-scoping actually buys and costs

**Gains.** Cross-*section* drift has never been checked by anything — the supervisor's remit stopped
at the section boundary. Moving it to the whole change closes that gap. Cross-block drift *within* a
section, which was its old remit, is now covered by construction: the worker writes the whole section
and the reviewer audits the whole section.

**The one genuine regression.** The supervisor used to run N times per change, so it caught drift in
section 2 before section 5 was built on top of it. Running once at the end means drift surfaces later,
when it is more expensive to remediate — every section is already committed and ticked, so remediation
is a whole-change pass rather than a section-local one.

Judged acceptable on the grounds that net coverage still improves. Worth revisiting if long changes
prove painful; the escape hatch, should it be needed, is letting the Architect invoke the supervisor
mid-change at a section boundary. **Not proposed for 0.6** — it reintroduces a scope decision per
invocation, which is the complication §3 exists to avoid.

**Unattended runs sharpen this.** The supervisor's `Request changes` now lands when every section
is already committed and ticked. That is unlike the reviewer's, whose loop runs pre-commit with the
gate still ahead — a remediation pass at change level rewrites work that has already landed. It
should **notify the Product Owner and pause** rather than auto-remediating; see
`dmons-0.6-claude-md-split.md` §8c for the unattended policy this belongs to.

## 5. Section sizing

Sections vary — 4 tasks in one, 20 in another — and 0.6 makes them the same unit of work. The
Architect may **split an oversized section into more than one worker invocation by judgement**,
briefing each on a stated range of tasks. This is deliberately *not* a named concept with its own
template slots and DEVLOG conventions; that is what `block` was, and re-introducing it under another
name would forfeit the simplification. Splitting is an Architect judgement call, recorded as a DEVLOG
post, and it does not change the commit or tick cadence: the section still lands as one commit when
the reviewer approves the whole of it.

## 6. Cadence: gates, ticks, commits

Unchanged in kind, coarser in frequency. Gates run once per section rather than once per block,
against the section's full diff, still quoting `LABEL_EXIT:<n>` from the Makefile targets. The
Architect still owns the three things — the commit, the ticked boxes, the decision to invoke an agent
— and still owns them exclusively.

Consequence to accept: **one commit per section** instead of one per block. Coarser history, and a
bisect lands on a section rather than a block. Judged acceptable because an OpenSpec change is already
a scoped unit and the DEVLOG carries the finer narrative.

A **remediation pass** (renamed from remediation block) carries no new task numbers — a section's
boxes are only ticked after approval — and where it follows a commit it lands as a `fix(...)` commit
with the DEVLOG as its record.

## 7. Cost

Per section, assuming ~3.5 blocks under the old shape:

- **Old:** 3.5 × sonnet reviews + 1 × opus supervisor, plus a spawn-and-brief round trip per block.
- **New:** 1 × opus review, and the supervisor amortised across the whole change rather than per
  section.

Opus 5 is 2.5× Sonnet 5 per token ($5/$25 vs $2/$10), so 3.5 units of review become 2.5 — before
counting the per-block spawn overhead that disappears and the supervisor running once instead of once
per section. The same diff gets read either way; consolidating removes the context each spawn
re-establishes.

Expect 0.6 to be **cheaper as well as faster**, which is unusual enough to be worth measuring on the
first real change rather than assuming.

## 8. Why the reviewer moves to opus

It is now the only audit each section gets, on a diff several times larger than a block's, with the
supervisor no longer backstopping it per section. The template comment currently justifies `sonnet`
by calling the reviewer "the workflow's hot path" running "once per block" — that premise is gone at
one invocation per section.

## 9. What is deliberately unchanged

- **The boundary hooks.** `dmons-guard.sh` is wired through each agent's own frontmatter and is
  indifferent to the unit of work; `dmons-tripwire.sh` brackets `SubagentStart`→`SubagentStop` and is
  likewise unaffected. Both must keep working, and the agents must stay **generated into the repo** —
  plugin subagents silently ignore `hooks:` frontmatter, which would evaporate the enforcement.
- **Architect-only commits, ticks, and invocations.** Fewer, larger units make this *more* important,
  not less: each commit is now the only evidence a whole section was gated.
- **The Makefile command surface** and its `LABEL_EXIT:<n>` gate targets.
- **The DEVLOG as the shared channel**, including `## NEXT` and append-only posts.
- **The auditors' `auditor` guard role**, confining both to writing `DEVLOG.md`.

## 10. Knock-on work

- `CLAUDE.md.template` — two loops become one plus a terminal step (and see the separate question of
  moving procedure into a `/dmons:apply` skill, which would change where this text lives).
- `worker.md.template` — re-scoped block → section.
- `reviewer.md.template` — re-scoped block → section; `model: sonnet` → `opus`; the *"Stay
  diff-local"* fence re-points at the change rather than the section.
- `supervisor.md.template` — re-scoped section → change; the *"You are not the reviewer"* fence
  re-points; `## Your scope` moves from `<section-base>..HEAD` to the change's base commit.
- `scaffold/SKILL.md` — audit prompts, the section-container step, reviewer model assignment.
- `devlog/SKILL.md` and the DEVLOG conventions — posts reference a section, not a block; the
  supervisor's post is no longer filed under a `## N.` heading and needs a home.
- `update-scaffold/migrations/0.6.0.md` — new.
- Both READMEs.

## 11. Migration hazards

- **In-flight changes.** A repo mid-change on 0.5.1 has a DEVLOG written in block conventions and half
  its sections ticked. Consistent with the existing rule, migration should leave in-flight DEVLOGs
  alone and post one note rather than retro-fitting.
- **Harvesting a re-scoped agent.** `update-scaffold` harvests slot values out of existing files
  rather than re-auditing. `{{ARCHITECTURAL_COHERENCE_BULLETS}}` was authored as *"structural risks
  that accumulate across blocks"* and is being re-pointed at a whole change — the harvested text may
  read at the wrong altitude even though the slot transfers cleanly. Same for the reviewer's
  `{{DOMAIN_HAZARDS}}`, authored diff-local.
- **Model assignment drift.** A repo that hand-edited the reviewer onto a different model should not
  be silently moved to opus.
- **No file is deleted**, which keeps 0.6 inside the migration format's existing capabilities.
