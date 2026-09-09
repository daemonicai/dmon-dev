# dmons 0.6 — release plan

Companion to `dmons-0.6-loop-redesign.md` and `dmons-0.6-claude-md-split.md`. **Both halves ship
together**, proven through previews before `0.6.0`.

## 1. The plan

`0.6.0-preview1` … `-previewN` → tested on the Product Owner's day-job machine against a real repo →
`0.6.0` when the exit criteria in §6 are met. Both design notes land in the same release; the loop
redesign and the CLAUDE.md split are not separable in testing, since the split exists to hold the new
loop.

## 2. Iterating between previews: wipe and re-scaffold

`update-scaffold` cannot carry a repo from one preview to the next (§3), so between previews the test
repo is **reset and re-scaffolded from scratch** rather than migrated.

`scaffold` detects an already-scaffolded repo and hands off to `update-scaffold`, so the reset has to
be **complete** or it will refuse to run fresh. Everything the scaffold owns:

| Path | Note |
|---|---|
| `CLAUDE.md` | revert to the pre-scaffold backup (project facts only) |
| `.claude/agents/worker.md` *or* `worker-<stack>.md` | delete all variants |
| `.claude/agents/reviewer.md` | delete |
| `.claude/agents/supervisor.md` | delete |
| `.claude/hooks/dmons-guard.sh` | delete — stamp is on line 2 |
| `.claude/hooks/dmons-tripwire.sh` | delete — stamp is on line 2 |
| `Makefile` | stamp is `# dmons-scaffold:` on line 1 |
| `.claude/settings.json` | **merged into, not owned** — remove only the gate permission rules and the tripwire wiring, keep the rest |

Cheapest way to make this repeatable: commit the pre-scaffold state on the test repo, then reset those
paths to that commit before each re-scaffold. `settings.json` is the one file that needs care rather
than deletion.

Expect to re-answer scaffold's audit questions (build/test/format/lint commands, tagline, full-stack
vs per-stack workers) on every iteration. If that friction bites, it is itself a finding worth noting.

## 3. Why preview → preview migration does not work

Not a bug to fix for 0.6 — a property of the migration format worth recording so nobody trips on it.

`update-scaffold` reads the provenance stamp as the source version and applies every migration file
**strictly between source (exclusive) and target (inclusive)**, from
`skills/update-scaffold/migrations/<version>.md` (SKILL.md lines 57, 78–80). With a single
`migrations/0.6.0.md` and a repo stamped `0.6.0-preview1` targeting `0.6.0-preview2`, there is no file
in that interval: **nothing is applied, the repo is re-stamped, and the preview's changes never land.**
Silently.

Making it work would mean one migration note per preview, with `0.6.0.md` as their union — carrying
maintenance for versions nobody will ever migrate from. Not worth it. Re-scaffold instead (§2), and
write exactly **one** `migrations/0.6.0.md`, kept accurate to the final shape.

Also note SKILL.md line 73: if the stamp equals the target, `update-scaffold` stops. **Every preview
needs its own version number** — iterating without bumping means the tool declines to do anything.

## 4. The migration path still has to be proven

§2 means the path existing users take — `0.5.1` → `0.6.0` — gets no exercise during previews. It is
also the riskiest part of the release: three agents re-scoped, the reviewer's model changed, and
`{{ARCHITECTURAL_COHERENCE_BULLETS}}` / `{{DOMAIN_HAZARDS}}` harvested from prose authored at the old
altitude (loop note §11).

**Keep a 0.5.1-stamped repo snapshot** and run `update-scaffold` against it once the shape has stopped
moving — at the last preview, before tagging `0.6.0`. Check the harvested slot values read correctly
one level up, not just that the migration completed.

## 5. Distribution — previews must not reach the marketplace

This repo *is* the distribution channel. A version bump in `.claude-plugin/marketplace.json` on `main`
is public to anyone who runs `/plugin marketplace update dmon-dev`. **Previews belong on a branch**,
with `main` left on `0.5.1` until `0.6.0` is proven.

To confirm before the first push:

- **Does the plugin loader accept a prerelease version string** (`0.6.0-preview1`) in `plugin.json`
  and `marketplace.json`? A rejected manifest is a bad thing to discover on the day-job machine.
- **Can a marketplace be added from a branch**, so the day-job machine installs the preview without
  it reaching `main`? If not, an alternative is needed before preview1 — a separate marketplace entry,
  or a local install path.

## 6. Exit criteria for 0.6.0

Each maps to something the design notes assert but have not demonstrated:

1. **A real multi-section change, end to end** — the baseline.
2. **At least one large section** (~15+ tasks) taken by a single worker invocation. This is the claim
   blocks existed to hedge; if it fails anywhere it fails here.
3. **The re-invocation actually fires at each section boundary** (split note §8a) — verify in the
   transcript, not by assuming. If it silently stops happening, the procedure is out of force and
   nothing says so.
4. **One unattended run with a stop-and-ask answered over `rc`**, confirming the stop notifies rather
   than merely halting (split note §8c.1).
5. **One supervisor `Request changes` at end of change**, confirming it notifies and pauses rather
   than auto-remediating committed work (loop note §4).
6. **A deliberately provoked tripwire** — ask a worker to commit — confirming enforcement survived the
   re-scoping. Still the only end-to-end proof that the boundary hooks are live.
7. **The 0.5.1 → 0.6.0 migration on a restored snapshot** (§4).
8. **A version-skew check**: a repo scaffolded on an earlier preview against a later plugin should say
   something, not quietly run a newer loop against older agents (split note §7).

## 7. Docs

`README.md` and `plugins/dmons/README.md` still describe blocks, the per-block reviewer, and the
per-section supervisor. Leave them until the shape settles, then update once for `0.6.0` — a README
rewritten per preview is a README that will disagree with itself. Standing rule applies: if a preview
falsifies a claim in a published release's notes, the correction goes on the published release rather
than being quietly dropped.
