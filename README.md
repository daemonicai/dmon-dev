# dmon-cc

A Claude Code plugin marketplace for the **dmon** family of repos (`dcli`, `dmon-core`, `dmon-meko`,
`dmon-websearch`, …).

## Plugins

| Plugin | What it does |
|--------|--------------|
| [`openspec-orchestration`](plugins/openspec-orchestration) | Scaffolds the OpenSpec Apply Workflow into a repo — generates a tailored orchestrator `CLAUDE.md` plus `worker` and `reviewer` subagents by auditing the repo's openspec specs and changes. |

## Install

From any Claude Code session:

```
# Add this marketplace (local path or git remote)
/plugin marketplace add /Users/emmz/github/emmz/dmon-dev
# or, once pushed:  /plugin marketplace add <git-owner>/<repo>

# Install the plugin
/plugin install openspec-orchestration@dmon-cc
```

Then, in a target repo that already has `openspec/` set up:

```
/openspec-orchestration:scaffold
```

The skill audits the repo's specs and changes and writes `CLAUDE.md`, `.claude/agents/worker.md`, and
`.claude/agents/reviewer.md`, each tailored to that project's tech stack, build gates, and binding
decisions.

## Layout

```
dmon-cc/
├── .claude-plugin/
│   └── marketplace.json          # marketplace manifest (lists plugins)
└── plugins/
    └── openspec-orchestration/
        ├── .claude-plugin/
        │   └── plugin.json        # plugin manifest
        ├── skills/
        │   └── scaffold/
        │       ├── SKILL.md       # the generator
        │       └── templates/     # annotated skeletons it fills
        └── README.md
```

## License

[Mozilla Public License 2.0](LICENSE).
