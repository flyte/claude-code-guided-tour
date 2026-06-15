# claude-code-guided-tour Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin that teaches an unfamiliar codebase via an interactive, subagent-explored guided tour (`/tour`) with per-subsystem deep dives (`/tour-dive <area>`), partner to `claude-code-quiz-master`.

**Architecture:** A prompt-layer plugin — a thin `.claude-plugin/plugin.json` manifest, two thin slash commands that invoke one `guided-tour` skill, and reference docs (map schema + subagent prompt templates) the skill draws on. The skill orchestrates a three-phase subagent fan-out (recon → parallel deep-dive → synthesis) into a cached curriculum (`.claude/tour-map.json`) and narrates it one stop at a time. No helper CLI; the only deterministic operations are `git` and `jq` one-liners run inline.

**Tech Stack:** Markdown (skill + commands), JSON (manifest + map), `bash` + `git` + `jq` (deterministic ops). No build step, no runtime dependencies.

**Testing note:** There is no compiled code to unit-test. "Tests" here are verification steps on the real deterministic surface: `jq` validation of JSON files, slash-command frontmatter parsing, `grep` checks that SKILL.md contains its load-bearing sections, and a fixture-based red/green check of the map-integrity `jq` one-liner. The final task is a manual smoke run against a real repo.

**Reference — spec:** `docs/superpowers/specs/2026-06-15-claude-code-guided-tour-design.md`

---

## File Structure

```
claude-code-guided-tour/
  .claude-plugin/plugin.json                       # manifest (Task 1)
  LICENSE                                           # MIT (Task 1)
  .gitignore                                        # ignore the cached map (Task 1)
  skills/guided-tour/references/map-schema.md       # tour-map.json schema (Task 2)
  skills/guided-tour/references/subagent-prompts.md # recon + deep-dive templates (Task 3)
  skills/guided-tour/references/fixtures/
      map-valid.json                                # fixture: well-formed map (Task 4)
      map-truncated.json                            # fixture: corrupt map (Task 4)
  skills/guided-tour/SKILL.md                       # the brain (Task 4)
  commands/tour.md                                  # /tour (Task 5)
  commands/tour-dive.md                             # /tour-dive <area> (Task 5)
  README.md                                         # user docs (Task 6)
```

Marketplace registration in the separate `../claude-plugins` repo happens in Task 7.

Responsibility split: the **commands** only route to the skill; the **skill** owns all behaviour; the **references** hold bulky, stable content (schema, prompt templates) so SKILL.md stays focused and the prompt templates are reusable verbatim.

---

### Task 1: Plugin scaffold (manifest, license, gitignore)

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `LICENSE`
- Create: `.gitignore`

- [ ] **Step 1: Write the manifest**

Create `.claude-plugin/plugin.json`:

```json
{
  "name": "claude-code-guided-tour",
  "version": "0.1.0",
  "description": "Teach yourself (or an agent) an unfamiliar codebase via an interactive, subagent-explored guided tour. Partner to claude-code-quiz-master.",
  "author": {
    "name": "flyte",
    "email": "claude-code-guided-tour@failcode.co.uk"
  },
  "homepage": "https://github.com/flyte/claude-code-guided-tour",
  "license": "MIT"
}
```

- [ ] **Step 2: Verify the manifest is valid JSON with the required keys**

Run:
```bash
jq -e 'has("name") and has("version") and has("description")' .claude-plugin/plugin.json
```
Expected: prints `true` and exits 0. (If `jq` errors, the JSON is malformed — fix it.)

- [ ] **Step 3: Write the LICENSE**

