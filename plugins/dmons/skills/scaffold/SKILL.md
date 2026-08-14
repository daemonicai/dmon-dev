---
name: scaffold
description: Scaffold the OpenSpec Apply Workflow into the current repo — generate a tailored Analyst/Architect CLAUDE.md plus one or more `worker` subagents, a `reviewer` subagent, and a `supervisor` subagent by auditing the repo's openspec specs and changes. Use when the user wants to "set up the openspec agents", "add the worker/reviewer/supervisor agents", "scaffold the OpenSpec workflow", "bootstrap CLAUDE.md for openspec", or onboard a new repo to the block-by-block apply workflow.
---

# Scaffold the OpenSpec Apply Workflow

Generate the repo-local files, each tailored to the current project by auditing its OpenSpec specs and
changes:

- `Makefile` — the project's **command surface**: one gate target per stack (`build`, `test`, `format`,
  and `lint` where it's distinct), the OpenSpec `validate`/`changes` pair every repo shares, and the
  `-k` gate sets that compose them. **Every gate target prints its own exit code** as `LABEL_EXIT:<n>`
  and exits with it, so an agent quotes a code instead of interpreting a log. Every other generated file
  cites these target names and nothing else. This is the one generated file you must show the Product
  Owner in full before writing (Step 6) — after which its gate targets are separately offered as an
  allowlist in the repo's committed `.claude/settings.json`, so the workflow's gates stop prompting.
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
- `.claude/hooks/dmons-guard.sh` + `.claude/hooks/dmons-tripwire.sh` — the **boundary hooks**. Copied
  verbatim (they hold no audited values), they turn the agents' Boundaries sections from prose into
  something the harness enforces: the guard blocks the calls, the tripwire catches what the guard
  couldn't see. Step 6 wires them.

**The model split is deliberate.** The worker and reviewer run on every block, so they are the hot path
and stay on sonnet; the supervisor runs once per section and carries opus. Preserve this when
generating — a supervisor on a weaker model removes the workflow's only cross-block check.

**The boundary hooks are not optional decoration.** The workflow's correctness rests on three things
belonging to the Architect alone — the commits, the ticked boxes, and the decision to invoke an agent —
and prose alone does not hold them: a worker that has just finished a block will sometimes tick its own
tasks and commit its own work, which lands code no gate ever ran. Generate the hooks, and keep the
`disallowedTools` and `hooks:` frontmatter in every agent file exactly as the templates set it.

The skill does **not** install OpenSpec, the `/opsx:*` commands, or the `openspec-*` skills — those come
from the `openspec` CLI (`openspec init`). It only produces the orchestration layer that wraps them.

Scaffold is the **third** step of the greenfield chain: **`/dmons:discovery`** (the Analyst gathers
requirements — the *what*) → **`/dmons:architecture`** (the Architect decides the technology — the
*how*, recorded in each change's `design.md ## Decisions` and any ADRs) → **`/dmons:scaffold`** (this
skill, which audits those decisions to generate the Apply Workflow agents). By the time scaffold runs the
project should already have at least one change carrying the decisions the agents will enforce; if it
doesn't, that's the signal to run discovery and architecture first (see Step 1).

The templates live at `${CLAUDE_PLUGIN_ROOT}/skills/scaffold/templates/` — `Makefile.template`,
`CLAUDE.md.template`, `worker.md.template`, `reviewer.md.template`, `supervisor.md.template`,
`dmons-guard.sh.template`, `dmons-tripwire.sh.template`. They are
annotated skeletons with `{{PLACEHOLDER}}` slots
and guidance comments — `<!-- ... -->` in the markdown templates, `#!!`-prefixed lines in
`Makefile.template` and the two `.sh.template` files (neither has HTML comments, and their plain `#`
comments are part of the generated output). Your job is to fill every slot from the audit and **delete
every guidance comment** before writing the final files.

**The two shell templates are the exception to all of that**: they carry no `{{PLACEHOLDER}}` at all.
Copy them through verbatim minus the `#!!` lines. Do not "adapt them to the project" — a guard tuned
per repo is a guard nobody can reason about, and every hole in one is a rule that silently stopped
applying.

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

**Build system & gates** — detect, don't assume. What you're gathering here are the **raw commands**,
which become the Makefile's recipes; the agents never see them again, only the target names.

- `*.slnx` / `*.csproj` / `*.sln` → `dotnet build` / `dotnet test`; check for a `dotnet format` /
  analyzers / `TreatWarningsAsErrors` policy (a format gate). Analyzers run inside `dotnet build`, so
  .NET has **no separate lint target**.
