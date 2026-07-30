# dmon-dev

A Claude Code plugin marketplace for the **dmon** family of repos (`dcli`, `dmon-core`, `dmon-meko`,
`dmon-websearch`, …).

## Plugins

| Plugin | What it does |
|--------|--------------|
| [`dmons`](plugins/dmons) | Scaffolds the OpenSpec Apply Workflow into a repo — generates a tailored Analyst/Architect `CLAUDE.md` plus one or more `worker` subagents, a per-block `reviewer`, and a per-section `supervisor`, by auditing the repo's openspec specs and changes. Also ships `/devlog`, which maintains a change's shared `DEVLOG.md`. |

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

Then, in a target repo that already has `openspec/` set up:

```
/dmons:scaffold
```

The skill audits the repo's specs and changes and writes `CLAUDE.md`, the worker file(s)
(`.claude/agents/worker.md`, or one `worker-<stack>.md` per tech stack), `.claude/agents/reviewer.md`, and
`.claude/agents/supervisor.md` — each tailored to that project's tech stack, build gates, and binding
decisions. The generated `CLAUDE.md` casts the main thread as an **Analyst/Architect** working for you
(the **Product Owner**) and drives apply as **two nested loops**: blocks go worker → reviewer → commit,
and each finished section gets a whole-section audit from the `supervisor` before the next one opens —
all coordinated through a shared `DEVLOG.md`.

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
