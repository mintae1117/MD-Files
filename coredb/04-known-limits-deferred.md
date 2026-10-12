# 04 · Known Limits & Deferred Items — Read Before Field Testing
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/377028653/04+Known+Limits+Deferred+Items+Read+Before+Field+Testing
> pageId: 377028653 · lastModified: 9월 20, 2026 · saved: 2026-10-12

> **Audience: everyone running field tests.** These are the model's honest edges as of v2.2. Every item below is *deliberately* deferred, with a written reopen trigger — none are forgotten. Hitting one is expected and valuable: **log it, don't work around it.**

## How to log a hit

1. Don't patch silently, don't dump the shape into `attrs` (each "put it in attrs" is itself a coverage defect).
2. Record: which source, which data shape, what you tried, which rule/guard blocked it.
3. Add it to the table below (this page is the successor of the Notion coverage backlog) and raise it — the fix arrives as a ratified decision on the [05 · Decision Log — D-001…D-050 (append-only)](./05-decision-log.md), possibly superseding an earlier one. That's the designed path, not bureaucracy: it's how the schema stays universal instead of forked.

## Specification gaps — the ratified DDL does not implement what a ratified page asserts

A class of defect this page had no row for until Test 1 found two of them. These are not coverage defects (nothing fails to fit as rows) and not deferred design choices. They are places where prose on one ratified page describes behaviour that **no DDL in any ratified block actually creates**, so an instance built exactly to spec does not behave as the spec says. Both were found by measurement on , not by reading.

| Gap | What the pages say | What is actually true | Resolution |
| --- | --- | --- | --- |
| `link_guard()` advisory lock | [08 · Operations — Instances, Migrations, Retention, Guards](./08-operations.md) states as operational fact that the guard takes `pg_advisory_xact_lock(hashtext(NEW.link_kind))` before the cycle walk on acyclic kinds, "closes the READ COMMITTED race". Block 2 repeats it in a trailing comment. | **No **`CREATE OR REPLACE FUNCTION link_guard()`** carrying the lock exists in Block 1, 2 or 3.** Measured: with the ratified guard, two concurrent READ COMMITTED sessions inserting A→B and B→A both commit, and a cycle lands on an acyclic kind. With the lock, the second blocks then raises `link ASSEMBLY would create a cycle`. | **OPEN — needs D-051.** Fix drafted at `INFRA/sql/udb/migrations/003_link_guard_advisory_lock.sql`, deliberately unapplied. Blocks [07 · Field-Test Plan — Verification Mode](./07-field-test-plan.md) gate G6. |
| I12 correction policy on `measurement` | Block 2 states it as a POLICY comment: a re-send whose value differs fails loudly unless `batch.kind='CORRECTION'`, which archives the old row to `measurement_audit` first. I12 makes it an invariant. | No trigger, constraint or rule implements it. `measurement_audit` is created as a table and nothing ever writes to it. A plain `ON CONFLICT DO UPDATE` — which is what the v2.0 loader did — silently overwrites, exactly what I12 forbids. | **PARTIALLY CLOSED by convention.** Implemented loader-side in the SCK loader (proven by G4), which matches [02 · Data Engineer Runbook — New Data Arrives, Do This](./02-data-engineer-runbook.md) calling this the loader contract. **Nothing in the database enforces it against any other writer** — a second loader, a backfill script or a hand-run UPDATE bypasses I12 entirely. Reopen trigger: a second writer to `measurement` on any instance. |

**Worth generalizing:** both gaps have the same shape — normative behaviour living in a comment rather than in executable DDL. A comment cannot be tested and does not run. When auditing the spec, treat "the page says the guard does X" as a claim to verify against `pg_get_functiondef`, not as a fact.

## Deferred items with reopen triggers