- `package.json` → the project's `build` / `test` / `lint` / `format` scripts, and the prefix needed to
  run them if the package isn't at the repo root (`npm --prefix web run build`).
- `Cargo.toml` → `cargo build` / `cargo test` / `cargo clippy` (lint) / `cargo fmt --check` (format).
- `pyproject.toml` → the test runner plus the linter *and* formatter, which are usually distinct
  (`pytest`, `ruff check`, `ruff format --check`).
- Anything else: read the CI config or existing docs to find the real commands. CI is the best source —
  it holds the commands that actually gate the project.
- **Publish / clean:** a real packaging command (`dotnet pack -c Release`, `npm publish`,
  `cargo publish`) and a real clean, if the project has them. Don't invent either; a service deployed by
  CI often has no publish command at all.

**A format target must be a check, not a rewrite** (`--verify-no-changes`, `--check`). A target that
reformats the tree turns a gate into an unreviewed edit.

**An existing `Makefile`** is an audit source, not an obstacle: read its targets and reuse its recipes
as the raw commands. If it already follows this shape, most of the generation is confirmation. Note
which targets it lacks (`validate`, the `LABEL_EXIT:` echoes, the gate sets) — those are what Step 6
proposes adding. **Never overwrite a recipe the project already wrote**; propose additions and let the
Product Owner decide.

**Gates you can't derive.** If a stack's real test or format command isn't discoverable, don't guess and
don't quietly drop the target — carry it into Step 4 and ask. A missing gate target is a gate the
workflow will never run.

**Tech stacks (for the worker roster)**
- Enumerate the **distinct** tech stacks in the repo — a stack is a body of code with its own toolchain,
  idioms, and build/test commands. Examples: a Vite/TypeScript frontend + a `.NET`/`*.csproj` backend;
  a Unity/C# game + Objective-C (iOS) + Kotlin/Java (Android) native components; a Rust core + a Python
  binding.
- **One stack → one full-stack worker** (no question needed). **Two or more → ask in Step 4** whether
  the Product Owner wants a single full-stack worker or a dedicated worker per stack. Record each
  stack's own build/test command — a per-stack worker's gates and idioms come from its stack, not the
  repo average.
- **Stack naming drives the Makefile targets.** The **primary** stack (the one the project is mostly
  about) takes the unprefixed targets — `build`, `test`, `format` — and every additional stack is
  prefixed with its name: `web-build`, `web-test`, and the gate set `gates-web`. Pick each stack's short
  name here and use it everywhere: the target prefix, the `EXIT` label (`WEB_BUILD_EXIT`), and the
  worker name (`worker-web`) must all agree.

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
- the exact **raw** build / test / format / lint commands per stack that the Makefile will wrap (offer
  your detected guess as the recommended option) — and explicitly ask about any gate you **couldn't**
  derive rather than shipping a Makefile that's missing it;
- whether the project has a real `publish` and `clean` command, if the audit didn't settle it;
- **worker roster (only if Step 2 found more than one stack):** ask whether they want a **single
  full-stack worker** or **a dedicated worker per stack**. If per-stack, propose the worker names
  (`worker-<stack>`, e.g. `worker-frontend` / `worker-dotnet`) for confirmation. The reviewer and the
  supervisor are always a single agent each, regardless of the answer.
- whether to overwrite any target file that already exists (see Step 6).

Prefer making reasonable, clearly-stated decisions over interrogating the user. Don't ask about things
you can read.

## Step 5 — Fill the templates

**Fill `Makefile.template` first** — it defines the target names every other file cites, and getting
them settled first is what keeps the four files agreeing with each other.

For each template:
1. Read `${CLAUDE_PLUGIN_ROOT}/skills/scaffold/templates/<file>.template`.
2. Replace every `{{PLACEHOLDER}}` with the audited content. Keep the prose voice of the template —
   blunt, specific, imperative.
3. **Delete every `<!-- ... -->` comment**, including the header block. The final files must contain no
   template scaffolding. (Step 6 then adds the one comment that belongs in the output — the provenance
   stamp.)
   - **In `Makefile.template` the rule is different:** delete every line starting with `#!!`, and
     **keep** the plain `#` comments — the header explaining the `LABEL_EXIT:` convention and the note
     above `validate` are part of the generated file, written for whoever opens the Makefile next.
   - Recipe lines are **tab-indented**. A Makefile with space-indented recipes doesn't run; check
     before you write it.
4. For optional lines (format gate, graphify tools, ADR clauses): include them only when the audit says
   they apply; otherwise remove the line cleanly (no dangling placeholder, no empty bullet).

