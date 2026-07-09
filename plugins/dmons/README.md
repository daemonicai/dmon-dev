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
context `/dmons:scaffold` later feeds the worker/reviewer agents. It wraps `opsx:propose`, then points the
Product Owner at `/dmons:scaffold` and `/opsx:apply`.

```
/dmons:architecture
```

## Scaffold

Scaffolds the **OpenSpec Apply Workflow** into a repo. It generates the repo-local files, each tailored
to the project by auditing its OpenSpec specs and changes:

- `CLAUDE.md` — **Analyst/Architect** instructions: a project header plus the authoritative *OpenSpec
  Workflow*. The main thread is cast as the Analyst/Architect (Analyst hat during `opsx:explore`,
  Architect hat during `opsx:propose` and apply), working for you — the **Product Owner** — to realise
  your vision. Apply runs **block by block** (roles, select-the-change, pre-flight, block-by-block
  implement, stop-and-ask, done).
- `.claude/agents/worker.md` — the **implementer** subagent (Sonnet). Implements one **block** — a
  coherent run of tasks within a section — from the Architect's brief; self-tests; never ticks boxes or
  commits. Multi-stack repos can opt for **one worker per stack** (`worker-<stack>`).
- `.claude/agents/reviewer.md` — the **auditor** subagent (Opus), one per change. Reviews each block's
  diff for correctness, decision/ADR compliance, scope, idiom, and domain hazards; reports findings,
  never edits.

The roles coordinate through the change's shared `DEVLOG.md` — an attributed thread where the Architect
briefs, the worker reports and asks questions, and the review loop plays out.

## What it does *not* do

It does not install OpenSpec itself, the `/opsx:*` commands, or the `openspec-*` skills — those are
produced by the `openspec` CLI (`openspec init`). This plugin only adds the orchestration layer that
drives them via the `worker`/`reviewer` split.

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

## Design

The templates are annotated skeletons distilled from the hand-tuned agents across `dcli`, `dmon-core`,
`dmon-meko`, and `dmon-websearch`: a shared structure with `{{PLACEHOLDER}}` slots for the parts that
genuinely differ per repo (tech stack, build gates, binding decisions, domain hazards, HITL examples).
The skill fills the slots from the audit and strips the scaffolding.

## Devlog

`tasks.md` is a checklist — *what*, and *whether done*. `DEVLOG.md` is the change's **shared working
channel** — where the Analyst/Architect, the worker(s), and the reviewer talk to each other as they
build: attributed thread posts, in-thread questions (`❓ @architect`), handoffs (`→ @reviewer`), and the
whole review loop. It lives next to `tasks.md` at `openspec/changes/<change-name>/DEVLOG.md`.

```
/devlog
```

The skill locates the active change (via `openspec list --json`, or a name you pass), then appends your
attributed post under the matching `## N. <section>` heading from `tasks.md` — referencing the block it
concerns — and rewrites a pinned `## NEXT` carrying forward the next work and open questions. Posts are
append-only and persist forever (committed with each block, archived with the change); only `## NEXT` is
rewritten. Use it for a brief, a result, a question, or a review verdict.
