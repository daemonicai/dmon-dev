---
name: implementation
description: Run the implementation phase of an OpenSpec change as the Architect — section by section, briefing a worker per section, having the reviewer audit it, then gating, ticking, and committing it, and closing with a supervisor review of the whole change. Wraps opsx:apply. Use when the user says "go ahead with the implementation", "implement the change", "start/continue implementing", "build the change", and — as the scaffolded CLAUDE.md instructs — at the start of every section and before the final change review. Needs a repo scaffolded by /dmons:scaffold at 0.6.0 or later; otherwise run /dmons:scaffold or /dmons:update-scaffold first.
---

# Implementation — building the change, section by section

This is the **procedure** for the implementation phase. The repo's `CLAUDE.md` holds the
**configuration** it runs against — the section term, the worker roster, the gate targets, the
human-in-the-loop examples, the commit footer — plus the roles, the boundaries, and one instruction:
invoke this skill at every section boundary. It wraps **`opsx:apply`**: where `opsx:apply` walks a
change's tasks, this skill walks them through the dmons roles — the Architect orchestrates, a `worker`
builds, the `reviewer` audits each section, the `supervisor` audits the whole change.

**Authority.** This skill is authoritative for the procedure. `CLAUDE.md` is authoritative for project
facts, roles, and boundaries. Any `opsx:*` skill is subordinate to both.

**You are re-invoked, not remembered.** Expect to be loaded at the start of the phase, again at the
start of every `## N.` section, and again before the change review. Each invocation starts with the
orientation in Step 0 and then does the *next* thing — never replay a section that has already landed.

---

## Step 0 — Orient (every invocation)

Cheap, and not optional: a long run or a compaction may have dropped everything else.

1. **Read the configuration.** Find the fenced block whose first line is `# dmons-config` in
   `CLAUDE.md` (`grep -n 'dmons-config' CLAUDE.md`) and read its YAML:
   - `unit` — the section term (*section* or *group*). Use it in every post and commit you write. This
     document says "section" throughout; substitute the repo's term.
   - `workers` — each worker's `name`, `stack`, and `gates` (its stack's `-k` gate set).
   - `gates` — the individual targets (`build`, `test`, `format`, `lint`, `validate`), `all` (the
     primary gate set), and `every_stack` on a multi-stack repo.
   - `decisions_noun`, `hitl_examples`, `coauthor`.

   **No configuration block → stop.** The repo was scaffolded before 0.6.0 and its `CLAUDE.md` still carries the old
   in-file procedure. Tell the Product Owner to run `/dmons:update-scaffold`; do not reconstruct the
   values from prose, and do not fall back to the old procedure.
2. **Check for version skew.** Read the provenance stamp under the `OpenSpec Workflow` heading
   (`<!-- dmons-scaffold: X -->`) and this plugin's version from
   `${CLAUDE_PLUGIN_ROOT}/.claude-plugin/plugin.json`.
   - **Equal** → carry on, silently.
   - **Only the patch differs** (`0.6.0` vs `0.6.1`) → carry on, and post one `[architect]` line to the
     DEVLOG naming both versions, once per change.
   - **Anything else** — a different minor or major, or two different prerelease strings
     (`0.6.0-preview1` vs `0.6.0-preview2`) → **stop and notify** (see *Stopping*). The agents in
     `.claude/agents/` were written for the stamped version and this procedure for the plugin's; running
     one against the other is how two people on the same repo end up running different loops without
     either knowing. Between previews the fix is to re-scaffold; for a release it is
     `/dmons:update-scaffold`. The Product Owner may tell you to proceed anyway — record that in the
     DEVLOG.
3. **Find your place.** Work out which of these is true, from `tasks.md`, the DEVLOG, and `git status`:
   - **No change chosen yet** in this session → Step 1.
   - **A section is open** — its `Base:` is posted but it has not been committed → resume it in Step 3
     at the step the DEVLOG says it reached. If the working tree holds WIP from a stop, read the
     `STOPPED` post before touching anything.
   - **The previous section just landed** and the next has unticked tasks → Step 3 for the next section.
   - **Every task is ticked** and there is no `[supervisor]` `Approve` under `## Change review` → Step 4.
   - **The change review approved** → Step 5.

## Step 1 — Select the change

1. List active changes = directories in `openspec/changes/` **excluding `archive/`** (`make changes`).
2. **Always ask the Product Owner which change to implement**, even when there is exactly one. If there
   are none, say so and stop.
3. Resume point = the **first unticked `- [ ]` task** in that change's `tasks.md`.
4. **Check the previous run closed cleanly.** Ticked boxes are not proof a section passed review — a
   session can die between the reviewer's `Approve` and the commit. If the DEVLOG shows a section with a
   `Base:` but no commit for it in `git log`, resume that section (Step 3) before anything else.

## Step 2 — Pre-flight (once per change)

1. Read `proposal.md`, `design.md`, and the `specs/<capability>/spec.md` files the change touches.
2. **Working tree must be clean** (`git status`) unless Step 0 found WIP from a recorded stop. Otherwise
   a dirty tree is a stop.
