---
id: S04
parent: M001
milestone: M001
provides:
  - Hot-path callers migrated to DB — dispatch loop no longer parses markdown for planning state
  - Sequence-aware query ordering proven in getMilestoneSlices/getSliceTasks — ORDER BY sequence, id
  - Cross-validation test infrastructure — planning-crossval.test.ts pattern for DB↔rendered↔parsed parity
  - isDbAvailable() + lazy createRequire fallback pattern — reusable for S05 warm/cold caller migration
  - Schema v9 with sequence column on slices and tasks tables
requires:
  - slice: S01
    provides: Schema v8, insertMilestonePlanning/getMilestonePlanning query functions, renderRoadmapFromDb, tool handler pattern
  - slice: S02
    provides: getSliceTasks/getTask query functions, renderPlanFromDb/renderTaskPlanFromDb renderers, slice/task v8 columns populated
affects:
  - S05
  - S06
key_files:
  - src/resources/extensions/gsd/gsd-db.ts
  - src/resources/extensions/gsd/dispatch-guard.ts
  - src/resources/extensions/gsd/auto-dispatch.ts
  - src/resources/extensions/gsd/auto-verification.ts
  - src/resources/extensions/gsd/parallel-eligibility.ts
  - src/resources/extensions/gsd/markdown-renderer.ts
  - src/resources/extensions/gsd/tests/schema-v9-sequence.test.ts
  - src/resources/extensions/gsd/tests/dispatch-guard.test.ts
  - src/resources/extensions/gsd/tests/planning-crossval.test.ts
key_decisions:
  - Used lazy createRequire with .ts/.js extension fallback instead of dynamic import() — keeps hot-path callers synchronous, avoiding cascading async changes (D007)
  - Added sequence column to initial CREATE TABLE DDL in addition to migration block — required for fresh databases that skip migrations
  - Fixed renderRoadmapMarkdown depends serialization from JSON.stringify to join-based — required for parser round-trip parity
  - Kept loadFile in auto-dispatch.ts module imports — still used by 15 other rules for non-planning file content
  - TaskRow.files already parsed as string[] by rowToTask() — no additional JSON.parse needed in consumer code
patterns_established:
  - isDbAvailable() gate + lazy createRequire fallback — standard pattern for migrating synchronous callers from parser to DB queries without breaking call chain signatures
  - Cross-validation test pattern (planning-crossval.test.ts) — DB→render→parse round-trip parity tests for planning artifacts, following derive-state-crossval.test.ts for completion artifacts
  - Sequence-aware query ordering — ORDER BY sequence, id with DEFAULT 0 fallback ensures reassessment reordering propagates through all readers
observability_surfaces:
  - isDbAvailable() gate in 4 migrated files — stderr diagnostic when DB unavailable and fallback to disk parse
  - SQLite slices.sequence and tasks.sequence columns — inspect via SELECT id, sequence FROM slices ORDER BY sequence, id
  - schema-v9-sequence.test.ts — 7 tests covering migration, ordering, defaults
  - dispatch-guard.test.ts — 8 tests with DB seeding (primary DB-path verification)
  - planning-crossval.test.ts — 65 assertions across 3 cross-validation scenarios
  - SCHEMA_VERSION=9 — verify via PRAGMA user_version on DB file
drill_down_paths:
  - .gsd/milestones/M001/slices/S04/tasks/T01-SUMMARY.md
  - .gsd/milestones/M001/slices/S04/tasks/T02-SUMMARY.md
  - .gsd/milestones/M001/slices/S04/tasks/T03-SUMMARY.md
  - .gsd/milestones/M001/slices/S04/tasks/T04-SUMMARY.md
duration: ""
verification_result: passed
completed_at: 2026-03-23T17:21:49.297Z
blocker_discovered: false
---

# S04: Hot-path caller migration + cross-validation tests

**Six hot-path dispatch-loop callers migrated from markdown parsing to DB queries, with 65-assertion cross-validation tests proving DB↔rendered↔parsed parity and schema v9 sequence-aware ordering.**

## What Happened

This slice eliminated markdown parsing from the auto-mode dispatch loop's hottest code paths, replacing 6 parser callers across 4 files with SQLite DB queries.

**T01 — Schema v9 + sequence ordering:** Added `sequence INTEGER DEFAULT 0` to both `slices` and `tasks` tables via a v9 migration block, plus updated initial CREATE TABLE DDL for fresh databases. All 4 slice/task ORDER BY queries changed from `ORDER BY id` to `ORDER BY sequence, id`. Updated `SliceRow`/`TaskRow` interfaces and `insertSlice`/`insertTask` to accept optional sequence params. 7 tests verify migration, ordering, and defaults.

**T02 — dispatch-guard.ts migration:** Replaced `parseRoadmapSlices(roadmapContent)` with `getMilestoneSlices(mid)` behind an `isDbAvailable()` gate. Lazy `createRequire`-based fallback loads parser only when DB is unavailable, keeping the function synchronous (avoiding cascading async changes through loop-deps.ts and phases.ts). All 8 test cases rewritten to seed state via `openDatabase`/`insertMilestone`/`insertSlice` instead of writing ROADMAP markdown. `findMilestoneIds()` still reads disk for milestone queue ordering (out of scope).

**T03 — auto-dispatch.ts, auto-verification.ts, parallel-eligibility.ts migration:** Applied the same `isDbAvailable()` + lazy `createRequire` fallback pattern to the remaining 3 files. In auto-dispatch.ts, migrated 3 rules (uat-verdict-gate, validating-milestone, completing-milestone) from `parseRoadmap().slices` to `getMilestoneSlices(mid)`. In auto-verification.ts, replaced `parsePlan().tasks.find()` with `getTask(mid, sid, tid)?.verify`. In parallel-eligibility.ts, replaced both `parseRoadmap().slices` and `parsePlan().filesLikelyTouched` with DB queries. `loadFile` kept in auto-dispatch.ts for 15 other rules that read non-planning file content.

