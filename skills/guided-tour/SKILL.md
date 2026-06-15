---
name: guided-tour
description: Teaches the user (or an agent) an unfamiliar codebase via an interactive guided tour. Explores with a recon subagent, fans out parallel subagents to study each subsystem, synthesizes their findings into an ordered curriculum cached at .claude/tour-map.json, then narrates it one stop at a time. Partner to claude-code-quiz-master (tour teaches, quiz tests). Invoked by the /tour and /tour-dive slash commands.
---

# guided-tour

You teach the user their codebase. They opted in via `/tour` (whole-codebase
lesson) or `/tour-dive <area>` (deep dive into one subsystem). Your job is to
build a correct mental model in their head, grounded in real code — not to dump
files at them.

This plugin is the teaching partner to `claude-code-quiz-master`: the tour
teaches, the quiz tests. There is no hard dependency between them.

## Pedagogical rules (non-negotiable)

1. **Top-down before bottom-up.** Start with what the project is and why it
   exists, then the architecture, then details.
2. **Trace a real flow.** Always include at least one concrete end-to-end trace;
   following the data teaches better than listing files.
3. **One stop at a time.** Staged disclosure. Never dump the whole curriculum —
   present a stop, then wait for the user.
4. **Concrete over abstract.** Show real call sites, not bare signatures.
5. **Ground every "why" in evidence.** Never speculate about rationale. (The
   deep-dive subagents enforce this; carry it through when you narrate.)
6. **Close each stop with gotchas + read-next.**

## Mode

The invoking command tells you the mode:
- `/tour` → **overall tour**: walk the whole curriculum.
- `/tour-dive <area>` → **deep dive**: teach one subsystem in depth.

## Argument parsing

Arguments are passed through from the command. Parse:
- Bare positional (deep dive only) → the target area.
- `--level <beginner|intermediate|expert>` → session-only depth override; do NOT
  persist.
- `--set-level <level>` → write `level` into the map, confirm the update to the
  user, then stop (do not run a tour). If no map exists yet, do NOT trigger a
  full exploration just to store a preference — write a minimal valid map
  (`{"schemaVersion":1,"level":"<level>","stops":[],"order":[]}`); the next
  `/tour` will build the curriculum.
- `--refresh` → force a full rebuild (Phases 1–3) before proceeding.

Level controls depth: **beginner** = more framing, less jargon, smaller steps;
**expert** = terser, assumes fluency, more cross-references.

## Loading the map

Map path: `.claude/tour-map.json` (project-relative). Schema:
`${CLAUDE_PLUGIN_ROOT}/skills/guided-tour/references/map-schema.md`.

1. **Integrity check.** If the file exists, validate it:
   ```bash
   jq -e 'has("schemaVersion") and (.schemaVersion == 1) and has("stops") and has("order")' .claude/tour-map.json
   ```
   If `jq` exits non-zero (missing, corrupt, truncated, wrong version), treat the
   map as **missing**.
2. **Staleness check** (only if integrity passed and not `--refresh`):
   - `git rev-parse HEAD` → current sha; `git ls-files | wc -l` → current count.
   - Compare to the map's `builtAtSha` / `builtFileCount`.
   - **Rebuild** when the sha differs AND file-count drift exceeds ~15%; also
     rebuild if the stored `builtAtSha` is no longer in the repo
     (`git cat-file -e <builtAtSha>` exits non-zero — e.g. after a force-push or
     rebase). Otherwise reuse the map — teaching tolerates mild staleness. (A
     failed `git rev-parse` means this isn't a git repo — handled by the next
     bullet.)
   - If this is **not a git repo** (`git rev-parse` fails), skip staleness, note
     to the user that staleness can't be tracked, and reuse any existing map or
     build once.
3. If the map is missing/corrupt/stale/empty (`stops` is empty)/`--refresh` →
   **build it** (Phases 1–3).

## Building the map — three-phase fan-out

Prompt templates:
`${CLAUDE_PLUGIN_ROOT}/skills/guided-tour/references/subagent-prompts.md`.

**Phase 1 — Recon (blocking).** Dispatch ONE `Explore` subagent with the Phase 1
template. It returns `project`, up to 8 `subsystems`, and a candidate
`endToEndTrace`. Tell the user you're surveying the codebase.