Create `LICENSE` (MIT, matching the sibling plugin's copyright holder):

```
MIT License

Copyright (c) 2026 Ellis Percival

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 4: Write .gitignore**

Create `.gitignore`:

```
.claude/tour-map.json
.DS_Store
*.log
```

- [ ] **Step 5: Commit**

```bash
git add .claude-plugin/plugin.json LICENSE .gitignore
git commit -m "feat: scaffold plugin manifest, license, gitignore"
```

---

### Task 2: Map schema reference

**Files:**
- Create: `skills/guided-tour/references/map-schema.md`

- [ ] **Step 1: Write the schema reference**

Create `skills/guided-tour/references/map-schema.md`:

````markdown
# `tour-map.json` schema

Project-relative cache at `.claude/tour-map.json`. It is the **curriculum** plus
the metadata for staleness checks and level-awareness. The skill narrates *from*
this file; it is never pasted to the user.

```jsonc
{
  "schemaVersion": 1,
  "builtAtSha": "<git HEAD sha when built>",
  "builtFileCount": 0,            // `git ls-files | wc -l` at build time, for drift
  "level": "intermediate",        // "beginner" | "intermediate" | "expert"
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
      "dependsOn": ["other-stop-id"],
      "source": "deep-dive"        // "deep-dive" | "stub"
    }
  ],
  "order": ["stop-id", "..."]      // sequenced curriculum; every id MUST exist in stops[]
}
```

## Field rules

- **`source`** — `"deep-dive"` when a Phase-2 subagent returned real findings;
  `"stub"` when the subsystem was found in recon but its deep-dive failed or
  returned nothing usable. A stub holds only what recon knew and is surfaced to
  the user as "not fully explored yet."
- **Referential integrity** — every id in `order[]` and in any `dependsOn[]`
  MUST exist in `stops[]`. Enforce this before writing the file.
- **Ordering** — `order[]` runs orientation → architecture → the end-to-end
  trace → subsystems, central/simple first; respect `dependsOn[]`.

## Integrity check (run before trusting a loaded map)

```bash
jq -e 'has("schemaVersion") and (.schemaVersion == 1) and has("stops") and has("order")' .claude/tour-map.json
```

Exit 0 / `true` → usable. Non-zero (corrupt, truncated, or wrong version) →
treat the map as **missing** and rebuild.
````

- [ ] **Step 2: Verify the embedded integrity command is itself valid jq**

Run (compiles the filter without needing a file):
```bash
echo '{}' | jq -e 'has("schemaVersion") and (.schemaVersion == 1) and has("stops") and has("order")'; echo "exit=$?"
```
Expected: prints `false` then `exit=1` (the filter compiles and correctly rejects an empty object). A `jq: error (compile)` message means the filter in the doc is wrong — fix it.

- [ ] **Step 3: Commit**

```bash
git add skills/guided-tour/references/map-schema.md
git commit -m "docs: add tour-map.json schema reference"
```

---

### Task 3: Subagent prompt templates reference

**Files:**
- Create: `skills/guided-tour/references/subagent-prompts.md`

- [ ] **Step 1: Write the prompt templates**

Create `skills/guided-tour/references/subagent-prompts.md`:

````markdown
# Subagent prompt templates

The skill fills the `<...>` placeholders before dispatch. Keep the
evidence-grounding and structured-return instructions verbatim — they are
load-bearing.

## Phase 1 — Recon (1 subagent, `Explore` type, blocking)

```
You are surveying an unfamiliar codebase to build a table of contents for a
teaching tour. Do NOT teach or explain yet — just map the terrain.

Return a structured report with:
1. project: { summary (one paragraph: what this is and why it exists),
   languages[], buildSystem, entryPoints[] (the files where execution begins) }.
2. subsystems: a list of the MAJOR subsystems, CAPPED AT 8. If you find more,
   merge the smallest/most-peripheral into broader groupings or drop them —
   never exceed 8. For each: { name, oneLinepurpose, representativePaths[] }.
3. endToEndTrace: ONE representative end-to-end flow a newcomer should follow to
   understand how the system works (e.g. an incoming request, a CLI invocation,
   a build). Give it a name and the ordered steps you can identify so far, each
   { description, file, symbol }.

Be concrete: cite real paths. If the repo is too small/empty to have distinct
subsystems, say so and return whatever structure genuinely exists.
```

## Phase 2 — Deep-dive (N subagents, one per subsystem, dispatched together)

```
You are studying ONE subsystem of a codebase so it can be taught to a newcomer.
Subsystem: <name>
Representative paths from recon: <representativePaths>

Investigate, then return structured findings:
- purpose: what this subsystem does and the role it plays in the whole.
- keyFiles: [{ path, role }] — the files someone must read to understand it.
- callSites: [{ path, symbol, note }] — REAL usages, not just definitions
  (concrete usage teaches better than signatures).
- dataFlow: how data moves through this subsystem.
- workedExample: a specific flow within this subsystem worth tracing, if one
  exists.
- gotchas: pitfalls to know before changing this code.
- why: the design rationale.

EVIDENCE RULE — every "why" / rationale claim MUST be backed by concrete
evidence you actually gathered:
  - `git log --follow <file>` and `git log -S "<symbol>" -- <path>` (pickaxe)
    to find when and why code was introduced,
  - `git blame -L <range> <file>` for line-level history,
  - quoting the code comment, commit message, or doc you relied on.
If you cannot find evidence for a rationale, write "rationale unknown" — do NOT
invent one. False "why" answers poison the learner's mental model.

