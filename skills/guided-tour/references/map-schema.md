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
