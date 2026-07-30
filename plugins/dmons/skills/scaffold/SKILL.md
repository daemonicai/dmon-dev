---
name: scaffold
description: Scaffold the OpenSpec Apply Workflow into the current repo — generate a tailored Analyst/Architect CLAUDE.md plus one or more `worker` subagents, a `reviewer` subagent, and a `supervisor` subagent by auditing the repo's openspec specs and changes. Use when the user wants to "set up the openspec agents", "add the worker/reviewer/supervisor agents", "scaffold the OpenSpec workflow", "bootstrap CLAUDE.md for openspec", or onboard a new repo to the block-by-block apply workflow.
---

# Scaffold the OpenSpec Apply Workflow

Generate the repo-local files, each tailored to the current project by auditing its OpenSpec specs and
changes:

- `CLAUDE.md` — the **Analyst/Architect** instructions: a project header plus the authoritative OpenSpec
  Workflow. The main thread is cast as the **Analyst/Architect** (Analyst hat during `opsx:explore`,
  Architect hat during `opsx:propose` and apply) working for the user, who is the **Product Owner**.
  Apply proceeds through **two nested loops** — the outer over `## N.` sections, the inner **block by
  block** (a block = a coherent run of tasks within one section), with a section-wide audit closing
  each outer iteration.
- `.claude/agents/worker.md` — the implementer subagent (**sonnet**). **One** worker for a single-stack
  (or full-stack) project, or **one worker per tech stack** (`worker-<stack>.md`) if the Product Owner
  chooses per-stack workers (Step 4).
- `.claude/agents/reviewer.md` — the per-block auditor subagent (**sonnet**). Always exactly **one**
  reviewer, even with per-stack workers. Diff-local: it reviews one block at a time.
- `.claude/agents/supervisor.md` — the per-section auditor subagent (**opus**). Always exactly **one**,
  even with per-stack workers. Runs once all a section's blocks have landed and is the only agent that
  ever sees more than one block: cross-block drift, duplicated abstractions, dead scaffolding, and
  whether the section genuinely satisfies its spec rather than merely ticking its tasks.

**The model split is deliberate.** The worker and reviewer run on every block, so they are the hot path
and stay on sonnet; the supervisor runs once per section and carries opus. Preserve this when
generating — a supervisor on a weaker model removes the workflow's only cross-block check.

The skill does **not** install OpenSpec, the `/opsx:*` commands, or the `openspec-*` skills — those come
from the `openspec` CLI (`openspec init`). It only produces the orchestration layer that wraps them.

Scaffold is the **third** step of the greenfield chain: **`/dmons:discovery`** (the Analyst gathers
requirements — the *what*) → **`/dmons:architecture`** (the Architect decides the technology — the
*how*, recorded in each change's `design.md ## Decisions` and any ADRs) → **`/dmons:scaffold`** (this
skill, which audits those decisions to generate the Apply Workflow agents). By the time scaffold runs the
project should already have at least one change carrying the decisions the agents will enforce; if it
doesn't, that's the signal to run discovery and architecture first (see Step 1).

The templates live at `${CLAUDE_PLUGIN_ROOT}/skills/scaffold/templates/` — `CLAUDE.md.template`,
`worker.md.template`, `reviewer.md.template`, `supervisor.md.template`. They are annotated skeletons
with `{{PLACEHOLDER}}` slots
and `<!-- guidance -->` comments. Your job is to fill every slot from the audit and **delete every
guidance comment** before writing the final files.

---

## Step 1 — Verify preconditions

Run a quick check (use context-mode or plain `ls`/`test`, keep output small):