If you cannot meaningfully investigate this subsystem, say so explicitly and
briefly — the orchestrator will record it as a stub.
```

## Phase 2b — Re-drill (1 subagent, used by `/tour-dive` on a thin/stub stop)

```
Use the Phase 2 deep-dive prompt above for subsystem <name>, but go DEEPER than
an overview: trace the workedExample step by step with exact symbols, expand
callSites, and surface non-obvious gotchas. The reader has already had the
high-level tour and now wants real depth.
```
````

- [ ] **Step 2: Verify the file contains the three template headers**

Run:
```bash
grep -c -E '^## Phase (1|2|2b)' skills/guided-tour/references/subagent-prompts.md
```
Expected: prints `3`.

- [ ] **Step 3: Verify the evidence rule and pickaxe are present (the load-bearing bit)**

Run:
```bash
grep -q 'log -S' skills/guided-tour/references/subagent-prompts.md && grep -q 'rationale unknown' skills/guided-tour/references/subagent-prompts.md && echo OK
```
Expected: prints `OK`.

- [ ] **Step 4: Commit**

```bash
git add skills/guided-tour/references/subagent-prompts.md
git commit -m "docs: add recon and deep-dive subagent prompt templates"
```

---

### Task 4: The `guided-tour` skill + map fixtures

**Files:**
- Create: `skills/guided-tour/references/fixtures/map-valid.json`
- Create: `skills/guided-tour/references/fixtures/map-truncated.json`
- Create: `skills/guided-tour/SKILL.md`

- [ ] **Step 1: Write a valid map fixture**

Create `skills/guided-tour/references/fixtures/map-valid.json`:

```json
{
  "schemaVersion": 1,
  "builtAtSha": "0000000000000000000000000000000000000000",
  "builtFileCount": 3,
  "level": "intermediate",
  "project": {
    "summary": "Sample project for fixture testing.",
    "languages": ["markdown"],
    "buildSystem": "none",
    "entryPoints": ["README.md"]
  },
  "endToEndTrace": {
    "name": "sample flow",
    "steps": [{ "description": "start", "file": "README.md", "symbol": "n/a" }]
  },
  "stops": [
    {
      "id": "overview",
      "title": "Overview",
      "subsystem": "root",
      "objective": "Understand the sample.",
      "framing": "Think of this as a fixture.",
      "keyFiles": [{ "path": "README.md", "role": "entry" }],
      "callSites": [],
      "workedExample": "",
      "gotchas": [],
      "readNext": [],
      "dependsOn": [],
      "source": "deep-dive"
    }
  ],
  "order": ["overview"]
}
```

- [ ] **Step 2: Write a truncated (corrupt) map fixture**

Create `skills/guided-tour/references/fixtures/map-truncated.json` (deliberately invalid JSON — simulates a write cut off mid-synthesis):

```
{
  "schemaVersion": 1,
  "builtAtSha": "0000",
  "stops": [
    { "id": "overview", "tit
```

- [ ] **Step 3: Verify the integrity one-liner passes on valid and fails on truncated (red/green for the deterministic surface)**

Run:
```bash
jq -e 'has("schemaVersion") and (.schemaVersion == 1) and has("stops") and has("order")' skills/guided-tour/references/fixtures/map-valid.json; echo "valid_exit=$?"
jq -e 'has("schemaVersion") and (.schemaVersion == 1) and has("stops") and has("order")' skills/guided-tour/references/fixtures/map-truncated.json 2>/dev/null; echo "truncated_exit=$?"
```
Expected: `valid_exit=0` (prints `true` first) and `truncated_exit` is non-zero (`4` from jq's parse error, or similar). This proves the load-time integrity guard works.

- [ ] **Step 4: Write the skill**

Create `skills/guided-tour/SKILL.md`:

````markdown
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
- `--set-level <level>` → write `level` into the map and exit (rebuild only if no
  map exists).
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
   - **Rebuild** when the sha differs AND file-count drift exceeds ~15% (or HEAD
     is unreachable). Otherwise reuse the map — teaching tolerates mild staleness.
   - If this is **not a git repo** (`git rev-parse` fails), skip staleness, note
     to the user that staleness can't be tracked, and reuse any existing map or
     build once.
3. If the map is missing/corrupt/stale/`--refresh` → **build it** (Phases 1–3).

## Building the map — three-phase fan-out

Prompt templates:
`${CLAUDE_PLUGIN_ROOT}/skills/guided-tour/references/subagent-prompts.md`.

**Phase 1 — Recon (blocking).** Dispatch ONE `Explore` subagent with the Phase 1
template. It returns `project`, up to 8 `subsystems`, and a candidate
`endToEndTrace`. Tell the user you're surveying the codebase.

**Phase 2 — Deep-dive fan-out (parallel).** Dispatch ONE subagent per subsystem
using the Phase 2 template. **Issue ALL of these Agent tool calls in a SINGLE
message so they run concurrently** — do not send them one message at a time, or
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
- `${CLAUDE_PLUGIN_ROOT}` resolves and the two reference files exist.
- `jq` and `git` are available (`jq --version`, `git --version`).
- The integrity one-liner returns `true` on
  `references/fixtures/map-valid.json` and non-zero on
  `references/fixtures/map-truncated.json`.
Report each as pass/fail and stop.
````

- [ ] **Step 5: Verify the skill frontmatter parses and has name + description**

Run:
```bash
awk 'NR==1&&$0=="---"{f=1;next} f&&$0=="---"{exit} f{print}' skills/guided-tour/SKILL.md | grep -E '^(name|description):'
```
Expected: prints both a `name:` line (`name: guided-tour`) and a `description:` line.

- [ ] **Step 6: Verify the skill contains its load-bearing sections**

Run:
```bash
for h in "Building the map" "Overall tour" "Deep dive" "SINGLE message" "Lazy rename" "Smoke test"; do
  grep -q "$h" skills/guided-tour/SKILL.md && echo "OK: $h" || echo "MISSING: $h"
done
```
Expected: every line prints `OK:`. Any `MISSING:` means a required section/phrase was dropped — add it.

- [ ] **Step 7: Commit**

```bash
git add skills/guided-tour/SKILL.md skills/guided-tour/references/fixtures/
git commit -m "feat: add guided-tour skill and map fixtures"
```

---

### Task 5: Slash commands

**Files:**
- Create: `commands/tour.md`
- Create: `commands/tour-dive.md`

- [ ] **Step 1: Write the `/tour` command**

Create `commands/tour.md`:

```markdown
---
description: Take an interactive guided tour that teaches you this codebase
argument-hint: "[--level <level>] [--set-level <level>] [--refresh] [--smoke]"
---

You have been invoked via the `/tour` slash command.

Invoke the `guided-tour` skill immediately. You are running the **overall tour**
(the whole-codebase lesson). Pass any user arguments through to the skill.

User arguments: $ARGUMENTS
```

- [ ] **Step 2: Write the `/tour-dive` command**

Create `commands/tour-dive.md`:

```markdown
---
description: Take a deep-dive lesson into one area of this codebase
argument-hint: "<area> [--level <level>] [--refresh]"
---

You have been invoked via the `/tour-dive` slash command.

Invoke the `guided-tour` skill immediately. You are running a **deep dive** into
a single subsystem. The first positional argument is the target area. Pass all
arguments through to the skill.

User arguments: $ARGUMENTS
```

- [ ] **Step 3: Verify both commands have valid frontmatter with a description**

Run:
```bash
for f in commands/tour.md commands/tour-dive.md; do
  awk 'NR==1&&$0=="---"{f=1;next} f&&$0=="---"{exit} f{print}' "$f" | grep -q '^description:' && echo "OK: $f" || echo "BAD: $f"
done
```
Expected: both print `OK:`.

- [ ] **Step 4: Verify each command routes to the skill and forwards arguments**

Run:
```bash
grep -q 'guided-tour' commands/tour.md && grep -q '\$ARGUMENTS' commands/tour.md \
  && grep -q 'guided-tour' commands/tour-dive.md && grep -q '\$ARGUMENTS' commands/tour-dive.md \
  && echo OK
```
Expected: prints `OK`.

- [ ] **Step 5: Commit**

```bash
git add commands/tour.md commands/tour-dive.md
git commit -m "feat: add /tour and /tour-dive slash commands"
```

---

### Task 6: README

**Files:**
- Create: `README.md`

- [ ] **Step 1: Write the README**

Create `README.md`:

```markdown
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

## How it works

The first run explores the codebase and caches a **curriculum** at
`.claude/tour-map.json` (an ordered set of teaching "stops"). Later runs reuse
it and only re-explore when the codebase has changed meaningfully, so they're
fast. The cache is git-ignored.

## Requirements

`git` and `jq` on PATH. Staleness tracking needs a git repo; outside one the
tour still works but won't auto-refresh.

## License

MIT
```

- [ ] **Step 2: Verify the README documents both commands**

Run:
```bash
grep -q '/tour-dive' README.md && grep -q '/tour\b' README.md && echo OK
```
Expected: prints `OK`.

- [ ] **Step 3: Commit**

```bash
git add README.md
git commit -m "docs: add README"
```

---

### Task 7: Register in the marketplace

**Files:**
- Modify: `../claude-plugins/.claude-plugin/marketplace.json`
- Modify: `../claude-plugins/README.md`

This repo (`claude-code-guided-tour`) and the marketplace repo
(`../claude-plugins`) are separate git repositories. This task edits the
marketplace repo.

- [ ] **Step 1: Add the plugin to marketplace.json**

In `../claude-plugins/.claude-plugin/marketplace.json`, add this object to the
`plugins` array (after the existing `claude-code-quiz-master` entry, comma-separated):

```json
{
  "name": "claude-code-guided-tour",
  "description": "Teach yourself (or an agent) an unfamiliar codebase via an interactive, subagent-explored guided tour. Partner to claude-code-quiz-master.",
  "source": {
    "source": "url",
    "url": "https://github.com/flyte/claude-code-guided-tour.git"
  },
  "homepage": "https://github.com/flyte/claude-code-guided-tour"
}
```

- [ ] **Step 2: Verify the marketplace JSON is still valid and lists the plugin**

Run:
```bash
jq -e '.plugins[] | select(.name=="claude-code-guided-tour")' ../claude-plugins/.claude-plugin/marketplace.json
```
Expected: prints the new plugin object and exits 0. (A `jq` parse error means the edit broke the JSON — fix the comma/braces.)

- [ ] **Step 3: Add a row to the marketplace README table**

In `../claude-plugins/README.md`, add this row to the "Available plugins" table,
below the `claude-code-quiz-master` row:

```markdown
| [`claude-code-guided-tour`](https://github.com/flyte/claude-code-guided-tour) | Interactive guided tour that teaches you an unfamiliar codebase. Partner to the quiz master. |
```

- [ ] **Step 4: Commit the marketplace repo**

```bash
cd ../claude-plugins
git add .claude-plugin/marketplace.json README.md
git commit -m "feat: add claude-code-guided-tour to marketplace"
cd -
```

---

### Task 8: Manual smoke run

No new files. Validates the assembled plugin end to end. Because the behaviour
is model-driven, this is a manual checklist, not an automated test.

- [ ] **Step 1: Structural sanity across the whole plugin**

Run from the plugin root:
```bash
jq -e '.name=="claude-code-guided-tour"' .claude-plugin/plugin.json \
  && test -f skills/guided-tour/SKILL.md \
  && test -f skills/guided-tour/references/map-schema.md \
  && test -f skills/guided-tour/references/subagent-prompts.md \
  && test -f commands/tour.md && test -f commands/tour-dive.md \
  && echo "STRUCTURE OK"
```
Expected: prints `STRUCTURE OK`.

- [ ] **Step 2: Install the plugin locally and run the skill smoke test**

In a Claude Code session with the plugin installed, run:
```
/tour --smoke
```
Expected: the skill reports pass for `${CLAUDE_PLUGIN_ROOT}` resolution, both
reference files present, `jq`/`git` available, and the integrity one-liner
passing on `map-valid.json` / failing on `map-truncated.json`. Fix anything
reported as fail.

- [ ] **Step 3: Run a real overall tour**

In a session opened on a non-trivial repo (e.g. the sibling
`claude-code-quiz-master`), run:
```
/tour
```
Verify: it surveys, builds `.claude/tour-map.json`, then presents an **overview
followed by one stop at a time** (not a wall of text), starting high-level, with
at least one end-to-end trace. Confirm `next` advances and `stop` exits.

- [ ] **Step 4: Verify the cache and a deep dive**

```bash
jq -e 'has("stops") and has("order")' .claude/tour-map.json && echo "MAP OK"
```
Expected: prints `MAP OK`. Then in-session run `/tour-dive <one subsystem name>`
and confirm it teaches that area in more depth than the overview did, and that
the second invocation reuses the cache (no full re-exploration).

- [ ] **Step 5: Final commit (if any fixes were made during smoke)**

```bash
git add -A
git commit -m "fix: smoke-test corrections" || echo "nothing to commit"
```

---

## Notes for the implementer

- `${CLAUDE_PLUGIN_ROOT}` is set by Claude Code at runtime to the installed
  plugin directory; reference files are addressed through it from inside the skill.
- The `.claude/tour-map.json` cache is project-relative (the repo being toured),
  NOT inside the plugin — and it is git-ignored in this repo via Task 1.
- Keep the commands thin: all behaviour lives in the skill, mirroring
  quiz-master's `/quiz` → `quiz-master` pattern.
- Tasks 1–6 commit to THIS repo; Task 7 commits to the separate
  `../claude-plugins` repo.
```