3. **Change must validate:** the `gates.validate` target → `VALIDATE_EXIT:0`. If not, stop.
4. **Be on the change branch** `change/<change-name>`. Create it from the default branch if missing:
   `git switch -c change/<change-name>`.

## Step 3 — One section

Walk the `## N.` sections **in order**. One section is one unit of work: one worker brief, one review
loop, one gate run, one tick, **one commit**.

### 3.1 Open it

Post the section's **base commit** as the first entry under its `## N.` heading in the DEVLOG:

```
**[architect]** Base: <sha> — <one line: what this section delivers>
```

`<sha>` is the current `HEAD` (`git rev-parse --short HEAD`). It is the reviewer's scope for this
section, and the first section's base is the supervisor's scope for the whole change. Post it before
any work starts.

### 3.2 Brief the worker

Post the brief to the DEVLOG (`[architect]`, under the section): **every task in the section**
(`N.1`…`N.k`), the spec requirements the section delivers (excerpted, not referenced), the
`decisions_noun` that bind it, and the gates it must pass. The worker should not need to go hunting.

**Routing.** One full-stack `worker` takes every section. With per-stack workers, route the section to
its stack's worker; a section that genuinely spans stacks is briefed as **one invocation per stack's
worker**, each on a stated range of tasks, run one after the other. Say which in the brief.

**An oversized section may be split — by judgement, not by rule.** If a section is too large to brief
well in one go, brief the worker on a stated range (`N.1–N.8`), then the rest, and record the split
and your reason as a DEVLOG post. This is deliberately not a named concept: it has no template, no
numbering of its own, and it changes nothing else — the reviewer still audits the whole section, and the
section still lands as one commit. (It is the thing 0.5.x called a *block*, and it is gone on purpose.)

### 3.3 The worker implements

The worker implements the brief, self-tests, posts to the DEVLOG as it goes, and reports back with its
exit lines and the `N.M` tasks it completed.

- **If the tripwire reports at the end of your turn,** deal with it before the review: read what
  landed, `git reset --soft <the sha it names>`, untick anything you didn't tick, and post it to the
  DEVLOG. An ungated commit is not a starting point for a review.
- **If the worker stopped,** read why. A missing or stale `Makefile` target is yours: fix the Makefile,
  say so in the DEVLOG, re-brief. A product question — ambiguity, contradiction, scope, a wrong spec —
  is the Product Owner's: see *Stopping*.

### 3.4 The reviewer audits the section

Spawn `reviewer` on the section: the base SHA from 3.1, the section's spec requirements, and the
worker's report. Its scope is `git diff <base-sha>` plus untracked files — the section is uncommitted.
It posts its verdict under the section as `[reviewer]`.

### 3.5 Remediation — at most two passes

