---
name: scaffold
description: Scaffold the OpenSpec Apply Workflow into the current repo — generate a tailored orchestrator CLAUDE.md plus `worker` and `reviewer` subagents by auditing the repo's openspec specs and changes. Use when the user wants to "set up the openspec agents", "add the worker/reviewer agents", "scaffold the OpenSpec workflow", "bootstrap CLAUDE.md for openspec", or onboard a new repo to the section-by-section apply workflow.
---

# Scaffold the OpenSpec Apply Workflow

Generate three repo-local files, each tailored to the current project by auditing its OpenSpec specs and
changes:

- `CLAUDE.md` — the orchestrator instructions (project header + the authoritative OpenSpec Apply Workflow).
- `.claude/agents/worker.md` — the implementer subagent.
- `.claude/agents/reviewer.md` — the auditor subagent.

The skill does **not** install OpenSpec, the `/opsx:*` commands, or the `openspec-*` skills — those come
from the `openspec` CLI (`openspec init`). It only produces the orchestration layer that wraps them.

The templates live at `${CLAUDE_PLUGIN_ROOT}/skills/scaffold/templates/` — `CLAUDE.md.template`,
`worker.md.template`, `reviewer.md.template`. They are annotated skeletons with `{{PLACEHOLDER}}` slots
and `<!-- guidance -->` comments. Your job is to fill every slot from the audit and **delete every
guidance comment** before writing the final files.

---

## Step 1 — Verify preconditions

Run a quick check (use context-mode or plain `ls`/`test`, keep output small):

1. `openspec/` exists in the repo root. If not: stop and tell the user this repo isn't OpenSpec-managed
   yet — they should run `openspec init` first.
2. There is **at least one change** under `openspec/changes/` (excluding `archive/`). If there are none,
   warn that the generated agents will be thin on binding decisions, and ask whether to proceed using
   only archived changes / `openspec/project.md`, or stop.
3. Note whether `openspec/specs/` has committed capability specs.

## Step 2 — Audit the repo

Gather the facts that fill the templates. Prefer `openspec` commands and targeted reads; keep large
output in context-mode.

**Project identity & decisions**
- `openspec/project.md` (if present) — the one-paragraph "what this project is", tech stack, conventions.
- The active change(s): read each `openspec/changes/<slug>/proposal.md` and **`design.md` `## Decisions`**.
  These are the binding constraints the agents must enforce.
- Archived changes under `openspec/changes/archive/` — skim recent ones for additional load-bearing
  decisions and the domain's real hazards.
- ADRs: does `docs/adrs/ADR-*.md` (or similar) exist? Is there a `coding-agent-brief.md` or equivalent
  vision doc? If so, those are *also* binding context; if not, the binding decisions live only in each
  change's `design.md`.

**Build system & gates** — detect, don't assume:
- `Makefile` → `make build` / `make test` (read the targets to confirm).
- `*.slnx` / `*.csproj` / `*.sln` → `dotnet build` / `dotnet test`; check for a `dotnet format` /
  analyzers / `TreatWarningsAsErrors` policy (a format gate).
- `package.json` → the project's `build` / `test` / `lint` scripts.
- `Cargo.toml` → `cargo build` / `cargo test` / `cargo clippy` / `cargo fmt --check`.
- `pyproject.toml` → the test runner + linter/formatter (pytest, ruff, etc.).
- Anything else: read the CI config or existing docs to find the real commands.

**Tooling signals**
- `graphify-out/graph.json` present? → include the graphify tool lines in the agents.
- Confirm context-mode is in use (this plugin assumes it; keep the context-mode tool guidance).

**Language & domain**
- Primary language (for the idiom/style bullets and the `{{LANG}}` slot).
- Project type → the reviewer's **domain hazards** section. A library, a CLI, a service, an agent, and a
  web component each break in different ways. Name the hazards that are actually true here, mined from
  the design decisions and specs — don't paste a generic checklist.

## Step 3 — Decide the unit-of-work term

OpenSpec `tasks.md` groups tasks under `## N.` headings. Repos call this unit either a **"section"** or a
**"group"**. Look at how the existing `tasks.md` / `design.md` refer to it and match that. Default to
**"section"** if there's no signal. This term fills every `{{UNIT}}` slot consistently across all three
files.

## Step 4 — Confirm gaps with the user

You will rarely have every fact. Use the **AskUserQuestion tool** to confirm only what you genuinely
can't infer — typically:
- the one-line project description / tagline (if `project.md` is absent or vague);
- the exact build / test / format commands (offer your detected guess as the recommended option);
- whether to overwrite any of the three target files that already exist (see Step 6).

