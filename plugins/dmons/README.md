# dmons

Scaffolds the **OpenSpec Apply Workflow** into a repo. It generates three repo-local files, each tailored
to the project by auditing its OpenSpec specs and changes:

- `CLAUDE.md` — orchestrator instructions: a project header plus the authoritative *OpenSpec Apply
  Workflow* (roles, select-the-change, pre-flight, section-by-section implement, stop-and-ask, done).
- `.claude/agents/worker.md` — the **implementer** subagent (Sonnet). Implements one section/group of a
  change's `tasks.md` from the orchestrator's brief; self-tests; never ticks boxes or commits.
- `.claude/agents/reviewer.md` — the **auditor** subagent (Opus). Reviews the worker's diff for
  correctness, decision/ADR compliance, scope, idiom, and domain hazards; reports findings, never edits.

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
2. Audits `openspec/project.md`, change `design.md` `## Decisions`, ADRs, the build system, and tooling
   signals (e.g. `graphify-out/`).
3. Picks the unit-of-work term (*section* vs *group*) to match the repo.
4. Confirms any gaps (build commands, project tagline, overwrites) via a short prompt.
5. Fills the templates in [`skills/scaffold/templates/`](skills/scaffold/templates) and writes the files,
   never clobbering an existing `CLAUDE.md` without asking.

## Design

The templates are annotated skeletons distilled from the hand-tuned agents across `dcli`, `dmon-core`,
`dmon-meko`, and `dmon-websearch`: a shared structure with `{{PLACEHOLDER}}` slots for the parts that
genuinely differ per repo (tech stack, build gates, binding decisions, domain hazards, HITL examples).
The skill fills the slots from the audit and strips the scaffolding.
