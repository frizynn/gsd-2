---
id: T04
parent: S04
milestone: M001
key_files:
  - src/resources/extensions/gsd/tests/planning-crossval.test.ts
  - src/resources/extensions/gsd/markdown-renderer.ts
  - .gsd/milestones/M001/slices/S04/tasks/T04-PLAN.md
key_decisions:
  - Fixed renderRoadmapMarkdown depends serialization from JSON.stringify (quoted) to join-based (unquoted) — required for parser round-trip parity since parseRoadmapSlices doesn't strip quotes from dependency IDs
observability_surfaces:
  - "planning-crossval.test.ts — 65 assertions across 3 scenarios (ROADMAP parity, PLAN parity, sequence ordering)"
  - "Cross-validation pattern follows derive-state-crossval.test.ts established in prior work"
duration: ""
verification_result: passed
completed_at: 2026-03-23T17:15:58.443Z
blocker_discovered: false
---

# T04: Add planning-crossval tests proving DB↔rendered↔parsed parity and fix renderer depends quoting

**Add planning-crossval tests proving DB↔rendered↔parsed parity and fix renderer depends quoting**

## What Happened

Created `planning-crossval.test.ts` with 3 test scenarios (65 assertions) proving DB→render→parse round-trip parity for planning data:

**Test 1: ROADMAP round-trip parity** — Seeds 4 slices with varied status (2 complete, 2 pending), depends arrays, risk levels, and demo strings. Renders via `renderRoadmapFromDb()`, parses back via `parseRoadmapSlices()`, asserts field-by-field parity for id, title, done↔status, risk, and depends.

**Test 2: PLAN round-trip parity** — Seeds 1 slice with 3 tasks having planning fields (description, files arrays, verify commands, estimates). Renders via `renderPlanFromDb()`, parses back via `parsePlan()`, asserts task count, per-task field parity (id, title, verify, done↔status, files), filesLikelyTouched aggregation, and sequence ordering.

**Test 3: Sequence ordering parity** — Seeds 4 slices inserted in scrambled order (seq 3,1,4,2). Verifies DB query returns sequence order, render produces slices in sequence order, and parsed-back slices preserve that order through the full round-trip.

**Renderer fix:** Discovered and fixed a parity bug in `renderRoadmapMarkdown()` — it used `JSON.stringify()` for the depends array, producing `depends:["S01","S02"]` with quoted strings. The parser doesn't strip quotes, so round-trip produces `['"S01"', '"S02"']` instead of `['S01', 'S02']`. Changed to `[${deps.join(",")}]` to produce `depends:[S01,S02]` matching the parser's expected format. All 106 existing renderer tests and 189 derive-state-crossval assertions pass with this fix.

## Verification

1. `planning-crossval.test.ts` — 65/65 assertions pass across 3 scenarios (149ms).
2. `schema-v9-sequence.test.ts` — 7/7 pass (T01 regression).
3. `dispatch-guard.test.ts` — 8/8 pass (T02 regression).
4. `markdown-renderer.test.ts` — 106/106 pass (renderer fix regression).
5. `derive-state-crossval.test.ts` — 189/189 pass (renderer fix regression).
6. `auto-recovery.test.ts` — 33/33 pass (renderPlanFromDb regression).
7. Diagnostic: `getMilestoneSlices('NONEXISTENT')` returns `[]` (no crash).

## Verification Evidence

| # | Command | Exit Code | Verdict | Duration |
|---|---------|-----------|---------|----------|
| 1 | `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types --test src/resources/extensions/gsd/tests/planning-crossval.test.ts` | 0 | ✅ pass — 65/65 assertions across 3 scenarios | 153ms |
| 2 | `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types --test src/resources/extensions/gsd/tests/schema-v9-sequence.test.ts` | 0 | ✅ pass — 7/7 | 135ms |
| 3 | `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types --test src/resources/extensions/gsd/tests/dispatch-guard.test.ts` | 0 | ✅ pass — 8/8 | 543ms |
| 4 | `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types --test src/resources/extensions/gsd/tests/markdown-renderer.test.ts` | 0 | ✅ pass — 106/106 | 192ms |
| 5 | `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types --test src/resources/extensions/gsd/tests/derive-state-crossval.test.ts` | 0 | ✅ pass — 189/189 | 527ms |
| 6 | `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types --test src/resources/extensions/gsd/tests/auto-recovery.test.ts` | 0 | ✅ pass — 33/33 | 627ms |
| 7 | `grep parseRoadmapSlices|parseRoadmap|parsePlan dispatch-guard.ts auto-verification.ts parallel-eligibility.ts` | 0 | ✅ pass — only lazy-loader references, no module-level imports | 5ms |
| 8 | `node --import resolve-ts.mjs --experimental-strip-types -e getMilestoneSlices(NONEXISTENT) diagnostic` | 0 | ✅ pass — returns [] | 200ms |


## Deviations

Fixed a depends-quoting bug in `renderRoadmapMarkdown()` in `markdown-renderer.ts` — the renderer used `JSON.stringify()` for the depends array, which produced quoted strings `["S01"]` that didn't round-trip through the parser. Changed to `[S01]` format. This was required to make Test 1 pass and is a genuine parity fix, not scope creep.

## Known Issues

None.

## Diagnostics

- Run cross-validation tests: `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types --test src/resources/extensions/gsd/tests/planning-crossval.test.ts`
- Verify renderer fix: `grep 'join.*","' src/resources/extensions/gsd/markdown-renderer.ts` — depends serialization should use `.join(",")` not `JSON.stringify`
- Run renderer regression: `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types --test src/resources/extensions/gsd/tests/markdown-renderer.test.ts`

## Files Created/Modified

- `src/resources/extensions/gsd/tests/planning-crossval.test.ts`
- `src/resources/extensions/gsd/markdown-renderer.ts`
- `.gsd/milestones/M001/slices/S04/tasks/T04-PLAN.md`
