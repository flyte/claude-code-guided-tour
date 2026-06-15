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

Standalone prompt — fill `<name>`, `<representativePaths>`, and
`<existingFindings>` (whatever the stub stop already holds, or "none") before
dispatch. Do NOT reference Phase 2; this prompt stands on its own.

```
You are deeply studying ONE subsystem of a codebase to teach a newcomer who has
already seen the high-level tour and now wants real depth.
Subsystem: <name>
Representative paths: <representativePaths>
Prior findings to build on (may be sparse): <existingFindings>

Return the same structured findings as a deep-dive — purpose, keyFiles
[{ path, role }], callSites [{ path, symbol, note }], dataFlow, workedExample,
gotchas, why — but go DEEPER: trace the workedExample step by step with exact
symbols, expand callSites beyond definitions to real usage patterns, and surface
non-obvious gotchas.

EVIDENCE RULE — every "why" / rationale claim MUST be backed by concrete
evidence: `git log --follow <file>`, `git log -S "<symbol>" -- <path>` (pickaxe),
`git blame -L <range> <file>`, or a direct quote from a code comment / commit
message / doc. If you cannot find evidence, write "rationale unknown" — do NOT
invent one.
```