**T04 — Cross-validation tests + renderer fix:** Created `planning-crossval.test.ts` with 3 test scenarios (65 assertions): ROADMAP round-trip (field parity for id, done/status, depends, risk, title across 4 slices), PLAN round-trip (task count, per-task fields, filesLikelyTouched aggregation), and sequence ordering (scrambled insertion order preserved through full round-trip). Discovered and fixed a depends-quoting bug in `renderRoadmapMarkdown()` — JSON.stringify produced quoted strings that didn't survive parser round-trip. Changed to unquoted join format.

## Verification

**Slice-level verification (all pass):**
1. schema-v9-sequence.test.ts — 7/7 pass (migration, ordering, defaults)
2. dispatch-guard.test.ts — 8/8 pass (DB-seeded dispatch blocking/allowing)
3. planning-crossval.test.ts — 65/65 assertions across 3 scenarios (DB↔rendered↔parsed parity)
4. No module-level parser imports in dispatch-guard.ts, auto-dispatch.ts, auto-verification.ts, parallel-eligibility.ts — verified via grep
5. No module-level parseRoadmap in auto-dispatch.ts — only lazy fallback references
6. getMilestoneSlices('NONEXISTENT') returns [] — graceful empty-state handling

**Regression suites (confirmed passing by task executors):**
- plan-milestone.test.ts — 15/15
- plan-slice.test.ts, plan-task.test.ts — all pass
- integration-mixed-milestones.test.ts — 54/54 (exercises disk-parse fallback)
- markdown-renderer.test.ts — 106/106 (renderer depends fix regression)
- derive-state-crossval.test.ts — 189/189 (renderer fix regression)
- auto-recovery.test.ts — 33/33

## Requirements Advanced

None.

## Requirements Validated

- R009 — dispatch-guard.ts, auto-dispatch.ts (3 rules), auto-verification.ts, parallel-eligibility.ts all migrated to DB queries. Zero module-level parser imports. Tests: dispatch-guard.test.ts 8/8, integration-mixed-milestones.test.ts 54/54.
- R014 — planning-crossval.test.ts — 65 assertions across 3 scenarios proving DB→render→parse round-trip parity for ROADMAP, PLAN, and sequence ordering.
- R016 — Schema v9 adds sequence column. All 4 slice/task ORDER BY queries use ORDER BY sequence, id. schema-v9-sequence.test.ts 7/7 plus cross-validation test 3 proves ordering survives render→parse round-trip.

## New Requirements Surfaced

None.

## Requirements Invalidated or Re-scoped

None.

## Deviations

1. Depends-quoting fix in markdown-renderer.ts (T04): renderRoadmapMarkdown() used JSON.stringify for depends arrays, producing quoted strings that broke parser round-trip. Changed to unquoted join format. This was a genuine parity bug, not scope creep — required for cross-validation tests to pass.

2. Sequence column in CREATE TABLE DDL (T01): Added to initial DDL, not just migration block. Fresh databases skip migrations, so the column must be in the CREATE TABLE statement.

3. createRequire pattern instead of dynamic import() (T02, applied in T03): Kept callers synchronous to avoid cascading async changes through loop-deps.ts, phases.ts, and test mocks. Not planned but architecturally necessary.

## Known Limitations

1. findMilestoneIds() in dispatch-guard.ts still reads milestone directories from disk for queue ordering — DB doesn't own milestone queue discovery. This is acceptable because milestone discovery is a directory scan, not a parser call.

2. Lazy createRequire fallback blocks use the parser at runtime when DB is unavailable. The parsers aren't removed — they're moved from module-level imports to lazy-loaded fallback paths. Full parser removal happens in S06.

3. 15 of 18 auto-dispatch.ts rules still use loadFile for non-planning content (UAT files, context files). These are warm/cold callers, not hot-path planning callers — migrated in S05.

## Follow-ups

None. All remaining work (warm/cold callers, flag files, parser removal) is already planned in S05 and S06.

## Files Created/Modified

- `src/resources/extensions/gsd/gsd-db.ts` — Schema v9 migration (sequence column on slices/tasks), ORDER BY sequence,id in 4 queries, insertSlice/insertTask accept sequence param
- `src/resources/extensions/gsd/dispatch-guard.ts` — Migrated from parseRoadmapSlices to getMilestoneSlices with isDbAvailable gate and lazy createRequire fallback
- `src/resources/extensions/gsd/auto-dispatch.ts` — Migrated 3 rules (uat-verdict-gate, validating-milestone, completing-milestone) from parseRoadmap to getMilestoneSlices with fallback
- `src/resources/extensions/gsd/auto-verification.ts` — Migrated from parsePlan to getTask with isDbAvailable gate and lazy createRequire fallback
- `src/resources/extensions/gsd/parallel-eligibility.ts` — Migrated from parseRoadmap+parsePlan to getMilestoneSlices+getSliceTasks with isDbAvailable gate and lazy fallback
- `src/resources/extensions/gsd/markdown-renderer.ts` — Fixed depends serialization from JSON.stringify to unquoted join for parser round-trip parity
- `src/resources/extensions/gsd/tests/schema-v9-sequence.test.ts` — New: 7 tests for schema v9 migration, sequence ordering, defaults
- `src/resources/extensions/gsd/tests/dispatch-guard.test.ts` — Rewritten: 8 tests now seed state via DB instead of writing ROADMAP markdown files
- `src/resources/extensions/gsd/tests/planning-crossval.test.ts` — New: 65 assertions across 3 cross-validation scenarios proving DB↔rendered↔parsed parity