0. **Is this repo already scaffolded?** Look for a heading whose text is `OpenSpec Workflow` in
   `CLAUDE.md` — **at any level** (`grep -nE '^#{1,4} +OpenSpec Workflow'`; generated files usually
   demote it to `##` under the project's own H1) — plus `.claude/agents/reviewer.md`.
   **If both exist, this is an update, not a scaffold** — the repo has
   audited slot values and possibly hand edits that regenerating would destroy. Tell the user and hand
   off to **`/dmons:update-scaffold`**, which migrates it onto the current version one confirmable
   change at a time. Only continue here if they explicitly want to regenerate from scratch.
1. `openspec/` exists in the repo root. If not: stop and tell the user this repo isn't OpenSpec-managed
   yet — they should run `openspec init` first.
2. There is **at least one change** under `openspec/changes/` (excluding `archive/`). If there are none:
   - If the repo is **greenfield** (empty, no changes at all), the decisions the agents enforce don't
     exist yet — point the Product Owner at **`/dmons:discovery`** to gather requirements, then
     **`/dmons:architecture`** to decide the technology, which produces the first change. Scaffolding
     before that yields agents with nothing binding to enforce.
   - Otherwise (e.g. only archived changes or a `project.md` to lean on), warn that the generated agents
     will be thin on binding decisions, and ask whether to proceed using only archived changes /
     `openspec/project.md`, or stop.
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

**Tech stacks (for the worker roster)**
- Enumerate the **distinct** tech stacks in the repo — a stack is a body of code with its own toolchain,
  idioms, and build/test commands. Examples: a Vite/TypeScript frontend + a `.NET`/`*.csproj` backend;
  a Unity/C# game + Objective-C (iOS) + Kotlin/Java (Android) native components; a Rust core + a Python
  binding.
- **One stack → one full-stack worker** (no question needed). **Two or more → ask in Step 4** whether
  the Product Owner wants a single full-stack worker or a dedicated worker per stack. Record each
  stack's own build/test command — a per-stack worker's gates and idioms come from its stack, not the
  repo average.

**Tooling signals**
- `graphify-out/graph.json` present? → include the graphify tool lines in the agents.
- Confirm context-mode is in use (this plugin assumes it; keep the context-mode tool guidance).

**Language & domain**
- Primary language (for the idiom/style bullets and the `{{LANG}}` slot).
- Project type → the reviewer's **domain hazards** section. A library, a CLI, a service, an agent, and a
  web component each break in different ways. Name the hazards that are actually true here, mined from
  the design decisions and specs — don't paste a generic checklist.
- Project type → the supervisor's **architectural coherence** list, which is a *different* cut of the
  same project. Ask "what erodes here across several blocks that no single block's diff would show?" —
  public API surface consistency for a library; contract/versioning stability and unevenly-applied
  cross-cutting concerns for a service; tool-surface and permission-boundary coherence for an agent;
  the stack seam (does the frontend's assumed contract match the backend block that shipped?) for a
  multi-stack repo. **These bullets must not duplicate the reviewer's** — if a bullet is checkable from
  one diff, it belongs to the reviewer.

## Step 3 — The two-level unit model (section → block)

The workflow has **two** levels; the templates already bake this in, but you must fill the section term
correctly:

- **Section = the `## N.` container.** OpenSpec `tasks.md` groups tasks under `## N.` headings. Repos
  call this container either a **"section"** or a **"group"** — look at how the existing `tasks.md` /
  `design.md` refer to it and match that; default to **"section"** with no signal. This term fills
  every `{{UNIT}}` slot consistently across all files. The Architect walks sections **in order**.
- **Block = the unit of work.** A **block** is a coherent run of tasks *within one section* (e.g.
  `N.1–N.3`) that the Architect judges to be one sensible deliverable for a worker to build, the
  reviewer to review, and one commit to land. **"block" is a fixed term** — it is not templated and
  does not vary per repo; leave every literal "block" in the templates as-is. A section is one or more
  blocks; a block never spans sections; with per-stack workers a block is single-stack.

So: `{{UNIT}}` = the section container's name (section/group); "block" stays "block" everywhere.

**The two levels map onto the two auditors**, which is why getting `{{UNIT}}` right matters: the
`reviewer` audits a **block**, the `supervisor` audits a **{{UNIT}}**. Every file uses the `{{UNIT}}`
slot for the container — keep the term identical across `CLAUDE.md`, the worker(s), the reviewer, and
the supervisor, or the generated agents will disagree about what they are reviewing.

## Step 4 — Confirm gaps with the user

You will rarely have every fact. Use the **AskUserQuestion tool** to confirm only what you genuinely
can't infer — typically:
- the one-line project description / tagline (if `project.md` is absent or vague);
- the exact build / test / format commands (offer your detected guess as the recommended option);
- **worker roster (only if Step 2 found more than one stack):** ask whether they want a **single
  full-stack worker** or **a dedicated worker per stack**. If per-stack, propose the worker names
  (`worker-<stack>`, e.g. `worker-frontend` / `worker-dotnet`) for confirmation. The reviewer and the
  supervisor are always a single agent each, regardless of the answer.
- whether to overwrite any target file that already exists (see Step 6).

Prefer making reasonable, clearly-stated decisions over interrogating the user. Don't ask about things
you can read.

## Step 5 — Fill the templates

For each template:
1. Read `${CLAUDE_PLUGIN_ROOT}/skills/scaffold/templates/<file>.template`.
2. Replace every `{{PLACEHOLDER}}` with the audited content. Keep the prose voice of the template —
   blunt, specific, imperative.
3. **Delete every `<!-- ... -->` comment**, including the header block. The final files must contain no
   template scaffolding. (Step 6 then adds the one comment that belongs in the output — the provenance
   stamp.)
4. For optional lines (format gate, graphify tools, ADR clauses): include them only when the audit says
   they apply; otherwise remove the line cleanly (no dangling placeholder, no empty bullet).

Key slots and where they come from:

| Slot | Source |
|------|--------|
| `{{PROJECT_NAME}}`, `{{PROJECT_DESCRIPTION}}`, `{{PROJECT_DESCRIPTION_SHORT}}` | `project.md`, specs |
| `{{TECH_STACK}}`, `{{ENGINEER_STRENGTHS}}`, `{{LANG}}`, `{{LANG_IDIOMS}}`, `{{STYLE_BULLETS}}` | detected language + conventions |
| `{{UNIT}}` | Step 3 — the section container's name (section/group) |
| `{{WORKER_NAME}}`, `{{STACK}}`, `{{WORKER_STACK_LINE}}`, `{{WORKER_ROLE_LINES}}`, `{{BLOCK_STACK_RULE}}` | Step 4 worker roster (see the multi-stack note below the table) |
| `{{BUILD_CMD}}`, `{{TEST_CMD}}`, `{{GATE_LIST}}`, format lines | detected build system (per-stack worker: **its** stack's commands) |
| `{{BINDING_DECISIONS}}`, `{{DECISIONS_HEADING}}`, `{{DECISIONS_NOUN}}` | change `design.md` `## Decisions` + ADRs |
| `{{ADR_CONTEXT_LINES}}`, `{{ADR_STOP_CLAUSE}}`, `{{COMPLIANCE_NOUN}}` | whether ADRs exist (ADRs → "ADR"; else "design-decision") |
| `{{DOMAIN_HAZARDS}}`, `{{DOMAIN_HAZARDS_HEADING}}`, `{{DOMAIN_QUALITY}}` | project type + specs (multi-stack: each stack's hazards under its own sub-bullet) |
| `{{HITL_EXAMPLES}}` | the change's human-verification tasks (real-terminal behaviour, interactive prompts, etc.) |
| `{{WORKER_DESCRIPTION}}`, `{{REVIEWER_DESCRIPTION}}` | compose: role + project + tech (+ the stack a per-stack worker owns) + task areas + handoff |
| `{{SUPERVISOR_DESCRIPTION}}` | compose: {{UNIT}}-level auditor + project + what it catches that block review can't + when it runs (after a {{UNIT}}'s last block lands) |
| `{{SUPERVISOR_TITLE}}` | a rung above `{{PRINCIPAL_TITLE}}` — e.g. "Staff Engineer" / "Principal Architect" to the reviewer's "Principal Engineer" |
| `{{ARCHITECTURAL_COHERENCE_BULLETS}}` | Step 2 — the cross-block structural hazards; **must not restate the reviewer's `{{DOMAIN_HAZARDS}}`** |
| `{{GRAPHIFY_TOOL_LINE}}` | present only if `graphify-out/` exists |
| `{{COAUTHOR_LINE}}` | the repo's existing commit convention, e.g. `Co-Authored-By: Claude <noreply@anthropic.com>` (check `git log`) |

**Worker roster (single vs per-stack).** The worker slots depend on the Step 4 answer:

- **Single full-stack worker:** fill `worker.md.template` once with `{{WORKER_NAME}}` = `worker`;
  **delete** the `{{WORKER_STACK_LINE}}` line. In `CLAUDE.md`, `{{WORKER_ROLE_LINES}}` is one bullet
  (`**` + `` `worker` `` + `** agent — implements each block.`) and `{{BLOCK_STACK_RULE}}` is **deleted**.
- **Per-stack workers:** fill `worker.md.template` **once per stack**, writing a separate
  `.claude/agents/worker-<stack>.md`, with `{{WORKER_NAME}}` = `worker-<stack>`, `{{STACK}}` = that
  stack, its own `{{BUILD_CMD}}`/`{{TEST_CMD}}`/idioms/hazards, and the `{{WORKER_STACK_LINE}}` kept. In
  `CLAUDE.md`, `{{WORKER_ROLE_LINES}}` lists one bullet per worker and `{{BLOCK_STACK_RULE}}` is kept
  (the single-stack-block routing rule).

When ADRs do **not** exist, set `{{COMPLIANCE_NOUN}}` → "design-decision", `{{DECISIONS_NOUN}}` →
"binding design decisions", `{{DECISIONS_NOUN_SINGULAR}}` → "binding design decision", and make
`{{ADR_CONTEXT_LINES}}`/`{{ADR_STOP_CLAUSE}}` state that decisions live in the change's `design.md`.
When ADRs exist, use "ADR" wording and list the ADR files as binding context.

## Step 6 — Write the files (don't clobber blindly)

The target files are `CLAUDE.md`, the worker file(s) — `.claude/agents/worker.md` **or** one
`.claude/agents/worker-<stack>.md` per stack — `.claude/agents/reviewer.md`, and
`.claude/agents/supervisor.md`. For each:
- If it does **not** exist → write it (create `.claude/agents/` if needed).
- If it **does** exist → do **not** overwrite silently. Show the user a short diff/summary of what would
  change and ask (AskUserQuestion) whether to overwrite, merge, or skip. A pre-existing `CLAUDE.md` often
  has project content worth preserving — prefer merging the OpenSpec Workflow section in over
  replacing the whole file.

**Stamp every file you write** with the plugin's provenance marker, so a future
`/dmons:update-scaffold` knows which version generated it and which migrations to apply:

```
<!-- dmons-scaffold: <version> -->
```

`<version>` is the `version` field from the plugin's `plugin.json`. Place it **immediately after the
`OpenSpec Workflow` heading** in `CLAUDE.md` — at whatever level you wrote that heading; when merging
into an existing file you'll usually demote it to `##` under the project's own H1, and the `## 1.`…
`## 5.` sub-headings to `###` to match. That heading marks the region this skill owns; the rest of the
file may be the project's own, or another generator's. Put the stamp for each agent file **immediately
after the closing `---` of the frontmatter**.

Never place the stamp inside a marker-delimited block owned by another tool (e.g.
`<!-- CODEGRAPH_START -->` … `<!-- CODEGRAPH_END -->`).

This is the **one** comment that survives Step 5's "delete every `<!-- ... -->` comment" rule — the
templates deliberately don't contain it, so strip them clean and add the stamp here, as you write.

## Step 7 — Report

Summarise: which files were written/skipped, the section term chosen, the worker roster (single
full-stack, or which per-stack workers), the build/test gates wired in, and the binding decisions the
agents now enforce. Then tell the user the next step: run `/opsx:apply` (or `/opsx:propose` to create a
change first), and the Analyst/Architect will drive it **{{UNIT}} by {{UNIT}}, block by block** —
delegating each block to a `worker` and the `reviewer`, then each finished {{UNIT}} to the
`supervisor`, all through the shared `DEVLOG.md`.

Mention the cost shape explicitly, since it's the thing they'll feel: worker and reviewer are sonnet
and run per block; the supervisor is opus and runs once per {{UNIT}} (twice if it requests changes).

Note the version you stamped, and that a later plugin release is applied with `/dmons:update-scaffold`
rather than by re-running this skill.

---

## Guardrails
- Never invent binding decisions or ADRs. If the repo has none, say the agents derive their constraints
  from each change's `design.md`, and leave the decisions list lean.
- Detect build commands; never hard-code `dotnet`/`make` without confirming the repo actually uses it.
- Always strip template comments and unused optional lines from the generated files.
- The generated CLAUDE.md's "OpenSpec Workflow" section is authoritative and must keep its
  structure — tailor the commands, nouns, and worker roster, not the workflow shape (Analyst/Architect
  + Product Owner roles, the two nested loops, the {{UNIT}} review that closes the outer one, shared
  DEVLOG).
- Keep the models as the templates set them: `worker` sonnet, `reviewer` sonnet, `supervisor` opus.
- Keep the reviewer diff-local and the supervisor cross-block. If you find yourself writing the same
  check into both files, it belongs in the reviewer only — a supervisor that re-runs block review is
  an expensive no-op, which is the main way this workflow degrades.
- The `{{UNIT}}` base-commit post (`**[architect]** Base: <sha> — …`) is how the supervisor gets its
  review scope. Keep it in the generated CLAUDE.md's DEVLOG conventions **and** in §3a; dropping it
  leaves the supervisor unable to see the {{UNIT}}.
- **Always stamp** (Step 6). An unstamped file forces the next update to feature-detect its version,
  which gets less reliable with every release.
- This skill **generates**; it does not migrate. If the repo is already scaffolded, that's
  `/dmons:update-scaffold` — re-running this one throws away audited slot values and hand edits.
