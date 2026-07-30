# dmons

OpenSpec workflow tooling. The plugin ships four skills:

- **`/dmons:discovery`** — the greenfield front door: gather initial requirements from the Product Owner as the Analyst, with zero tech assumptions, then hand off to `opsx:explore` (see [Discovery](#discovery)).
- **`/dmons:architecture`** — decide the *how* as the Architect: platform, language, frameworks, hosting — the choices discovery deferred — then hand off to `opsx:propose` (see [Architecture](#architecture)).
- **`/dmons:scaffold`** — scaffolds the OpenSpec Apply Workflow into a repo (details below).
- **`/devlog`** — maintains a change's `DEVLOG.md`, the shared channel its agents talk through (see [Devlog](#devlog)).

Together, `discovery` → `architecture` → `scaffold` walk a greenfield project from an empty repo through
requirements (the *what*), technology decisions (the *how*), and the Apply Workflow agents that build it.

## Discovery

The step *before* there is anything to build. On an empty, OpenSpec-initialized repo with **no open or
archived changes**, `/dmons:discovery` casts the main thread as the **Analyst** and the user as the
**Product Owner**, and gathers the project's initial requirements — *what* and *why* — while assuming
**nothing** about the *how*: no tech stack, language, platform, hosting, framework, or library. All of
that stays open until the requirements are set. When the requirements hold together it hands off to
`opsx:explore` to record them, then points the Product Owner at `/opsx:propose` (where the Architect
decides the *how*) and `/dmons:scaffold`. For a repo that already has changes or code, use `opsx:explore`
directly — discovery is greenfield-only.

```
/dmons:discovery
```

## Architecture

The step *after* discovery and the inverse of it. Where the Analyst assumed nothing, the **Architect**
now makes the technology decisions discovery deferred — **platform, language(s), frameworks/libraries,
datastore, hosting, tooling** — each argued against the captured requirements. The Architect is
**opinionated but evidence-backed**: it lays out options and trade-offs, recommends with reasons, pushes
back once if the Product Owner leans the wrong way, and defers to them when they insist (recording their
call and the flagged trade-off either way). Decisions land in the change's `design.md ## Decisions`, with
a standalone **ADR** (`docs/adrs/ADR-*.md`) for the load-bearing, hard-to-reverse calls — extra binding
context `/dmons:scaffold` later feeds the worker/reviewer/supervisor agents. It wraps `opsx:propose`, then
points the Product Owner at `/dmons:scaffold` and `/opsx:apply`.

```
/dmons:architecture
```

## Scaffold

Scaffolds the **OpenSpec Apply Workflow** into a repo. It generates the repo-local files, each tailored
to the project by auditing its OpenSpec specs and changes:

- `CLAUDE.md` — **Analyst/Architect** instructions: a project header plus the authoritative *OpenSpec
  Workflow*. The main thread is cast as the Analyst/Architect (Analyst hat during `opsx:explore`,
  Architect hat during `opsx:propose` and apply), working for you — the **Product Owner** — to realise
  your vision. Apply runs as **two nested loops** (roles, select-the-change, pre-flight, section/block
  implement, stop-and-ask, done).
- `.claude/agents/worker.md` — the **implementer** subagent (Sonnet). Implements one **block** — a
  coherent run of tasks within a section — from the Architect's brief; self-tests; never ticks boxes or
  commits. Multi-stack repos can opt for **one worker per stack** (`worker-<stack>`).
- `.claude/agents/reviewer.md` — the per-block **auditor** subagent (Sonnet), one per change. Reviews
  each block's diff for correctness, decision/ADR compliance, scope, idiom, and domain hazards; reports
  findings, never edits.
- `.claude/agents/supervisor.md` — the per-section **auditor** subagent (Opus), one per change. Runs
  once all a section's blocks have landed and is the only agent that ever sees more than one block:
  cross-block drift, duplicated abstractions, dead scaffolding, eroded decisions, and whether the
  section genuinely satisfies its spec rather than merely ticking its tasks. Reports findings, never
  edits.

### The two loops

```
OUTER — for each ## N. section, in order
  ├─ architect posts the section's base commit to the DEVLOG
  ├─ INNER — for each block in the section
  │    brief worker → worker implements → reviewer audits → loop until Approve
  │    → gates pass → tick boxes → commit
  └─ SECTION REVIEW — supervisor audits base..HEAD
       Approve → next section
       Request changes → remediation block, re-enter INNER (max 2 rounds, then it's your call)
```

A remediation block carries no new task numbers — the section's boxes are already ticked — and lands as
a `fix(...)` commit with the DEVLOG as its record.

The model split is deliberate: worker and reviewer run on **every block**, so they stay on Sonnet; the
supervisor runs **once per section** and carries Opus.

The roles coordinate through the change's shared `DEVLOG.md` — an attributed thread where the Architect
briefs, the worker reports and asks questions, and both review loops play out.

## What it does *not* do

It does not install OpenSpec itself, the `/opsx:*` commands, or the `openspec-*` skills — those are
produced by the `openspec` CLI (`openspec init`). This plugin only adds the orchestration layer that
drives them via the `worker`/`reviewer`/`supervisor` split.

## Usage

In a repo that already has `openspec/` with at least one change:

```
/dmons:scaffold
```

The skill:
1. Verifies the repo is OpenSpec-managed.
2. Audits `openspec/project.md`, change `design.md` `## Decisions`, ADRs, the build system, distinct tech
   stacks, and tooling signals (e.g. `graphify-out/`).
3. Picks the section-container term (*section* vs *group*) to match the repo; the unit of work within a
   section is always a **block**.
4. Confirms any gaps (build commands, project tagline, full-stack vs per-stack workers, overwrites) via a
   short prompt.
5. Fills the templates in [`skills/scaffold/templates/`](skills/scaffold/templates) and writes the files,
   never clobbering an existing `CLAUDE.md` without asking.

## Update scaffold

Migrates a repo that was **already scaffolded with an earlier plugin version** up to the current one.

```
/dmons:update-scaffold
```

Re-running `/dmons:scaffold` on such a repo is the wrong move: the generated files hold audited slot
values (binding decisions, domain hazards, build gates) and any hand edits made since, and regenerating
throws both away. `scaffold` now detects this case and hands off here.

How it works:

1. **Establishes the source version** from the provenance stamp the scaffold leaves in each generated
   file (`<!-- dmons-scaffold: 0.3.0 -->`). Repos scaffolded before 0.3.0 have no stamp, so those are
   feature-detected — a scaffolded repo with no `supervisor.md` is pre-0.3.0.
2. **Harvests** the existing values out of the current files rather than re-auditing the repo. The
   previous audit is crystallised in them; a fresh audit would re-phrase the same decisions and leave
   the agents subtly disagreeing with each other.
3. **Applies each release's migration note** in order, from
   [`skills/update-scaffold/migrations/`](skills/update-scaffold/migrations) — one file per release that
   changed the generated output, naming each discrete change and its exact target content.
4. **Confirms each migration individually** — apply, skip, or show the edit first. Skipping is a
   legitimate outcome; the stamp then records the last fully-applied version rather than overstating.
5. **Leaves in-flight DEVLOGs alone.** A change built under the old workflow keeps its history; the skill
   posts one note saying what changed rather than retro-fitting conventions into an append-only record.

## Design

The templates are annotated skeletons distilled from the hand-tuned agents across `dcli`, `dmon-core`,
`dmon-meko`, and `dmon-websearch`: a shared structure with `{{PLACEHOLDER}}` slots for the parts that
genuinely differ per repo (tech stack, build gates, binding decisions, domain hazards, HITL examples).
The skill fills the slots from the audit and strips the scaffolding.

## Devlog

`tasks.md` is a checklist — *what*, and *whether done*. `DEVLOG.md` is the change's **shared working
channel** — where the Analyst/Architect, the worker(s), the reviewer, and the supervisor talk to each
other as they build: attributed thread posts, in-thread questions (`❓ @architect`), handoffs
(`→ @reviewer`), each section's base commit, and both review loops. It lives next to `tasks.md` at
`openspec/changes/<change-name>/DEVLOG.md`.

```
/devlog
```

The skill locates the active change (via `openspec list --json`, or a name you pass), then appends your
attributed post under the matching `## N. <section>` heading from `tasks.md` — referencing the block it
concerns — and rewrites a pinned `## NEXT` carrying forward the next work and open questions. Posts are
append-only and persist forever (committed with each block, archived with the change); only `## NEXT` is
rewritten. Use it for a brief, a result, a question, or a review verdict.
