# dmons 0.6 — splitting CLAUDE.md and `/dmons:implementation`

**Status:** proposed. Nothing implemented. Companion to `dmons-0.6-loop-redesign.md`, which redesigns
the loop this note relocates. Either can ship without the other; together they are 0.6.

**Settled:** the skill is `/dmons:implementation`, and it **wraps** `opsx:apply` — the same
relationship discovery has to `opsx:explore` and architecture to `opsx:propose`. Its description ends
`Wraps opsx:apply.`, where the other two state theirs. It is named for the phase, never for the
command it wraps: `/dmons:apply` was rejected for colliding with the ordinary sentence "apply the
change".

## 1. What the file holds today

`CLAUDE.md.template` is 310 lines in two halves, with a `---` at line 99:

| Lines | Section | Kind |
|---|---|---|
| 9–17 | `# {{PROJECT_NAME}}` + description | project fact |
| 18–40 | The DEVLOG — the shared working channel | mixed |
| 41–72 | Commands — the Makefile is the command surface | project fact |
| 73–99 | Boundaries — enforced by hooks, not by trust | project fact |
| 100–114 | `# OpenSpec Workflow` — authority clause + the three-phase map | mixed |
| 115–149 | Roles | identity |
| 150–161 | 1. Select the change | procedure |
| 162–170 | 2. Pre-flight | procedure |
| 171–247 | 3. Implement (3a opening, 3b each block) | procedure |
| 248–282 | 3c. Closing a {{UNIT}} — the supervisor review | procedure |
| 283–302 | 4. Stop and ask | mixed |
| 303–310 | 5. Done | procedure |

Roughly 150 of 310 lines are procedure that only matters during the implementation phase, and they
are re-read into context in every session that touches the repo, whatever the session is doing.

## 2. The principle

Not "identity stays, procedure moves" — that framing loses the placeholders. The sharper split:

> **`CLAUDE.md` is the repo's *configuration* for the workflow. The skill is the *procedure* that
> reads it.**

Everything the scaffold audit discovered about *this repo* — the section-container term, the worker
roster, the gate commands, the HITL examples, the binding decisions — stays in the committed file,
because it is the audit's output and cannot live in a plugin. Everything that is the same in every
repo — the loop, the ordering, the handoffs — moves into the skill, because keeping N copies of it
in N repos is what makes every loop change a migration.

## 3. The cut

**Stays in `CLAUDE.md`:** project header, the DEVLOG's *definition*, the Makefile command surface,
the whole Boundaries section, the three-phase map, Roles, and the Product Owner's authority.

**Moves to `/dmons:implementation`:** select-the-change, pre-flight, the section loop, the audit
handoffs, the gate/tick/commit cadence, the phase-specific stop triggers, and the done/archive steps.

Estimated result: `CLAUDE.md.template` drops from ~310 lines to ~150.

## 4. The four places it doesn't cut cleanly

### 4a. The authority clause inverts — this is the dangerous one

Line 102 currently reads:

> **This section is authoritative.** If a skill's behavior ever conflicts with what's written here,
> **follow this document.**

That sentence exists to stop an `opsx:*` skill overriding the workflow. Move the procedure into a
skill and the sentence, left as-is, instructs the Architect to prefer a stale copy of the loop over
the live one — silently, and exactly when they disagree.

It must be **rewritten, not moved**, and split by kind: `CLAUDE.md` stays authoritative for **project
facts, roles, and boundaries**; `/dmons:implementation` is authoritative for **the procedure**. Any
`opsx:*` skill remains subordinate to both. Getting this wrong is worse than not doing the split.

### 4b. Roles must stay, though it sits under `# OpenSpec Workflow`

The Roles section is inside the half that looks like it should move, but the main thread must know it
is the Analyst/Architect **for the whole session** — including during `opsx:explore` and
`opsx:propose`, which `/dmons:implementation` never runs. Move it and discovery and architecture lose
their role definition.

What *does* move out of Roles: the two paragraphs describing what the auditors do and when each is
spawned. Those are loop mechanics. The roster itself — who exists, and that only the main thread
invokes agents — stays.

### 4c. Stop-and-ask splits down the middle

"These are the Product Owner's calls, not yours" holds in every phase and stays. The specific
triggers are phase-bound and move — the HITL bullet, the supervisor-still-requests-changes bullet,
and the mid-work rule about leaving WIP uncommitted.

### 4d. The DEVLOG is defined in one place and used in another

Its definition (what it is, where it lives, that posts are append-only and `## NEXT` is rewritten)
stays. The per-step instructions — post the base commit here, post the verdict there — move with the
loop. `/devlog` remains a separate skill and is unaffected.

## 5. Mechanism: the skill has to read the repo's configuration

Today the procedure is *rendered* per repo — scaffold fills `{{UNIT}}`, `{{WORKER_ROLE_LINES}}`,
`{{HITL_EXAMPLES}}`, `{{BUILD_CMD}}` and the rest before writing the file. A plugin skill cannot be
pre-rendered, so it must **read those values from the repo at runtime**, and `CLAUDE.md` becomes the
place it reads them from.

Practically, the skill needs, on invocation:

- the section-container term (*section* vs *group*) — from the committed prose;
- the worker roster — one full-stack `worker`, or `worker-<stack>` per stack;
- the gate target names, so it can quote `LABEL_EXIT:<n>` correctly;
- the HITL examples, for the stop-and-ask triggers;
- the scaffold provenance stamp (§7).