| Item | Status | Reopen trigger | Decision |
| --- | --- | --- | --- |
| Per-product limit contexts (one test program, different limits per product/package) | DEFERRED — `param_spec` context stays PROCESS-domain; the guard rejects PRODUCT contexts | A real dataset where one program carries different limits per product/package | D-048 (D-023/D-032 lineage) |
| Clock-domain registry (per-source clock offset/uncertainty as data) | DEFERRED — guard-band convention covers attribution for now | First real cross-source mis-attribution incident, or observed drift in the first telemetry case | D-042 |
| Non-linear unit conversion (dB, log scales) | DEFERRED — linear scale/offset covers all current cases | First real non-linear unit case → conversion-function registry | D-040 |
| Non-numeric TIMESERIES telemetry (status/boolean channels) | DEFERRED — `observation` v1 is numeric-only | A real case needing status telemetry in SQL | D-035 (P-003 scope choice) |
| Lazy per-die entities for SCALAR\_BULK | DEFERRED — eager rows earn their keep while dies are individually referenced | A case whose file rows are never individually referenced, or measured entity-volume pain | D-032 |
| `attrs` key registry | DEFERRED — folded into D-044's convention for time keys; other keys per pack spec | Recurring cross-pack `attrs` key collisions | D-032 |
| Graph store as system of record | REJECTED for now — `link` + derived projection is the ratified hybrid | New proposal with real evidence only; the projection benchmark does NOT reopen this | D-033 |
| Multi-script packages vs a one-hash `script_sha256` | CLOSED BY CONVENTION — `schema_migration.script_sha256` is `CHAR(64)`, but the v2.2 package is two scripts. Convention: hash them concatenated in apply order, keep per-file hashes in a `CHECKSUMS.txt`, and write the value with a final `UPDATE` (a script cannot contain its own hash). | A package where per-script attestation actually matters — e.g. a partially-applied migration that must be proven script-by-script | D-045 |
| Conformance verified on PostgreSQL 16.13; instances run 17.10 | OPEN — named by D-050 and not yet resolved. Page 01 records the DDL as "verified end-to-end on PostgreSQL 16.13"; the live container is **17.10**. The v2.2 migration and all of Test 1's gates ran clean on 17.10, which is evidence but not a re-verification. | Either re-verify the full DDL stack on the deployed major version, or restate the claim's scope on page 01. Any instance on a third major version forces the issue. | D-050 (What this does NOT settle) |

## Residual edges closed by convention (not schema)

| Edge | The rule |
| --- | --- |
| Same pass, same chamber, **same** recipe, twice | If the source provides any discriminator → new RUN (folded into RUN `local_id`). If not → indistinguishable by construction; idempotent same-row landing is correct (D-039). |
| Open-ended occupancy windows (`window_end` NULL) | Never synthesized. Attribution refuses open windows unless the pack declares a next-start rule (D-042). |
| Attribution near window edges | ± guard band per source; in-band samples flagged AMBIGUOUS, never silently assigned (D-042). |
| Purged raw files | Consciously accepted loss past retention; tombstone row survives; derived summaries must exist before purge (D-043). |
| Rows written before their instance had provenance | `batch_id` stays NULL on rows loaded pre-v2.2. Never backfilled: inventing a batch for rows whose batch was never recorded would make the provenance column lie. A NULL `batch_id` reads as "written before this instance tracked batches", and the nightly orphan check must tolerate it (D-034). |
| All-NULL `link` windows in the overlap backstop | `tstzrange(NULL, NULL, '[)')` is unbounded, so it overlaps everything — a naive self-join reports a false positive on every pair. Restrict the overlap check to rows declaring a window on both sides; a row with no declared window has no occupancy to conflict over (D-042). |

## Unreconciled figures

Not defects in the model — defects in what we have written down about it. Left visible rather than quietly corrected, because each one was quoted somewhere as if measured.

- **SCK volume: "6.53 M rows" vs a measured 1,275,035.** Summing every one of the eleven v2.0 base tables in the loaded instance gives **1,275,035**. The registry row said 6.53 M. The likely explanation is that 6.53 M counted SCALAR\_BULK Parquet rows, which live in the object store and not in Postgres — but nothing has tested that. Settle it by summing `file_ref.row_count`. Recorded on 06a §9; the unqualified figure has been removed from [06 · Case Registry — Domain Packs](./06-case-registry.md).
- **The oracle has no auditable home.** `expected_counts.json` is the only correct copy of the SCK gate's expected values and it lives in `TESTS/`, which is unversioned and outside every agent's reach. D-050 defers this to its own decision; it is still open, and it means the gate that closed Test 1 has an authority nobody can audit. Test 1 did close the smaller half of it: the two guard labels `validate.py` computed without asserting (`disambiguated params (#)` = 211, `param_def name collisions` = 0) now carry expected values.

## Pending validations (field-test scope — the tests themselves)

These are ratified designs whose real-data validation is exactly what the field tests deliver. A flaw found here is fixed by a **superseding decision**, never a silent edit. Full plan: [07 · Field-Test Plan — Verification Mode](./07-field-test-plan.md).

- `sck`~~ instance migration v1.1 → v2.2 (one hop) — acceptance gates fixed.~~ ── **Corrected and DONE.** The scope was v2.0 → v2.2, not v1.1 → v2.2 (D-050, 2026-09-01). Migrated 2026-09-19; 7 of 8 gates green, G6 held open by the advisory-lock gap above.
- SCK + SEMES pack re-verification against v2.2 (both still marked Outdated).
- LG Innotek load: assembly genealogy (D-019), vocabulary + unit conversion (D-036/D-040), graph-projection benchmark (D-033).
- First standing-asset case (KOGAS or SEMES telemetry): TIMESERIES benchmark + TimescaleDB license check (D-035), window/guard-band demo (D-042).
- First corner-carrying SCK file: condition model (D-038). Now unblocked by Test 1.
- First irregular trace: `time_column` (D-041).
