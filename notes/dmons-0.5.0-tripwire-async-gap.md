# dmons 0.5.0 — the tripwire measures an empty window when agents run in the background

**Severity:** the detection half of hook-enforced boundaries never fires. It reports "all clear"
unconditionally, which is worse than not shipping it — the Architect reads silence as verification.

**Affects:** `plugins/dmons/skills/scaffold/templates/dmons-tripwire.sh.template`, and the wiring
described in `skills/update-scaffold/migrations/0.5.0.md` M6 and the `CLAUDE.md` M5 prose.

**Found in:** `devlog` (`/Users/rendle/github/daemonicai/devlog`), migrated 0.4.0 → 0.5.0 on 2026-08-14.
The installed `.claude/hooks/dmons-tripwire.sh` is byte-identical to the template (verified by diff,
modulo the stamp line), so this is not an install artefact.

## What happens

`dmons-tripwire.sh` is wired as a `PreToolUse`/`PostToolUse` pair on `Agent|Task`. `pre` writes a
snapshot of `HEAD` and each active change's tick count to `.claude/.dmons-tripwire/<call-id>`; `post`
reads it back, compares against current state, reports any movement, and deletes the snapshot
(`rm -f "$SNAPSHOT"`, line 73).

That pairing assumes the `Agent` tool call returns when the agent has finished. **In Claude Code, agents
run in the background: the tool call returns as soon as the agent is launched**, and the result arrives
later as a task notification. So `PostToolUse` fires seconds after `PreToolUse`, while the agent has not
yet done any work.

## Evidence

From the live repo, spawning a worker for one block:

```
tripwire dir created:  14:17:35     <- pre wrote the snapshot
tripwire dir modified: 14:17:37     <- post compared and rm'd it
wall clock:            14:19:43     <- worker still running, 2+ minutes later
```

The snapshot was created and destroyed inside a two-second window. Everything the agent went on to do —
including anything it committed or ticked — happened after the comparison, with the baseline already
deleted. In that session the worker did nothing wrong, so the tripwire's silence was correct by accident;
it would have been equally silent had the worker committed its own block.

## Why this matters more than the mechanism

The stated purpose (migration note, M6) is that the tripwire "catches what the guard couldn't see, and it
still works in a repo where frontmatter hooks are being skipped." Skipped frontmatter hooks are precisely
the failure mode M7 warns about — silent, invisible in the transcript. The tripwire is the backstop for
that, and the backstop is inert.

The prevention half (`dmons-guard.sh`, `PreToolUse` on the agent's own calls) is unaffected: those hooks
run synchronously with the calls they guard. Only detection is broken.

## Suggested fix

`SubagentStop` looks like the right event — it fires when a subagent finishes, which is the moment the
pairing actually wants. Worth confirming it exposes enough to correlate with the `pre` snapshot; if the
call id isn't available there, a single `latest` snapshot per repo would still be sound given agents are
spawned one at a time in this workflow.

Failing that, a **rolling baseline** avoids the event-timing question entirely: on each tripwire
invocation, compare current state against the last recorded baseline, report drift, then overwrite the
baseline. Movement is caught one agent-call late rather than immediately, but it is caught — and it
degrades safely, because a stale baseline over-reports rather than under-reports.

What should not survive the fix is the current shape, where the absence of a report is indistinguishable
between "nothing moved" and "the window was empty."

## Also worth a line in the migration note

M7's end-to-end check ("ask a worker to commit something; it should come back `BLOCKED`") is currently
the *only* way to confirm enforcement is live, since the tripwire cannot corroborate it. That makes it
load-bearing rather than optional, and it reads as a nice-to-have in the note today.
