# 07 · Field-Test Plan — Verification Mode
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/376995878/07+Field-Test+Plan+Verification+Mode
> pageId: 376995878 · lastModified: 9월 20, 2026 · saved: 2026-10-12

> **The point of this phase:** every design ratified ahead of validation (D-024 pattern: D-019/D-020/D-021, D-033–D-036, D-038–D-046) now gets proven on real data. Criteria below were fixed **before** the runs — a run can't move its own goalposts. A flaw found here is fixed by a **superseding decision** on [05 · Decision Log — D-001…D-050 (append-only)](./05-decision-log.md), never a silent edit. Each completed test updates this page's status column and the [06 · Case Registry — Domain Packs](./06-case-registry.md) row.

## Test 1 — `sck` instance migration v2.0 → v2.2 (first action, gates everything else)

**RUN COMPLETE **** — 7 of 8 gates green.** `sck` is on **v2.2**. G1–G5, G7 and G8 pass; **G6 is blocked** on a gap between two ratified pages and needs **D-051**. Results are in the section below the gate table.

The loaded `sck` instance (26 STDF files, \~40 GB) is on **v2.0** — read from the instance, not assumed ([D-050 — Full Record: Test 1's oracle and scope superseded by the measured v2.0 baseline (ratified 2026-09-01)](./05-D050-full-record.md), ratified 2026-09-01). It migrates **v2.0 → v2.2 (Block 2 then Block 3; Block 1 is never re-run)**, through the [08 · Operations — Instances, Migrations, Retention, Guards](./08-operations.md) discipline — the first migration run through D-045. The path is **ALTER in place**: `pg_dump` `sck_raw` into the target name, then Block 2, then Block 3, so the SCK domain pack travels with the dump. ── **INVERTED CHECK, 2026-09-01 (D-050 limbs 2 and 4).** This paragraph read *"is on v1.1. It migrates ****once****, v1.1 → v2.2"* until today. The v1.1 instance is a separate stopped container (`universal_db`) that shares no data with `sck_raw`. **Standing up a fresh instance from Block 1+2+3 and restoring into it now FAILS** — `entity.type_code` is FK-constrained to `entity_type` and `link.link_kind` to `link_type`, and none of SCK's thirteen type\_codes nor `MEMBER_OF` is in Block 1's generic seed. Under the old wording that path passed review. It fails late and looks cleaner, which is why it is ruled rather than left to judgement.

| Gate | Criterion | From |
| --- | --- | --- |
| G1 | Count-identical re-verification against the **full 23-key oracle the loader harness already computes** (`expected_counts.json`) — not a hand-listed subset. Coverage: every `entity.type_code` (13) and every `link_kind` (3) present at the pinned pre-migration state, plus `measurement`, `param_def`, `param_spec`, `file_layout_set`, `file_layout`, `file_ref` (lane-filtered `SCALAR_BULK`) and multi-pass dies. Expected values are D-050's measured v2.0 baseline table, cited there rather than restated here. Reconciliation is part of the gate: `entity` = 418,503 (sum of all 13 types), `link` = 211,833 (sum of all 3 kinds), all `link` rows `is_active = t`. ── **INVERTED CHECK, 2026-09-01 (D-050).** This gate hand-listed eleven figures from P-001 §6, two of them stale: `param_def` 3,415 and limits 2,786. The measured values are **3,521** and **2,813**. **A run passing G1 against 3,415 / 2,786 now FAILS; under the old criterion it passed.** The seven entity types and two link kinds the old list omitted (PIN, PIN\_GROUP, TESTER, HANDLER, LOADBOARD, TEST\_PROGRAM, TEST\_PROG\_VER; MEMBER\_OF, TESTED\_ON) did not exist at v1.1 and are in scope now. Note `file_ref` is lane-filtered — an unfiltered `count(*)` agrees only because every current row is `SCALAR_BULK`, and a second lane would diverge. | [D-050 — Full Record: Test 1's oracle and scope superseded by the measured v2.0 baseline (ratified 2026-09-01)](./05-D050-full-record.md), superseding P-001 §6 |
| G2 | Per-die retest-history query returns identical results through `link` | P-001 §6 |
| G3 | Re-send one full STDF file → zero new rows, zero errors | P-002 §6 w2 |
| G4 | Re-send a value-mutated copy → fail-loud reject; then succeeds as CORRECTION batch with old value in `measurement_audit` | P-002 §6 w2 |
| G5 | Synthetic two-recipes-one-run pair inserts cleanly (C-05 recipe half) | P-002 §6 w3 |
| G6 | Two-session A→B / B→A race on an acyclic kind → second session blocks/fails under the advisory lock | P-002 §6 w4 |
| G7 | `schema_migration` shows v2.2 as latest; loader/pack SQL/onboard tooling updated to v2.2 targets | D-045 |
| G8 | Window columns present and NULL everywhere (no data impact) | D-042 |

### Test 1 results — run 2026-09-19

Path taken, exactly as D-050 limb 4 rules: `pg_dump -Fc sck_raw` → restore into the new database `sck` → `001_v2.0_to_v2.1.sql` (Block 2) → `002_v2.1_to_v2.2.sql` (Block 3), each in one transaction. `sck_raw` was never modified and remains the rollback, alongside the dump. Scripts live in `INFRA/sql/udb/migrations/`, and each is the ratified block verbatim — no statement was added to either. The recorded `script_sha256` is `6af0eccbbdb987070c6d82a07c719ba2624775dd8295af0b3c0240a5e821dae2`, the sha256 of the two files concatenated in apply order: `schema_migration.script_sha256` is CHAR(64) and the package is two scripts, so one hash covers both. Per-file hashes sit in `CHECKSUMS.txt` beside them.

| Gate | Verdict | Evidence |
| --- | --- | --- |
| G1 | PASS | All 23 oracle keys identical pre- and post-migration. The two labels `validate.py` computed but never asserted — `disambiguated params (#)` = 211 and `param_def name collisions` = 0 — were added to `expected_counts.json`, so the run is now **25/25 asserted**. That closes the gap D-050 named: *"a guard whose passing value is written nowhere is not a guard."* |
| G2 | PASS | The per-die retest-history spot checks are byte-identical pre vs post — diffed, not eyeballed. Probe die `STWGUX/W15/X085Y040` still returns 4 passes, `rtst=['0','1','1','2']`. |
| G3 | PASS | One STDF re-sent: zero new `entity` / `link` / `measurement` / `param_def` / `param_spec` rows. The run used a scratch Parquet prefix so the 26 live objects were never rewritten; the scratch `file_ref` row and its objects were removed afterwards, leaving `file_ref` at 26. |
| G4 | PASS | A copy with three PRR `HARD_BIN` values mutated, re-sent as `INGEST`, raised: *"I12 violation: re-send changes 3 existing measurement value(s) and batch 2 is kind='INGEST' … A changed value is only legal in a CORRECTION batch"*. The same file as a `CORRECTION` batch succeeded, archiving the three old values to `measurement_audit`. The originals were then restored by a second CORRECTION, and the audit table correctly retains **both** hops. |
| G5 | PASS | Two `PROCESSED_IN` links on one RUN to one TESTER with different `context_entity_id` (two RECIPE\_VERSION entities) both insert cleanly — `uq_link_natural` includes `context_entity_id`, so they are distinct rows, not a conflict. Executed inside a rolled-back transaction to keep the instance count-exact. |
| G6 | BLOCKED | Cannot pass against the ratified DDL. See the section below. |
| G7 | PASS | `schema_migration` returns `v2.2`. The loader was brought to v2.2: it now opens an `ingest_batch` per file and stamps `batch_id` on every `measurement` / `link` / `file_ref` / `param_spec`; `link` upserts target `uq_link_natural`; `param_spec` upserts target the partial `uq_param_spec_active` index; and `measurement` enforces I12 (see G4). Pre-v2.2 rows keep `batch_id = NULL` — deliberately not backfilled, because inventing provenance for rows whose batch was never recorded is worse than the NULLs. |
| G8 | PASS | `link.window_start` / `window_end` present and NULL on all 211,833 rows. |

**Post-migration state.** 18 base tables plus four monthly `observation` partitions (2026-09 … 2026-12). Partition creation is ops config under D-035, so it shipped as a separate labelled step rather than inside the ratified scripts. All four Block 1 guards still fire (6/6 negative tests). New constraints present: `uq_link_natural`, `uq_param_spec_active`, `link_window_ck`, `param_def_conv_pair`, `one_condition_value`, `file_ref_window`, `ix_link_occupancy`. Every nightly backstop query from page 08 returns 0. The Parquet round-trip still matches `row_count`.

### Why G6 is blocked — a gap between two ratified pages

[08 · Operations — Instances, Migrations, Retention, Guards](./08-operations.md) states as operational fact that `link_guard()` takes `pg_advisory_xact_lock(hashtext(NEW.link_kind))` before the cycle walk on acyclic kinds. Block 2 on [01 · Schema Spec — v2.2 (Committed Baseline)](./01-schema-spec-v2.2.md) mentions it only in a trailing **comment**. **No **`CREATE OR REPLACE FUNCTION link_guard()`** carrying the lock exists in Block 1, 2 or 3.** The guard that ships in the ratified DDL does not do what page 08 says it does.

This was measured, not reasoned about. On a throwaway database carrying the ratified guard, two concurrent READ COMMITTED sessions inserting A→B and B→A **both commit**, leaving `1->2` and `2->1` — a cycle on an acyclic kind, which is exactly what the guard exists to prevent. With the advisory lock added, the second session blocks until the first commits and then fails loudly:

```text
ERROR:  link ASSEMBLY would create a cycle (src=2 dst=1)
CONTEXT:  PL/pgSQL function link_guard() line 46 at RAISE
```

The fix is drafted at `INFRA/sql/udb/migrations/003_link_guard_advisory_lock.sql` — the v2.2 guard verbatim with one `PERFORM pg_advisory_xact_lock(...)` added — and is **deliberately not applied**. Changing a ratified guard needs its own ratified decision (next id **D-051**), and it is kept out of 001/002 so their `script_sha256` attests to the ratified text alone. G6 turns green the moment D-051 ratifies; nothing else about it is uncertain.

A second gap of the same shape was found and is logged on [04 · Known Limits & Deferred Items — Read Before Field Testing](./04-known-limits-deferred.md): Block 2 states the I12 correction policy as a POLICY comment with no DDL either. It is implemented loader-side, which matches page 02 calling it the loader contract, but nothing in the database enforces it against any other writer.

## Test 2 — SCK corner-file load (D-038 gate)

Corner-carrying STDF data (currently held out of loads) loads after Test 1. Criteria: (w1) corner-curve query returns one row per corner; (w2) per-corner limits resolve from `param_spec`; (w3) non-corner params and existing counts unaffected.

## Test 3 — LG Innotek load (D-019 / D-036 / D-040 / D-033 gates)

- **Assembly genealogy (D-019):** real parts-list sample → ASSEMBLY links load; full ancestry of a MODULE at depth ≥ 4 correct.
- **Vocabulary (D-036):** map ≥ 20 concepts shared across SCK + LG `param_def` rows; (w1) one concept-keyed query returns unit-consistent values from both sources with zero per-source SQL; (w2) alias resolution: '압력' / 'PRESS' / native id → same `param_id`; (w3) unmapped params provably unaffected.
- **Unit conversion (D-040):** one deliberate unit-mismatch case converts correctly through scale/offset.
- **Graph projection benchmark (D-033/P-005):** (w1) projection traversals ≡ recursive-CTE results row-for-row; (w2) p95 multi-hop latency vs CTE at depth ≥ 4; (w3) full rebuild time at case volume. **The benchmark tunes the projection engine only — the truth-placement verdict (D-033) is not reopened by any result.**

## Test 4 — First standing-asset case (KOGAS pipeline, or SEMES facility telemetry if first)

- **TIMESERIES (D-035/P-003):** (w1) one asset/param over an arbitrary window; (w2) monthly cross-asset aggregate; (w3) sustained ingest ≥ 2× source cadence; (w4) storage per 10⁹ points. Judged on p95 latency + ingest headroom, vanilla partitions vs TimescaleDB on identical runs — **TimescaleDB only if the TSL license check passes for on-prem appliances (D-026); check runs before the benchmark.**
- **Windows & attribution (D-042):** re-run the three exploration benchmarks on real data (attribution p95 with the composite GiST; overlap validation in the loader; rebuild/insert cost); guard-band demo on one real boundary case — in-band samples come back AMBIGUOUS.
- **Irregular traces (D-041):** if the case carries event-driven signals — layout with `time_column`, file findable by `window_start`/`window_end`, readable by in-file timestamps.

## Test 5 — SEMES pack re-verification

SEMES pack (Phase 0, Outdated) re-verified against v2.2: tree + routing + recipe versions re-checked, TRACE/TRACE+SCALAR routing confirmed, pack SQL updated, status → Live.

## Test 6 — Ops drills

- **Tiering (D-043):** tier one file COLD then PURGED on a test instance; reader errors name the tier; tombstone + provenance survive.
- **Time validation (D-044):** one deliberately bad `attrs` timestamp fails the load with a clear error.
- **Dataset manifest (D-046):** first Core-Model-bound extract ships a manifest; re-extract at the same watermark is row-count-identical.

## Status board

| Test | Blocked by | Status |
| --- | --- | --- |
| 1 · sck migration v2.0 → v2.2 | — | **7 of 8 gates green (2026-09-19).** G1–G5, G7, G8 pass. **G6 blocked pending D-051.**`sck` is on v2.2. |
| 2 · SCK corner load | ~~Test 1~~ — unblocked | Not started |
| 3 · Innotek load + benchmarks | data delivery | Not started |
| 4 · Standing-asset case | data delivery | Not started |
| 5 · SEMES re-verification | ~~Test 1~~ — unblocked (shared tooling now on v2.2) | Not started |
| 6 · Ops drills | ~~Test 1~~ — unblocked | Not started |

**Definition of done for the phase:** all six tests green (or superseding decisions logged for any failure), SCK/SEMES packs back to Live, and the D-033/D-035 benchmark verdicts recorded on [05 · Decision Log — D-001…D-050 (append-only)](./05-decision-log.md).
