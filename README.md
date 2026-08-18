# dmon-dev

A Claude Code plugin marketplace for the **dmon** family of repos (`dcli`, `dmon-core`, `dmon-meko`,
`dmon-websearch`, …).

## Built for OpenSpec

Everything here is designed to work with **OpenSpec**. The `dmons` plugin is the orchestration layer on
top of it, not a replacement for it: the skills assume a repo with `openspec/` in it, they read the
repo's specs and changes, and they hand off to the `/opsx:*` commands (`opsx:explore`, `opsx:propose`,
`opsx:apply`) at each step. OpenSpec itself — the CLI, the `/opsx:*` commands, the `openspec-*` skills —
comes from `openspec init`, and this marketplace does not install it. Run that first; without it, none of
the skills below have anything to work with.

## Plugins

| Plugin | What it does |
|--------|--------------|
| [`dmons`](plugins/dmons) | Walks an OpenSpec project from an empty repo to a building one: `/dmons:discovery` gathers requirements as the Analyst, `/dmons:architecture` makes the technology decisions as the Architect, and `/dmons:scaffold` generates a tailored Analyst/Architect `CLAUDE.md` plus one or more `worker` subagents, a per-block `reviewer`, and a per-section `supervisor` by auditing the repo's openspec specs and changes. Also ships `/devlog`, which maintains a change's shared `DEVLOG.md`. |

## Install

From any Claude Code session:

```
# Add this marketplace
/plugin marketplace add daemonicai/dmon-dev

# Install the plugin
/plugin install dmons@dmon-dev
```

To pick up later updates, refresh the marketplace and start a new session:

```
/plugin marketplace update dmon-dev
```

## Where to start

Both entry points assume the target repo has already had `openspec init` run in it.

**A brand-new project** — an empty, OpenSpec-initialized repo with no open or archived changes and no
real code yet — starts at discovery:

```
/dmons:discovery      # requirements, as the Analyst — assumes nothing about the how
/dmons:architecture   # platform, language, frameworks, hosting — as the Architect
/dmons:scaffold       # the Apply Workflow agents that build it
```

`/dmons:discovery` draws the *what* and *why* out of you as the Product Owner and hands off to
`opsx:explore`; `/dmons:architecture` then decides the *how* against those requirements and hands off to
`opsx:propose`. Scaffold comes last, because it needs at least one change to audit.

**An existing repo** — one that already has changes or code — skips discovery and goes straight to:

```
/dmons:scaffold
```

The skill audits the repo's specs and changes and writes a `Makefile`, `CLAUDE.md`, the worker file(s)
(`.claude/agents/worker.md`, or one `worker-<stack>.md` per tech stack), `.claude/agents/reviewer.md`, and
`.claude/agents/supervisor.md` — each tailored to that project's tech stack, build gates, and binding
decisions. The `Makefile` is the command surface every agent runs its gates through, and each gate target
prints its own exit code (`BUILD_EXIT:0`) so a result is quoted rather than read out of a log. The
generated `CLAUDE.md` casts the main thread as an **Analyst/Architect** working for you
(the **Product Owner**) and drives apply as **two nested loops**: blocks go worker → reviewer → commit,
and each finished section gets a whole-section audit from the `supervisor` before the next one opens —
all coordinated through a shared `DEVLOG.md`.

It also writes two **boundary hooks** into `.claude/hooks/`. The commits, the ticked boxes, and the
decision to invoke an agent belong to the Architect alone — they are the only evidence a block was
gated — so the agents are blocked from all three rather than asked to stay off them.

Already scaffolded with an earlier version? Don't re-run `/dmons:scaffold` — it would discard the
audited content and any hand edits. Run:

```
/dmons:update-scaffold
```

It reads the version stamp the scaffold left behind (feature-detecting for pre-0.3.0 repos, which have
none), then applies each release's changes as a discrete, confirmable migration — harvesting existing
values rather than re-auditing, so your tuned decisions and hazards survive.

## Layout

```
dmon-dev/
├── .claude-plugin/
│   └── marketplace.json              # marketplace manifest (lists plugins)
└── plugins/
    └── dmons/
        ├── .claude-plugin/
        │   └── plugin.json            # plugin manifest
        ├── skills/
        │   ├── discovery/
        │   │   └── SKILL.md           # greenfield requirements, as the Analyst
        │   ├── architecture/
        │   │   └── SKILL.md           # technology decisions, as the Architect
        │   ├── scaffold/
        │   │   ├── SKILL.md           # the generator
        │   │   └── templates/         # annotated skeletons it fills
        │   ├── update-scaffold/
        │   │   ├── SKILL.md           # migrates an already-scaffolded repo
        │   │   └── migrations/        # one note per release that changed output
        │   └── devlog/
        │       └── SKILL.md           # the shared-channel DEVLOG skill
        └── README.md
```

## License

[Mozilla Public License 2.0](LICENSE).
