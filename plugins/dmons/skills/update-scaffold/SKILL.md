---
name: update-scaffold
description: Migrate a repo that was already scaffolded with an earlier version of the dmons OpenSpec Apply Workflow up to the current one — adds new agents, applies workflow changes to CLAUDE.md, and updates the existing agents, one confirmable migration at a time, preserving audited content and hand edits. Use when the user says "update the openspec agents", "my agents are out of date", "migrate the scaffold", "I installed a new dmons version", or when /dmons:scaffold detects an already-scaffolded repo.
---

# Update an existing scaffold

`/dmons:scaffold` generates the Apply Workflow into a fresh repo. **This skill migrates a repo that
already has it** onto a newer version of the plugin — adding agents that didn't exist, applying workflow
changes to `CLAUDE.md`, and updating the agents that did.

It is **not** a re-scaffold. The generated files are not a deterministic render of the templates; each
one holds three intermingled things:

1. **Template structure** — the workflow shape. This is what a new plugin version changes.
2. **Audited slot values** — binding decisions, domain hazards, build commands, HITL examples, engineer
   titles. Expensive to recreate and often tuned since.
3. **Hand edits** — anything the project added after scaffolding.

Regenerating destroys (2) and (3). So this is a **merge**, and every change is applied as a discrete,
confirmable migration.

## The core move: harvest, don't re-audit

The previous scaffold's audit is **crystallised in the existing files**. `reviewer.md` already carries
the filled binding decisions, build commands, and domain hazards; `CLAUDE.md` already carries the gates,
the `{{UNIT}}` term, and the commit convention. **Read those values out of the existing files and reuse
them** rather than re-auditing the repo.

Re-audit **only** for slots that are genuinely new in the target version — a new agent's sections that
have no counterpart anywhere in the old files. Those are named per migration.

This matters for fidelity, not just cost: a fresh audit will phrase the same binding decisions
differently, and the drift shows up as agents that no longer agree with each other.

---

## Step 1 — Establish where the repo is

1. **Is it scaffolded at all?** Look for a heading whose **text** is `OpenSpec Workflow` in `CLAUDE.md`,
   and `.claude/agents/` containing a `reviewer.md` and at least one `worker*.md`.

   **Match the heading at any level** — `grep -nE '^#{1,4} +OpenSpec Workflow'`. The template writes it
   as `#`, but a generated `CLAUDE.md` nests it under the project's own H1, so in a real repo it is
   usually `##`, with `## 1. Select the change` … `## 5. Done` demoted to `###` to match. **Never grep
   for a literal `# OpenSpec Workflow`** — it misses ordinary scaffolded repos, and a false "not
   scaffolded" sends the user to `/dmons:scaffold`, which regenerates over their audited content. When
   in doubt, ask; do not conclude "not scaffolded" from one failed grep.

   **Record the heading level you found.** Everything in Step 5 is written relative to it.
   - **Not scaffolded** → this is the wrong skill. Tell the user to run `/dmons:scaffold`, and stop.
2. **Which version generated it?** Look for the provenance stamp — an HTML comment on the line after
   the `OpenSpec Workflow` heading in `CLAUDE.md`, and after the frontmatter in each agent:
   ```
   <!-- dmons-scaffold: 0.3.0 -->
   ```
   - **Stamp present** → that's the source version. Apply every migration **after** it, in order.
   - **No stamp** → the repo predates stamping (0.2.x or earlier). Feature-detect instead: a scaffolded
     repo with **no `.claude/agents/supervisor.md`** is pre-0.3.0. Treat the source version as `0.2.x`.
3. **Target version** = the plugin version in `${CLAUDE_PLUGIN_ROOT}/../.claude-plugin/plugin.json`
   (or `plugin.json` at the plugin root).
4. **Already current?** If the stamp equals the target, say so and stop — offer `/dmons:scaffold` only
   if they actually want to regenerate from scratch.

## Step 2 — Load the migration notes

Migrations live at `${CLAUDE_PLUGIN_ROOT}/skills/update-scaffold/migrations/<version>.md`, one file per
release that changed the generated output. Read **every** file between the source version (exclusive)
and the target (inclusive), in version order, and concatenate their migration lists.

Each note names its migrations, says which files they touch, gives the exact target content, and states
what to harvest versus what to audit fresh. **The notes are authoritative** — do not infer migrations by
diffing templates against the repo. A template diff shows *what* text differs; it cannot tell you which
differences are deliberate version changes and which are that repo's tuned slot values.

## Step 3 — Harvest the existing values

Before touching anything, read the existing generated files and extract the values a migration might
need. Typical harvest (the migration notes name what each one needs):

| Value | Harvest from |
|-------|--------------|
| The `{{UNIT}}` term actually in use (*section* / *group*) | `CLAUDE.md` §3 headings and `tasks.md` |
| Project name, description, tagline | `CLAUDE.md` header |
| Build / test / format commands and the gate list | `CLAUDE.md` "Commands" and the gates in §3 |
| Binding decisions, and the nouns used for them | `reviewer.md` / `worker*.md` decisions section |
| Domain hazards, style bullets, language idioms | `reviewer.md`, `worker*.md` |
| Worker roster — single or per-stack, and the names | `.claude/agents/worker*.md` |
| Engineer / principal titles | the agents' opening lines |
| Commit convention and co-author line | `CLAUDE.md` §3 commit block, and `git log` |