Key slots and where they come from:

| Slot | Source |
|------|--------|
| `{{PROJECT_NAME}}`, `{{PROJECT_DESCRIPTION}}`, `{{PROJECT_DESCRIPTION_SHORT}}` | `project.md`, specs |
| `{{TECH_STACK}}`, `{{ENGINEER_STRENGTHS}}`, `{{LANG}}`, `{{LANG_IDIOMS}}`, `{{STYLE_BULLETS}}` | detected language + conventions |
| `{{UNIT}}` | Step 3 — the section container's name (section/group) |
| `{{WORKER_NAME}}`, `{{STACK}}`, `{{WORKER_STACK_LINE}}`, `{{WORKER_ROLE_LINES}}`, `{{BLOCK_STACK_RULE}}` | Step 4 worker roster (see the multi-stack note below the table) |
| `{{RAW_BUILD_CMD}}`, `{{RAW_TEST_CMD}}`, `{{RAW_FORMAT_CMD}}`, `{{RAW_LINT_CMD}}`, `{{RAW_PUBLISH_CMD}}`, `{{RAW_CLEAN_CMD}}` | Step 2's detected raw commands — **`Makefile` only**. No other generated file ever names a toolchain command |
| `{{PROJECT_NAME}}`, `{{PRIMARY_STACK}}`, `{{PHONY_LIST}}`, `{{PRIMARY_GATE_TARGETS}}` | the stack names and target set settled in Step 2/4 |
| `{{EXIT_RATIONALE_EXAMPLE}}` | *optional* — one concrete sentence naming a tool in **this** project that exits non-zero while printing innocuous output (e.g. "`dotnet format --verify-no-changes` exits 2 while printing a single `warning: IDEnnnn` line"). Delete if the audit found no such case; don't invent one |
| `{{BUILD_CMD}}`, `{{TEST_CMD}}`, `{{VALIDATE_CMD}}`, `{{GATES_CMD}}`, `{{GATE_LIST}}`, the format/lint command and gate lines | **`make` target names** — `make build`, `make test`, `make validate`, `make gates`. A per-stack worker gets **its** stack's targets: `make web-build`, `make gates-web` |
| `{{EXTRA_STACK_COMMAND_LINES}}` | multi-stack only — one `CLAUDE.md` command line per additional stack's gate set, plus `make gates-all` |
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

### The `Makefile` — always show it, always ask

**Write the `Makefile` first, and never write it unprompted.** Show the Product Owner the **complete
proposed file** — whether or not the repo already has one — and ask **apply / edit / skip**
(AskUserQuestion). It is the one generated file whose contents they are most likely to have opinions
about, it encodes commands you inferred, and unlike the agent files a wrong recipe fails silently by
never running a gate.

- **No `Makefile` yet** → show the full proposal; on *apply*, write it.
- **One exists** → show a **merge**: their file with your additions marked, and say plainly which
  targets you'd add (usually `validate`, `changes`, the `LABEL_EXIT:` echoes, the gate sets) and which
  of their recipes you'd leave untouched. **Never rewrite an existing recipe** — if theirs lacks the
  exit-code echo, propose the echo as an addition to that recipe and let them accept or decline it.
- **Skip** → generate the rest anyway, but fill `{{BUILD_CMD}}` and friends with the **raw** commands
  rather than `make` targets, drop the exit-line assertions from `CLAUDE.md` and the agents, and say so
  in Step 7. Agents pointed at targets that don't exist are worse than agents pointed at real commands.

### Allowlist the gate targets — a separate ask

Once the Makefile is accepted, offer to allowlist its gate targets in **`.claude/settings.json`** (the
committed, team-wide file — *not* `settings.local.json`, which is gitignored and would only help
whoever ran scaffold). The generated agents are committed; the permissions they depend on belong beside
them, reviewable in the same PR.

**Ask separately from the Makefile.** Editing someone's permission config is its own kind of consent,
and the answers legitimately differ — yes to the Makefile, no to auto-approving it. Show the exact rules
and ask apply / skip.

- **Exact-match rules, one per gate target** actually emitted:
  ```
  "Bash(make build)", "Bash(make test)", "Bash(make format)", "Bash(make lint)",
  "Bash(make validate)", "Bash(make gates)", "Bash(make changes)"
  ```
  plus each additional stack's targets and gate set (`"Bash(make web-build)"`, `"Bash(make gates-web)"`,
  `"Bash(make gates-all)"`). Omit any target the Makefile doesn't define.
