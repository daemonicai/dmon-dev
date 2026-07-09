# dmon-dev

A Claude Code plugin marketplace for the **dmon** family of repos (`dcli`, `dmon-core`, `dmon-meko`,
`dmon-websearch`, …).

## Plugins

| Plugin | What it does |
|--------|--------------|
| [`dmons`](plugins/dmons) | Scaffolds the OpenSpec Apply Workflow into a repo — generates a tailored Analyst/Architect `CLAUDE.md` plus one or more `worker` subagents and a `reviewer`, by auditing the repo's openspec specs and changes. Also ships `/devlog`, which maintains a change's shared `DEVLOG.md`. |

## Install

From any Claude Code session:

```
# Add this marketplace (local path or git remote)
/plugin marketplace add /Users/emmz/github/emmz/dmon-dev
# or, once pushed:  /plugin marketplace add <git-owner>/<repo>

# Install the plugin
/plugin install dmons@dmon-dev
```

Then, in a target repo that already has `openspec/` set up:

```
/dmons:scaffold
```

The skill audits the repo's specs and changes and writes `CLAUDE.md`, the worker file(s)
(`.claude/agents/worker.md`, or one `worker-<stack>.md` per tech stack), and `.claude/agents/reviewer.md`
— each tailored to that project's tech stack, build gates, and binding decisions. The generated
`CLAUDE.md` casts the main thread as an **Analyst/Architect** working for you (the **Product Owner**) and
drives apply **block by block** through the `worker`/`reviewer` split and a shared `DEVLOG.md`.

## Layout

```
dmon-dev/
├── .claude-plugin/
│   └── marketplace.json          # marketplace manifest (lists plugins)
└── plugins/
    └── dmons/
        ├── .claude-plugin/
        │   └── plugin.json        # plugin manifest
        ├── skills/
        │   ├── scaffold/
        │   │   ├── SKILL.md       # the generator
        │   │   └── templates/     # annotated skeletons it fills
        │   └── devlog/
        │       └── SKILL.md       # the shared-channel DEVLOG skill
        └── README.md
```

## License

[Mozilla Public License 2.0](LICENSE).