Prefer making reasonable, clearly-stated decisions over interrogating the user. Don't ask about things
you can read.

## Step 5 — Fill the templates

For each template:
1. Read `${CLAUDE_PLUGIN_ROOT}/skills/scaffold/templates/<file>.template`.
2. Replace every `{{PLACEHOLDER}}` with the audited content. Keep the prose voice of the template —
   blunt, specific, imperative.
3. **Delete every `<!-- ... -->` comment**, including the header block. The final files must contain no
   template scaffolding.
4. For optional lines (format gate, graphify tools, ADR clauses): include them only when the audit says
   they apply; otherwise remove the line cleanly (no dangling placeholder, no empty bullet).

Key slots and where they come from:

| Slot | Source |
|------|--------|
| `{{PROJECT_NAME}}`, `{{PROJECT_DESCRIPTION}}`, `{{PROJECT_DESCRIPTION_SHORT}}` | `project.md`, specs |
| `{{TECH_STACK}}`, `{{ENGINEER_STRENGTHS}}`, `{{LANG}}`, `{{LANG_IDIOMS}}`, `{{STYLE_BULLETS}}` | detected language + conventions |
| `{{UNIT}}` | Step 3 (section/group) |
| `{{BUILD_CMD}}`, `{{TEST_CMD}}`, `{{GATE_LIST}}`, format lines | detected build system |
| `{{BINDING_DECISIONS}}`, `{{DECISIONS_HEADING}}`, `{{DECISIONS_NOUN}}` | change `design.md` `## Decisions` + ADRs |
| `{{ADR_CONTEXT_LINES}}`, `{{ADR_STOP_CLAUSE}}`, `{{COMPLIANCE_NOUN}}` | whether ADRs exist (ADRs → "ADR"; else "design-decision") |
| `{{DOMAIN_HAZARDS}}`, `{{DOMAIN_HAZARDS_HEADING}}`, `{{DOMAIN_QUALITY}}` | project type + specs |
| `{{HITL_EXAMPLES}}` | the change's human-verification tasks (real-terminal behaviour, interactive prompts, etc.) |
| `{{WORKER_DESCRIPTION}}`, `{{REVIEWER_DESCRIPTION}}` | compose: role + project + tech + task areas + handoff |
| `{{GRAPHIFY_TOOL_LINE}}` | present only if `graphify-out/` exists |
| `{{COAUTHOR_LINE}}` | the repo's existing commit convention, e.g. `Co-Authored-By: Claude <noreply@anthropic.com>` (check `git log`) |

When ADRs do **not** exist, set `{{COMPLIANCE_NOUN}}` → "design-decision", `{{DECISIONS_NOUN}}` →
"binding design decisions", `{{DECISIONS_NOUN_SINGULAR}}` → "binding design decision", and make
`{{ADR_CONTEXT_LINES}}`/`{{ADR_STOP_CLAUSE}}` state that decisions live in the change's `design.md`.
When ADRs exist, use "ADR" wording and list the ADR files as binding context.

## Step 6 — Write the files (don't clobber blindly)

For each of `CLAUDE.md`, `.claude/agents/worker.md`, `.claude/agents/reviewer.md`:
- If it does **not** exist → write it (create `.claude/agents/` if needed).
- If it **does** exist → do **not** overwrite silently. Show the user a short diff/summary of what would
  change and ask (AskUserQuestion) whether to overwrite, merge, or skip. A pre-existing `CLAUDE.md` often
  has project content worth preserving — prefer merging the OpenSpec Apply Workflow section in over
  replacing the whole file.

## Step 7 — Report

Summarise: which files were written/skipped, the unit term chosen, the build/test gates wired in, and the
binding decisions the agents now enforce. Then tell the user the next step: run `/opsx:apply` (or
`/opsx:propose` to create a change first), and the orchestrator will delegate to the new `worker` /
`reviewer` agents.

---

## Guardrails
- Never invent binding decisions or ADRs. If the repo has none, say the agents derive their constraints
  from each change's `design.md`, and leave the decisions list lean.
- Detect build commands; never hard-code `dotnet`/`make` without confirming the repo actually uses it.
- Always strip template comments and unused optional lines from the generated files.
- The generated CLAUDE.md's "OpenSpec Apply Workflow" section is authoritative and must keep its
  structure — tailor the commands and nouns, not the workflow shape.