**Note the repo's `{{UNIT}}` term explicitly.** If it is anything other than *section*, every piece of
new text you write must use that term — otherwise the updated files will disagree with the ones you
didn't touch.

## Step 4 — Present the migration plan

Show the user, before applying anything:

- source version → target version;
- the ordered list of migrations, each as one line: what file, what change, and whether it is
  **additive** (a new file or a new section) or an **in-place edit**;
- anything you detected as hand-edited — a section whose content has clearly diverged from what the
  source version's template would have produced. Flag these; they're where a migration is most likely
  to conflict.

Then apply them **one at a time**, using AskUserQuestion (or a clear inline confirm) per migration:
**apply / skip / show me the exact edit first**.

- **Additive migrations** (new agent file, new `## 3c` section) are low-risk — still confirm, but batch
  the trivial ones if the user says to go ahead.
- **In-place edits** to a section the user has hand-tuned: show the current text and the proposed text
  side by side before touching it.
- **Skipping is a legitimate outcome.** A partially-migrated repo is fine as long as the stamp reflects
  it — see Step 6.

## Step 5 — Apply

Follow each migration note exactly. General rules:

- **Heading levels are relative, never absolute.** Migration notes describe structure in template terms
  (`## 3.`, `### 3a.`), but a real `CLAUDE.md` is usually demoted one level — `### 3.` with sub-parts at
  `#### 3a.`. Read the level of the section you're editing and write new headings **relative to it**.
  Importing the template's absolute levels breaks the document outline.
- **Other tools may own regions of `CLAUDE.md`.** Marker-delimited blocks (e.g.
  `<!-- CODEGRAPH_START -->` … `<!-- CODEGRAPH_END -->`) belong to another generator. Never edit inside
  one, never place the stamp inside one, and never treat their markers as scaffolding to strip.
- **Never strip HTML comments from an existing file.** That rule belongs to `/dmons:scaffold` Step 5,
  which applies it to freshly-filled templates. Here, every comment already in the file is either
  another tool's marker or a deliberate note.
- **Preserve, don't recreate.** When a migration adds a paragraph to an existing file, insert it; do not
  rewrite the surrounding section to match the template.
- **Match the local voice.** If the repo's `CLAUDE.md` has been edited into a different register, keep
  its register rather than importing the template's.
- **Fill new slots from the harvest first**, and audit the repo only for what the note explicitly says
  must be audited fresh.
- **Never delete a hand edit to apply a migration.** If the two genuinely conflict, stop that migration,
  show the user both, and let them decide.
- **New agent files get the full treatment** — fill every slot, strip every guidance comment, exactly as
  `/dmons:scaffold` Step 5 requires. A half-filled agent is worse than no agent.

## Step 6 — Stamp

After the migrations are applied, write the provenance stamp into **every file the scaffold owns** —
`CLAUDE.md` (immediately after the `OpenSpec Workflow` heading, whatever level it is) and each agent
(immediately after the closing `---` of the frontmatter):

```
<!-- dmons-scaffold: <version> -->
```

- **All migrations applied** → stamp the target version.
- **Some skipped** → stamp the **last fully-applied version**, and tell the user which migrations were
  skipped and that the repo will be offered them again next time. A stamp that overstates what was
  applied is worse than no stamp: the next update will skip work that never happened.

## Step 7 — Note it in any in-flight change

If a change is mid-apply (an active change with an unticked `tasks.md`), the workflow just changed under
it. Post one `[architect]` note to that change's `DEVLOG.md` under the current `## N.` heading saying
what changed and what it means for the remaining work.

**Do not retro-fit the new conventions into existing DEVLOG posts.** The DEVLOG is append-only and is
the durable record of how the change was actually built — a section built under the old workflow has no
base commit and no section review, and rewriting it to pretend otherwise is a lie in the permanent
record. If the target version needs something the in-flight change lacks, say so in the note and let the
Architect handle it from the next section on.

## Step 8 — Report

Summarise: source → target version, which migrations applied, which were skipped and why, which files
changed, the version now stamped, and anything the user should verify by hand. If any migration was
skipped, say plainly that the repo is on a mixed workflow and what that means in practice.

## Guardrails

- **Never regenerate a file wholesale** to apply a migration. If you find yourself rewriting a whole
  agent, you have left the merge and started a re-scaffold — stop and ask.
- **Never re-audit for a value that can be harvested.** Fresh phrasing of the same decision is drift.
- **Never invent a migration** that isn't in a migration note. If the repo looks wrong in a way no note
  covers, report it as an observation for the user, not as an edit.
- **Never stamp a version whose migrations you did not fully apply.**
- **Don't touch `openspec/`.** Specs, changes, and tasks belong to the project; this skill only updates
  the orchestration layer that acts on them.
- If the repo is dirty (`git status`), say so before editing — these are the files that drive every
  subsequent session, and the user will want a clean diff to review.
