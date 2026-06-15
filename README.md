# claude-code-guided-tour

An interactive guided tour that **teaches** you (or an agent) an unfamiliar
codebase. Claude explores the repo with a recon subagent, fans out parallel
subagents to study each subsystem, then walks you through a sequenced lesson
one stop at a time — high-level first, grounded in real code.

It is the teaching partner to
[`claude-code-quiz-master`](https://github.com/flyte/claude-code-quiz-master):
the **tour teaches**, the **quiz tests**.

## Install

```
/plugin marketplace add flyte/claude-plugins
/plugin install claude-code-guided-tour@flyte
```

## Use

- `/tour` — take the overall guided tour of the whole codebase.
- `/tour-dive <area>` — deep dive into one subsystem (e.g. `/tour-dive auth`).

During the overall tour you drive the pace: `next`, `dig in`, `skip`,
`jump <area>`, `stop`.

### Options

- `--level <beginner|intermediate|expert>` — depth for this session.
- `--set-level <level>` — set and remember your level.
- `--refresh` — re-explore the codebase from scratch.
- `--model <haiku|sonnet|opus>` — override the model used by the exploration
  subagents for this run.
- `--smoke` — run a self-check that the plugin is installed correctly (no tour).

## How it works

The first run explores the codebase and caches a **curriculum** at
`.claude/tour-map.json` (an ordered set of teaching "stops"). Later runs reuse
it and only re-explore when the codebase has changed meaningfully, so they're
fast. The cache is git-ignored.

## Models & cost

Exploration is the dominant token cost (one subagent per subsystem, run in
parallel), so the subagents default to cheaper models rather than inheriting
your main session model:

| Phase | Default model |
|-------|---------------|
| Recon | Haiku |
| Deep-dive fan-out | Sonnet |
| Deep-dive re-drill (`/tour-dive`) | Sonnet |
| Synthesis (curriculum assembly) | your session model |

Override per run with `--model <haiku\|sonnet\|opus>`. If your environment sets
`CLAUDE_CODE_SUBAGENT_MODEL`, that takes precedence over both the defaults and
`--model`.

## Requirements

`git` and `jq` on PATH. Staleness tracking needs a git repo; outside one the
tour still works but won't auto-refresh.

## License

MIT