- **Never `"Bash(make *)"`.** The wildcard sweeps in `publish` and `clean` — the two targets that
  should still prompt. Exact matches are the whole point: the workflow invokes these targets bare, so
  nothing broader is needed.
- **`publish` and `clean` are deliberately excluded.** No agent runs them; they are the Product Owner's.
- **Merge, never replace.** Read the file first and append to the existing `permissions.allow` array,
  preserving every rule already there. If `.claude/settings.json` doesn't exist, create it with just the
  `permissions.allow` key. Leave `.claude/settings.local.json` alone entirely.
- **Skipped Makefile → skip this too.** Allowlisting targets that don't exist is noise.

Tell the user one thing they won't expect: the agents are told to run gates through **context-mode**
(`ctx_execute`) for large output, which is a separate MCP tool permission — these Bash rules cover the
direct invocations, not that path.

### The boundary hooks — copy, `chmod`, wire, verify

Write both scripts to `.claude/hooks/`, stripped of their `#!!` lines and otherwise **verbatim**:

- `dmons-guard.sh` — the `PreToolUse` guard. It is wired from **each agent's own frontmatter** (already
  in the templates), so it only ever sees that agent's calls. Workers pass the role `worker`; the
  reviewer and supervisor pass `auditor`.
- `dmons-tripwire.sh` — the before/after check around each agent's **run**, not around the Architect's
  Agent call: agents run in the background, so the call returns at launch and a `PreToolUse`/`PostToolUse`
  pair around it measures an empty window. It brackets `SubagentStart`/`SubagentStop` instead and reports
  on `Stop`. Wired in **`.claude/settings.json`**, because the reporting half has to run in the
  Architect's own session — that is the only session it is allowed to speak into.

Four things to get right, in order:

1. **`chmod +x` both files.** A hook script without the execute bit doesn't block anything — it fails,
   and a failing `PreToolUse` hook is not a denial. Verify it (`test -x`), don't assume it.
2. **They need `jq`.** The guard fails *closed* without it, so a machine with no `jq` will block every
   agent tool call with a clear message rather than waving them through. Say so in Step 7; it is the
   one way this setup can be loudly annoying.
3. **Wire the tripwire — a separate ask, like the allowlist.** This adds a `hooks` key to the committed
   `.claude/settings.json`, which is a bigger consent than a permission rule: show the exact JSON and
   ask apply / skip. **Merge into the existing file**; never replace it.
   ```json
   { "hooks": {
       "SubagentStart": [ { "matcher": "^(worker|worker-.+|reviewer|supervisor)$", "hooks": [ { "type": "command",
           "command": "\"$CLAUDE_PROJECT_DIR/.claude/hooks/dmons-tripwire.sh\" start" } ] } ],
       "SubagentStop":  [ { "matcher": "^(worker|worker-.+|reviewer|supervisor)$", "hooks": [ { "type": "command",
           "command": "\"$CLAUDE_PROJECT_DIR/.claude/hooks/dmons-tripwire.sh\" stop" } ] } ],
       "Stop":          [ { "hooks": [ { "type": "command",
           "command": "\"$CLAUDE_PROJECT_DIR/.claude/hooks/dmons-tripwire.sh\" report" } ] } ]
   } }
   ```
   The matcher is a regex over the **agent type** — the `name:` in each agent file's frontmatter, not the
   filename. If you generated workers under other names, widen the alternation to match and say which
   names you used; a matcher that misses is silent. `Stop` takes no matcher.
   Add `.claude/.dmons-tripwire/` to `.gitignore` — that's the scratch directory it snapshots into.
   Declining this is fine: the guard still does the preventing. Say which half they now have.
4. **Frontmatter hooks need the workspace trusted.** Claude Code skips a project-level agent's
   `hooks:` block until the folder containing it has been trusted, and it does so *quietly* — the agent
   still runs, unguarded. Tell the user to accept the workspace trust prompt, and give them the
   30-second check in Step 7.

**Never weaken a guard to make a block pass.** If the guard blocks something an agent legitimately
needs, that is a finding about the workflow, not a line to delete: bring it to the Product Owner.

Then the rest. The target files are `CLAUDE.md`, the worker file(s) — `.claude/agents/worker.md` **or**
one `.claude/agents/worker-<stack>.md` per stack — `.claude/agents/reviewer.md`, and
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

In the `Makefile`, which has no HTML comments, the stamp is a `#` comment on the **first line**:

```
# dmons-scaffold: <version>
```

In the two hook scripts the shebang must stay first, so the stamp is the **second** line, directly
below `#!/usr/bin/env bash`.