This argues for those values being **greppable rather than merely present** — a small stable block in
`CLAUDE.md` the skill can parse, rather than the skill inferring them from prose it hopes hasn't been
hand-edited. Worth designing deliberately; it is the part of this split most likely to be brittle.

## 6. The tradeoff: the procedure leaves the repo

Today the workflow is entirely in-repo. Clone it and you have the whole thing, committed, versioned
with the project, readable by a teammate who has never installed dmons. After this split, the repo
holds the configuration and the agents; the procedure arrives from the plugin.

Mitigation, not a fix: `CLAUDE.md` should say plainly that the implementation procedure lives in
`/dmons:implementation` and that the plugin is required to run the phase. A committed pointer to an
absent procedure is a better failure than silence.

**This is the Product Owner's call, and it is the one genuinely irreversible decision in 0.6.**
Everything else here can be walked back by editing a template.

## 7. The migration win is real but partial — and it buys a new problem

The pitch was: move the loop into the plugin and future loop changes stop needing a migration note.
That holds for the ~150 lines of procedure. It does **not** hold for everything:

- The **agent files stay in the repo** — they must, because plugin subagents silently ignore `hooks:`
  frontmatter and the boundary enforcement would evaporate. Their prompts describe the loop too
  ("you implement one section…", "the Architect commits"), so a loop change still touches
  `worker.md`, `reviewer.md`, `supervisor.md` and still needs a migration note.
- What genuinely stops needing migration is the *orchestration* half.

And it introduces **version skew**. Today the procedure is pinned in-repo and moves only when someone
runs `/dmons:update-scaffold` and confirms each step. After the split it floats with whatever plugin
version each person has installed — so two people on the same repo can run different procedures, on
the same change, with nothing telling either of them.

The trade is therefore: **deliberate-but-laborious migration → automatic-but-silent drift.** That
wants a guard. The scaffold already stamps each generated file (`<!-- dmons-scaffold: 0.5.1 -->`);
the skill should read that stamp on invocation and say something when the repo's scaffold predates
the plugin's expectations, rather than quietly running a newer loop against older agents.

## 8. Re-invocation, and what unattended actually needs

### 8a. Control flow must live in the file that never decays

A skill invocation is a tool call the main thread makes, so the Architect can re-invoke
`/dmons:implementation` at each section boundary and have the procedure re-land in context
immediately ahead of the section it governs.

The bootstrap catch: **if the instruction to re-invoke lives in the skill, the skill decaying out of
context is exactly what stops the re-invocation.** So the split gets one turn sharper than §3:

> `CLAUDE.md` holds the loop's **control flow**. The skill holds each section's **procedure**.

CLAUDE.md keeps roughly three lines — the implementation phase runs section by section, invoke
`/dmons:implementation` at the start of each one, do not open a section without it — and those lines
are always in context because the file always is. The ~150 lines of procedure still move to the
plugin, so the size win survives.

This also settles the compaction question, which matters more for long runs than interactive ones: a
summarized context loses the skill's procedure but keeps `CLAUDE.md`, so what survives is precisely
the instruction to reload.

### 8b. Firmer than prose

That is still prose the Architect is trusted to follow, and 0.5.0 exists because prose was not
enough. The precedent for doing better is already in the repo: **`dmons-tripwire.sh` is a hook that
injects information into the Architect's context at a chosen moment.** A `SubagentStop` hook could
inject "you are between sections — re-invoke `/dmons:implementation` before opening the next one" on
the same mechanism.

It must inject a **reminder, not the procedure**. A hook that injected the procedure would have moved
it back into the repo and forfeited the split. Exact event and payload shape need checking against
the hooks documentation before this is committed to.

### 8c. Unattended = unsupervised but reachable

The Product Owner answers via remote control, so **stop-and-ask and HITL tasks are acceptable breaks
in an unattended run**: a stop parks the run until it is answered rather than ending it. The policy
is therefore **halt and notify**. Park-and-continue is unnecessary (it would need a dependency notion
between sections that `tasks.md` does not express), and decide-and-flag is rejected outright — it
would move product calls to the Architect and quietly rewrite the Product Owner's role.

Three consequences for the procedure:

1. **A stop must notify, not merely stop.** §4 of `CLAUDE.md` currently says "stop **immediately** and
   ask", which assumes someone is watching the transcript. Unattended, "ask" has to mean *emit a
   notification*. Every stop trigger needs an explicit notify step.
2. **The existing WIP rule is already right and becomes more important.** Leave it uncommitted, do
   not tick, do not revert, log the stop in the DEVLOG with the exact task. With section-sized units
   that WIP may sit for hours, and the DEVLOG entry is what makes it recoverable if the session dies
   rather than pauses.
3. **The supervisor's verdict should notify and pause, not auto-remediate** — see the loop note §4.

What is *not* solved by reachability: an unattended run's failure mode is not the stops, it is the
things that never stop. A worker going subtly wrong produces a reviewer `Request changes` and loops
automatically; only the second failed round escalates. So in a long unattended run the DEVLOG and the
per-section commits are the entire audit trail, which raises the value of the tripwire and of keeping
commits section-granular rather than change-granular.

## 9. Open questions

1. **Where do the parameters live** — parsed from prose, or a dedicated greppable block (§5)?
2. **Which hook event carries the re-invocation reminder** (§8b), and does its payload reach the main
   thread's context the way the tripwire's does?
3. **Does the split ship with the loop redesign or after it?** Doing both at once means one migration
   and one coherent 0.6; doing them in sequence means the loop change can be validated on a real
   change before the procedure's location also moves.
