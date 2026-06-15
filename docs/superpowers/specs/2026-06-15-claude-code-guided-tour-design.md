# claude-code-guided-tour — Design

**Date:** 2026-06-15
**Status:** Approved design, ready for implementation planning

## Summary

A Claude Code plugin that **teaches** a user (or an agent) an unfamiliar codebase
through an interactive, in-session guided tour. It is the deliberate partner to
[`claude-code-quiz-master`](https://github.com/flyte/claude-code-quiz-master):
the tour **teaches** the codebase, the quiz **tests** it.

Claude explores the codebase with a recon subagent, fans out parallel subagents
to dive deeper into each subsystem, waits for them to return, then synthesizes
their findings into a **curriculum** — an ordered lesson plan — and walks the
user through it conversationally, one stop at a time.

The deliverable is an **ephemeral in-session walkthrough**. The only persisted
artifact is an internal **codebase map / curriculum** cached to disk so both
commands can reuse the expensive exploration instead of re-running it every time.

## Pedagogical foundation

The lesson plan is grounded in how developers actually learn unfamiliar
codebases (researched during brainstorming). Principles adopted:

1. **Top-down before bottom-up.** Open with "what is this and why does it
   exist," establish the high-level mental model, *then* descend into detail.
2. **Follow the data / trace a real flow.** Experts understand systems by
   following an execution path end-to-end. Every tour includes at least one
   concrete worked example: a real request/data path traced through the layers.
3. **Progressive (staged) disclosure.** Working memory is limited (Cognitive
   Load Theory, Sweller). The tour is paced in discrete, user-driven "stops" —
   never a wall of text.
4. **Concrete usage over abstract signatures.** Show real call sites, not just
   interfaces.
5. **Ground every "why" in evidence.** False rationale poisons a mental model.
   Deep-dive subagents must ground "why" claims in actual code, comments, or git
   history — never speculate.
6. **End each unit with "read next" + gotchas.** A concrete map of where to go
   and what to watch for, not a bare file list.
7. **Hierarchical, modular, breadth-first.** Map subsystems first (breadth),
   then depth on demand.

### Key reframe: curriculum-as-artifact

The cached artifact is not just a map of facts — it is a **curriculum**: an
ordered sequence of teaching "stops," each with a learning objective. The
synthesis step turns raw subagent findings into a sequenced lesson plan suited
to a newcomer (orientation → architecture → one end-to-end trace → subsystems,
central/simple first).

## Architecture

Lightweight: skill + slash commands + plain `bash` for git. **No helper CLI.**
The deterministic surface (two git commands and a JSON read/write) is too thin
to justify a build/test toolchain; the value is in subagent orchestration and
synthesis, which is prompt work regardless. This is a deliberate divergence from
quiz-master, whose CLI existed for genuinely error-prone grading math and
rolling-window state.

```
claude-code-guided-tour/
  .claude-plugin/plugin.json     # name, version, description, author, license
  commands/tour.md               # /tour — overall guided lesson
  commands/tour-dive.md          # /tour-dive <area> — deep lesson on one subsystem
  skills/guided-tour/SKILL.md    # orchestration + pedagogy + map schema (the brain)
  README.md
  LICENSE
```

Both commands are thin: they invoke the `guided-tour` skill and pass arguments
through, exactly as quiz-master's `/quiz` invokes `quiz-master`.

## The cached artifact: `.claude/tour-map.json`

Project-relative, like quiz-master's `quiz-map.json`. Holds the curriculum plus
the metadata needed for staleness checks and level-awareness.

```jsonc
{
  "schemaVersion": 1,
  "builtAtSha": "<git HEAD sha when built>",
  "builtFileCount": 0,            // tracked-file count at build time, for drift
  "level": "intermediate",        // beginner | intermediate | expert
  "project": {
    "summary": "one-paragraph what-and-why",
    "languages": ["..."],
    "buildSystem": "...",
    "entryPoints": ["..."]
  },
  "endToEndTrace": {              // the representative worked example
    "name": "e.g. an incoming HTTP request",
    "steps": [
      { "description": "...", "file": "path", "symbol": "fn/class" }
    ]
  },
  "stops": [
    {
      "id": "kebab-id",
      "title": "...",
      "subsystem": "...",
      "objective": "what you'll understand after this stop",
      "framing": "the one-sentence 'think of X as…' mental model",
      "keyFiles": [ { "path": "...", "role": "..." } ],
      "callSites": [ { "path": "...", "symbol": "...", "note": "..." } ],
      "workedExample": "optional: a flow to trace within this subsystem",
      "gotchas": ["..."],
      "readNext": ["..."],
      "dependsOn": ["other-stop-id"]   // dependency hints for ordering
    }
  ],
  "order": ["stop-id", "..."]     // the sequenced curriculum
}
```

The map is internal scaffolding, not user-facing prose. Claude narrates *from*
it; it does not paste it.

## Orchestration: the three-phase fan-out

**Phase 1 — Recon (1 subagent, blocking).**
A single `Explore`-style agent surveys the repo: languages, build system, entry
points, top-level subsystems, and selects one representative end-to-end flow.
Returns a shortlist of subsystems — **capped at ~6–8** to bound cost — plus the
candidate trace. This is the table of contents before deep work begins.

**Phase 2 — Deep-dive fan-out (N subagents, parallel).**
One subagent per subsystem from Phase 1, dispatched together in a single message
so they run concurrently. Each returns **structured findings** for its area:
purpose, key files/call sites, local data flow, the "why" (grounded in
code/comments/git — no speculation), and gotchas. The main agent waits for all
to return before proceeding.

**Phase 3 — Synthesis (main agent, no subagent).**
Collapse findings into the curriculum: order the stops (orientation →
architecture → end-to-end trace → subsystems, central/simple first), write each
stop's objective, framing, and read-next, record dependency hints, and persist
`.claude/tour-map.json` with the current git sha and file count.

## Commands

### `/tour` — the overall guided lesson

```
---
description: Take an interactive guided tour that teaches you this codebase
argument-hint: "[--level <level>] [--set-level <level>] [--refresh]"
---
```

1. **Load or build.** Load `.claude/tour-map.json`. If missing or stale, run
   Phases 1–3 first. `/tour` is self-sufficient — it never requires another
   command to have run first.
2. **Walk the curriculum** one stop at a time (staged disclosure). Each stop:
   framing → concrete anchors (files/call sites) → worked example/trace if any →
   gotchas → read-next.
3. **User drives the pace** after each stop:
   - `next` — advance to the next stop
   - `dig in` — zoom into the current stop in more depth without leaving the tour
   - `skip` — skip the current stop
   - `jump <area>` — jump to a specific subsystem
   - `stop` / `done` / `exit` — end the tour
4. **Level-aware depth** (beginner/intermediate/expert), persisted in the map,
   same `--level` / `--set-level` convention as quiz-master.
5. **On exit**, softly suggest `/quiz <area>` to test what was just taught.

### `/tour-dive <area>` — deep lesson on one subsystem

```
---
description: Take a deep-dive lesson into one area of this codebase
argument-hint: "<area> [--level <level>] [--refresh]"
---
```

1. **Load or build** the map (build if missing).
2. **Resolve `<area>`** against the map's subsystems by fuzzy match. If
   ambiguous, ask the user to disambiguate. If no `<area>` is given, list
   available subsystems.
3. **Re-drill if thin.** If the map's detail for that area is insufficient for a
   deep lesson, dispatch a fresh focused subagent to drill deeper *before*
   teaching — so a dive is always deeper than the overview gave.
4. **Teach the subsystem** as its own mini-lesson using the same stop structure.
5. **Close** by suggesting sibling areas, read-next, and `/quiz <area>`.

## Argument handling

Parsed by the skill (passed through from the commands):

- Bare positional (`/tour-dive`) → the target area.
- `--level <beginner|intermediate|expert>` → session-only override (do not
  persist).
- `--set-level <level>` → persist to the map's `level` field and exit.
- `--refresh` → force a full rebuild (Phases 1–3) before proceeding.

## Staleness

Inline in the skill (lightweight):

- On load, get git HEAD sha (`git rev-parse HEAD`) and the tracked-file count
  (`git ls-files | wc -l`).
- Compare against the map's `builtAtSha` / `builtFileCount`.
- Rebuild when the sha differs **and** drift exceeds a threshold (e.g. more than
  ~15% of files changed, or the sha is unreachable), or when `--refresh` is
  passed. A small drift reuses the existing map — teaching tolerates mild
  staleness better than grading does.
- If the directory is not a git repo, skip staleness entirely and build once per
  session (note this to the user).

## Relationship to quiz-master

- **Conceptual pair:** tour teaches, quiz tests. The tour cross-sells `/quiz` on
  exit; this is a soft suggestion with **no hard dependency** in either
  direction (no shared files, no required install order).
- **Shared conventions** for familiarity: project-relative `.claude/*.json`
  artifact, `--level` / `--set-level` level system, thin-command-invokes-skill
  structure.
- **Deliberate divergence:** no helper CLI (justified above).

## Error handling

- **Empty / tiny repo:** if recon finds too little to teach, say so plainly and
  offer to tour whatever exists rather than fabricating subsystems.
- **Subagent failure in fan-out:** proceed with the subsystems that returned;
  note which areas are missing and offer `/tour-dive <area> --refresh` to fill
  gaps later.
- **Unparseable / corrupt map:** treat as missing and rebuild.
- **Ambiguous `/tour-dive` area:** ask to disambiguate; never guess silently.
- **No git:** see Staleness above.

## Out of scope (YAGNI)

- Persisted user-facing tour documents (deliverable is in-session narration).
- A helper CLI / build toolchain.
- Progress tracking across sessions beyond the cached map + level.
- Visual/graph rendering of the codebase.
- Any hard runtime coupling to quiz-master.

## Success criteria

- Running `/tour` on an unfamiliar repo produces a coherent, correctly ordered
  lesson that starts high-level and descends, includes at least one grounded
  end-to-end trace, and is paced one stop at a time.
- `/tour-dive <area>` reliably teaches a single subsystem in more depth than the
  overview, drilling further when the map is thin.
- The second invocation of either command is markedly faster by reusing the
  cached map, and refreshes correctly when the codebase has meaningfully changed.
- Every "why" claim in the tour is traceable to real code, comments, or history.
```