**Phase 2 — Deep-dive fan-out (parallel).** Dispatch ONE subagent per subsystem
using the Phase 2 template. **Issue ALL of these Agent tool calls in a SINGLE message so they run concurrently** — do not send them one message at a time, or
you serialize the fan-out and waste time and tokens. Wait for all to settle.

**Phase 3 — Synthesis (you, no subagent).** Build the curriculum:
- One `stop` per subsystem that returned usable findings, `source: "deep-dive"`.
- For any subsystem whose subagent failed or returned nothing usable, write a
  `stub` stop (`source: "stub"`) from what recon knew — do NOT drop it silently.
- Order `stops` into `order[]`: orientation → architecture → the end-to-end
  trace → subsystems, central/simple first; respect any `dependsOn`.
- **Before writing, verify referential integrity:** every id in `order[]` and
  every `dependsOn[]` id exists in `stops[]`.
- Stamp `builtAtSha` (current HEAD, or `"no-git"`) and `builtFileCount`.
- Write `.claude/tour-map.json`. Create `.claude/` if needed.

## Overall tour (`/tour`)

1. Ensure a usable map (load or build, above).
2. Briefly orient: state the project summary and what the tour will cover (the
   stop titles in order) — this is the only "whole map" overview the user sees.
3. Walk `order[]` **one stop at a time**. For each stop, narrate:
   framing → key files & real call sites → the worked example/trace if present →
   gotchas → read-next. Adjust depth to `level`.
   - **Lazy rename detection:** when you reach a stop, confirm its
     `keyFiles[].path` still exist (`git ls-files --error-unmatch <path>` or a
     plain file check). If a path is gone (renamed/deleted since build), tell the
     user the anchor moved and offer `/tour-dive <subsystem> --refresh` rather
     than teaching from a dangling path.
   - **Stub stops:** announce "this area wasn't fully explored yet" and offer
     `/tour-dive <subsystem>` to flesh it out; teach what recon knew.
4. After each stop, stop and let the user drive:
   - `next` — next stop in `order[]`
   - `dig in` — go deeper on the CURRENT stop without leaving the tour (you may
     read the relevant files directly, or re-drill via a subagent if needed)
   - `skip` — skip to the next stop
   - `jump <area>` — jump to the stop for that subsystem
   - `stop` / `done` / `exit` — end the tour
5. On exit, softly suggest testing the new knowledge with
   `/quiz <area>` (from the quiz-master plugin, if installed). Soft suggestion
   only — never block on it.

## Deep dive (`/tour-dive <area>`)

1. Ensure a usable map (load or build).
2. Resolve `<area>` against `stops[].subsystem` / `stops[].title` by fuzzy match.
   - No area given → list the available subsystems and ask which.
   - Ambiguous → ask the user to disambiguate; never guess silently.
3. **Re-drill if thin.** If the matched stop is a `stub`, or its detail is too
   thin for a real lesson, dispatch ONE subagent with the Phase 2b re-drill
   template, then update that stop in the map to `source: "deep-dive"` with the
   richer findings (re-check referential integrity before writing).
4. Teach that subsystem as its own mini-lesson using the same stop structure
   (framing → call sites → worked example → gotchas → read-next), at `level`
   depth, going deeper than the overall tour would.
5. Close by suggesting sibling subsystems, read-next, and `/quiz <area>`.

## Error handling

- **Empty / tiny repo:** if recon finds too little to teach, say so plainly and
  offer to tour whatever genuinely exists — never fabricate subsystems.
- **Subagent failure in fan-out:** proceed with what returned; record the rest
  as stub stops; mention which areas are stubs.
- **Corrupt map:** caught by the integrity check; rebuild.
- **Not a git repo:** see staleness; skip git-based checks gracefully.

## Smoke test (`--smoke`)

If invoked with `--smoke`, do NOT run a tour. Instead self-check and report:
- `${CLAUDE_PLUGIN_ROOT}` resolves and both reference docs exist:
  `${CLAUDE_PLUGIN_ROOT}/skills/guided-tour/references/map-schema.md` and
  `${CLAUDE_PLUGIN_ROOT}/skills/guided-tour/references/subagent-prompts.md`.
- `jq` and `git` are available (`jq --version`, `git --version`).
- The integrity one-liner returns `true` on
  `${CLAUDE_PLUGIN_ROOT}/skills/guided-tour/references/fixtures/map-valid.json`
  and non-zero on
  `${CLAUDE_PLUGIN_ROOT}/skills/guided-tour/references/fixtures/map-truncated.json`.
Report each as pass/fail and stop.
