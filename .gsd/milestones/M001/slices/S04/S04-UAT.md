# S04: Hot-path caller migration + cross-validation tests — UAT

**Milestone:** M001
**Written:** 2026-03-23T17:21:49.297Z

# S04: Hot-path caller migration + cross-validation tests — UAT

**Milestone:** M001
**Written:** 2026-03-23

## UAT Type

- UAT mode: artifact-driven
- Why this mode is sufficient: All verification is through automated tests (DB queries, parser comparison, grep for imports) — no runtime behavior or human-facing UI to test

## Preconditions

- Working directory is the gsd-2 repo root
- Node.js with `--experimental-strip-types` support available
- No running DB connections (tests use in-memory SQLite)

## Smoke Test

Run `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types --test src/resources/extensions/gsd/tests/planning-crossval.test.ts` and verify 65/65 assertions pass across 3 scenarios. This single test proves the core deliverable: DB state survives render→parse round-trip.

## Test Cases

### 1. Schema v9 sequence ordering

1. Run `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types --test src/resources/extensions/gsd/tests/schema-v9-sequence.test.ts`
2. **Expected:** 7/7 tests pass covering migration, sequence-based ordering for slices and tasks, default fallback, and active-slice/task resolution

### 2. Dispatch guard DB migration

1. Run `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types --test src/resources/extensions/gsd/tests/dispatch-guard.test.ts`
2. **Expected:** 8/8 tests pass with DB-seeded state (not markdown files)

### 3. Cross-validation parity

1. Run `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types --test src/resources/extensions/gsd/tests/planning-crossval.test.ts`
2. **Expected:** 65/65 assertions pass across 3 scenarios (ROADMAP parity, PLAN parity, sequence ordering parity)

### 4. No module-level parser imports in migrated files

1. Run `grep -n '^import.*parseRoadmapSlices\|^import.*parseRoadmap\|^import.*parsePlan' src/resources/extensions/gsd/dispatch-guard.ts src/resources/extensions/gsd/auto-dispatch.ts src/resources/extensions/gsd/auto-verification.ts src/resources/extensions/gsd/parallel-eligibility.ts`
2. **Expected:** No output (exit code 1) — zero module-level parser imports

### 5. Disk-parse fallback path

1. Run `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types --test src/resources/extensions/gsd/tests/integration-mixed-milestones.test.ts`
2. **Expected:** 54/54 pass — these tests don't seed DB, so they exercise the lazy createRequire disk-parse fallback

### 6. Renderer regression after depends fix

1. Run `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types --test src/resources/extensions/gsd/tests/markdown-renderer.test.ts`
2. **Expected:** 106/106 pass — depends serialization change doesn't break existing rendering

## Edge Cases

### Empty milestone (no slices in DB)

1. Run `node --import ./src/resources/extensions/gsd/tests/resolve-ts.mjs --experimental-strip-types -e "import{openDatabase,getMilestoneSlices}from'./src/resources/extensions/gsd/gsd-db.ts';openDatabase(':memory:');console.log(JSON.stringify(getMilestoneSlices('NONEXISTENT')))"`
2. **Expected:** Outputs `[]` — no crash, graceful empty-state handling

### Sequence defaults to 0

1. In schema-v9-sequence.test.ts, test "sequence field defaults to 0 when not provided" verifies that slices/tasks inserted without explicit sequence get `sequence: 0`
2. **Expected:** Passes — backward compatible with pre-v9 data

## Failure Signals

- Any module-level `import ... parseRoadmap` or `import ... parsePlan` in the 4 migrated files
- planning-crossval.test.ts assertion failures indicating field mismatch between DB and parsed-back state
- dispatch-guard.test.ts failures indicating DB seeding doesn't produce correct blocking behavior
- integration-mixed-milestones.test.ts failures indicating broken disk-parse fallback

## Requirements Proved By This UAT

- R009 — All 6 hot-path parser callers migrated to DB queries (test cases 1-5)
- R014 — Cross-validation tests prove DB↔rendered↔parsed parity (test case 3)
- R016 — Sequence-aware ordering in all queries (test cases 1, 3)

## Not Proven By This UAT

- Live auto-mode runtime behavior (auto-dispatch rules exercised via integration tests, not live dispatch loop)
- S05 warm/cold callers (doctor, visualizer, github-sync, etc.)
- S06 parser removal from hot paths
- Flag file migration (CONTINUE, CONTEXT-DRAFT, etc.)

## Notes for Tester

- All tests use in-memory SQLite — no persistent DB files to clean up
- The lazy createRequire fallback references will still match grep for parser names in function bodies — this is intentional; only module-level imports should be absent
- `loadFile` remains in auto-dispatch.ts module imports — it's used by 15 non-planning rules and is not a parser caller