- **`Approve`** (or `Approve with nits` — roll the nits into `## NEXT`) → 3.6.
- **`Request changes`** → a **remediation pass**: brief a worker (the same stack's) citing the
  reviewer's post, then re-spawn the reviewer on the same scope. A remediation pass carries no new task
  numbers — the section's boxes are not ticked yet — and the DEVLOG is its record.
- **Two passes, then stop.** If the reviewer still requests changes after the second remediation pass,
  do not brief a third — stop and put it to the Product Owner (see *Stopping*). A section that won't
  converge in two passes usually means the spec or the section's breakdown is wrong, and more fixing
  won't resolve either.

### 3.6 Gates — all must pass before any box is ticked

Run each and **read its exit line**; a gate passed only when you saw its `LABEL_EXIT:0`:

- `gates.build` → `BUILD_EXIT:0`
- `gates.test` → `TEST_EXIT:0` — the section's new tests **and** every existing test
- `gates.format` → `FORMAT_EXIT:0` and `gates.lint` → `LINT_EXIT:0`, where configured
- `gates.validate` → `VALIDATE_EXIT:0`

`gates.all` runs the set in one `-k` pass and is the quickest way to the full picture — and on a
multi-stack repo, run the gate set of every stack the section touched (each worker's `gates`, or
`gates.every_stack`). A red set still needs the individual exit lines to say which gate failed. Never
conclude a gate passed from reading its output; quote the code in the DEVLOG.

A section commits green. If it must land with a failing test for a sound technical reason (a red test a
later section turns green), that is a deliberate Architect call — state the reason in the DEVLOG **and**
the commit body. Otherwise a failed gate sends you back to 3.5 as a remediation pass, not to a commit.

### 3.7 Human-in-the-loop tasks

If the section contains a task that only a human can verify (`hitl_examples` in the configuration),
the worker will have reported it as **needs human confirmation** with a verification recipe. Once the
reviewer has approved and the gates are green, **stop and notify** with that recipe — exact command,
what to do, what they should see — and wait for the Product Owner's confirmation before ticking. Post
the confirmation to the DEVLOG when it comes; the supervisor will look for it.

### 3.8 Tick, commit, close

1. **Tick** every `- [x] N.M` in the section in `tasks.md`.
2. **Commit** — one conventional commit for the section, with the DEVLOG in it:
   ```
   feat(<change-name>): <section summary> (§N)

   - N.1 <task summary>
   - N.2 <task summary>
   ...

   <coauthor>
   ```
3. **Rewrite `## NEXT`** in the DEVLOG to point at the next section.
4. **Re-invoke `/dmons:implementation`** before opening the next section — or, if that was the last
   section, before the change review. The tripwire hook reminds you after the commit; that reminder is
   the cue.

## Step 4 — The change review

When every section has landed:

1. Add a **`## Change review`** heading to the DEVLOG, after the last `## N.` section and before
   `## NEXT`, and open it with:
   ```
   **[architect]** Change review of <change-base>..HEAD — <N> sections, <commits> commits.
   ```
   `<change-base>` is the `Base:` posted under the first section.
2. **Spawn `supervisor`** on that range. Point it at the change's spec requirements, not just its
   tasks. It posts its verdict under `## Change review` as `[supervisor]`.
3. **`Approve`** → Step 5. Roll its architectural notes into `## NEXT`.
4. **`Request changes`** → **stop and notify. Do not remediate on your own authority.** Every section is
   already committed and ticked, so a remediation pass here rewrites work that has landed — unlike the
   reviewer's loop, which runs before the commit, with the gates still ahead. Put the findings to the
   Product Owner with the options: remediate, accept and park the findings in `## NEXT`, or fix the
   spec.
   - If they choose to remediate: brief a worker citing the supervisor's post; spawn the `reviewer` on
     the pass, naming its range (the uncommitted diff against `HEAD`); run the gates; commit it as a
     fix, not a feature:
     ```
     fix(<change-name>): address change review findings

     - <finding> — <what changed>
     ...

     <coauthor>
     ```
   - Then **re-run the supervisor** on `<change-base>..HEAD`, now including the fix.
   - **Two passes, then stop.** A change that still draws `Request changes` after two remediation
     passes goes back to the Product Owner whatever their earlier instruction was.

## Step 5 — Done

When every task is ticked and the change review has an `Approve`:

1. Report to the Product Owner: sections landed, commits made, the test summary, and any architectural
   notes parked in `## NEXT`.
2. **Propose archiving** — offer to run `/opsx:archive` and **wait for confirmation**. Do not archive
   automatically.

---

## Stopping — halt, record, notify

The general stop triggers are in `CLAUDE.md` and hold in every phase. This phase adds its own:

- a **human-in-the-loop** task is ready for verification (3.7);
- the reviewer still requests changes **after two remediation passes** (3.5);
- the supervisor requests changes **at all** (Step 4);
- a **version skew** or a **missing configuration block** (Step 0);
- a dirty tree or a failing `validate` at pre-flight (Step 2).

The Product Owner may be away from the transcript and answering remotely, so a stop has three parts,
and all three are required:

1. **Leave the state recoverable.** WIP stays **uncommitted**; do **not** tick; do **not** revert. The
   gap between stopping and an answer may be hours, and the session may die rather than resume.
2. **Record it.** Post `**[architect]** STOPPED at N.M — <exactly what blocked, and the question>` to the
   DEVLOG, and rewrite `## NEXT` so a cold session can pick it up from the file alone.
3. **Notify, then ask.** Send a push notification with a one-line summary — use the `PushNotification`
   tool, loading it with ToolSearch first if it is deferred. If no such tool is available, say so in the
   DEVLOG post; the question that ends your turn is then the only notification. Then end the turn with
   the question itself: precise, answerable in a line, and — for a human-in-the-loop check — with the
   copy-pasteable recipe.

**Never decide-and-flag.** Resolving a product question yourself so the run can keep moving, and
noting that you did, moves the Product Owner's call to you. Stop instead. That is what the pause is for.

## Guardrails

- **Re-invoke; don't run from memory.** Open no section without this skill freshly loaded. If you
  notice you are mid-section with no recollection of this procedure, stop and invoke it.
- **One section, one commit.** Not one per task, not one per worker invocation, not one per split —
  one, after the reviewer's `Approve` and green gates. Remediation after the change review lands as a
  separate `fix(...)` commit.
- **Read the configuration; never infer it.** The gate targets, the worker roster, and the section term
  come from the `# dmons-config` block. If it disagrees with the prose elsewhere in `CLAUDE.md`, the
  configuration block wins for this procedure — say so in the DEVLOG so the Product Owner can fix the prose.
- **Only you commit, tick, and invoke agents.** The hooks enforce it on the agents; nothing enforces it
  on you except this procedure.
- **Keep the auditors' lenses apart.** The reviewer audits each section; the supervisor audits the whole
  change, once, at the end. Do not invoke the supervisor mid-change and do not ask the reviewer to look
  across sections. If a long change is suffering from late-surfacing drift, raise it with the Product
  Owner as a finding about the workflow.
- **No hidden blocks.** Splitting an oversized section is a judgement recorded in one DEVLOG post. If
  you find yourself splitting every section, the sections are wrong — say so to the Product Owner
  rather than working around it.