Stamp it only if you wrote or merged into it — a Makefile the Product Owner told you to skip isn't
yours to stamp.

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
full-stack, or which per-stack workers), the **Makefile targets** the agents now run (and any gate the
audit couldn't derive, so they know what's missing), and the binding decisions the agents now enforce.
Say explicitly that every gate prints `LABEL_EXIT:<n>` and that the agents are told to quote that code
rather than read the log — it's the part of this setup a Product Owner won't guess. Report whether the
gate targets were allowlisted in `.claude/settings.json`, and that `publish`/`clean` were deliberately
left out so they still prompt. If they skipped the
Makefile, say the agents are wired to raw commands instead and that `/dmons:update-scaffold` can offer
it again later. Then tell the user the next step: run `/opsx:apply` (or `/opsx:propose` to create a
change first), and the Analyst/Architect will drive it **{{UNIT}} by {{UNIT}}, block by block** —
delegating each block to a `worker` and the `reviewer`, then each finished {{UNIT}} to the
`supervisor`, all through the shared `DEVLOG.md`.

Report the **boundary hooks** plainly, because they change what the agents can physically do:

- what the guard blocks (git writes, `tasks.md`, the `Makefile`, `CLAUDE.md`/`.claude/`, spawning
  agents — across Bash *and* the `ctx_*` tools), that the auditors can write only `DEVLOG.md`, and that
  none of it constrains the Architect;
- whether they took the tripwire, and that without it they have prevention but no detection;
- that both need `jq`, and that the guard fails closed without it;
- **the 30-second verification**, which is worth doing once: ask a worker to commit something. It
  should come back with `BLOCKED by the OpenSpec Apply Workflow` rather than a commit. If it commits,
  the frontmatter hooks are being skipped — almost always an untrusted workspace or a missing execute
  bit — and every agent is running unguarded.

Mention the cost shape explicitly, since it's the thing they'll feel: worker and reviewer are sonnet
and run per block; the supervisor is opus and runs once per {{UNIT}} (twice if it requests changes).

Note the version you stamped, and that a later plugin release is applied with `/dmons:update-scaffold`
rather than by re-running this skill.

---

## Guardrails
- Never invent binding decisions or ADRs. If the repo has none, say the agents derive their constraints
  from each change's `design.md`, and leave the decisions list lean.
- Detect build commands; never hard-code a toolchain without confirming the repo actually uses it. The
  Makefile's *targets* are fixed by this skill; its *recipes* are always audited.
- **Raw commands live in the `Makefile` and nowhere else.** Once it's written, `CLAUDE.md` and the
  agents name `make` targets only. A `dotnet test` left in an agent file is a gate that prints no exit
  code and drifts the moment the toolchain moves.
- **Never invent a recipe to fill a target.** A gate you couldn't derive is a question for the Product
  Owner (Step 4), and a target that would run nothing is worse than an absent one — it reports
  `EXIT:0` for work it never did.
- **Never overwrite an existing Makefile recipe**, and never write the Makefile without showing it in
  full and getting an explicit apply (Step 6).
- **Never widen a permission beyond the gate targets.** The allowlist is exact-match rules for targets
  the Makefile actually defines — never `Bash(make *)`, never `publish`/`clean`, never a rule the
  workflow doesn't need. And always merge into `permissions.allow`; replacing that array would silently
  drop whatever the project had already approved.
- Always strip template comments and unused optional lines from the generated files — `<!-- ... -->`
  everywhere, `#!!` lines in the `Makefile`, whose plain `#` comments stay.
- The generated CLAUDE.md's "OpenSpec Workflow" section is authoritative and must keep its
  structure — tailor the commands, nouns, and worker roster, not the workflow shape (Analyst/Architect
  + Product Owner roles, the two nested loops, the {{UNIT}} review that closes the outer one, shared
  DEVLOG).
- Keep the models as the templates set them: `worker` sonnet, `reviewer` sonnet, `supervisor` opus.
- **Copy the hook scripts verbatim and never soften one.** They carry no audited values, so there is
  nothing to tailor; a repo-specific guard is a guard with repo-specific holes. If one blocks something
  an agent genuinely needs, that's a question for the Product Owner and a fix upstream in the plugin,
  not a local edit.
- **Never drop the `disallowedTools` or `hooks:` frontmatter** from a generated agent, and never point
  an agent's guard at the wrong role — a `reviewer` wired as `worker` can edit the whole tree, which is
  precisely the review it was meant not to have.
- **`chmod +x` is part of writing the hooks**, not an afterthought. A non-executable guard blocks
  nothing and says nothing; the workflow looks identical right up until an agent commits.
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
