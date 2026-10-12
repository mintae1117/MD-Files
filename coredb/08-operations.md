# 08 · Operations — Instances, Migrations, Retention, Guards
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/377061409/08+Operations+Instances+Migrations+Retention+Guards
> pageId: 377061409 · lastModified: 9월 20, 2026 · saved: 2026-10-12

> **Audience: whoever operates a Universal DB instance.** One unforked schema, one physical instance per customer/project (D-026): `sck`, `semes`, `innotek`, `kogas`, … Each instance migrates on its own schedule; this page is the discipline that keeps that safe.

## Instance rules (D-026)

- Never fork the schema per customer. A customer-specific need that seems to require a schema change is a coverage defect — log it ([04 · Known Limits & Deferred Items — Read Before Field Testing](./04-known-limits-deferred.md)), don't patch locally.
- Every instance answers its own version: `SELECT version FROM schema_migration ORDER BY applied_at DESC LIMIT 1;` — if the table is absent, the instance is pre-v2.2. Treat the [01 · Schema Spec — v2.2 (Committed Baseline)](./01-schema-spec-v2.2.md) as the target, not the disk.
- Concepts (`param_concept`) ship as seed rows in every instance; per-source mappings are pack work.

## Migration discipline (D-045)

1. Migrations are **numbered scripts in the repo**, each header naming the `D-###` ids it implements and its `script_sha256`.
2. Path: **dev → verify → promote.** Run on a copy of the target instance; run the acceptance gates (count-identical checks per the P-001 pattern + the migration's own gates, e.g. [07 · Field-Test Plan — Verification Mode](./07-field-test-plan.md) Test 1 for `sck`); only then promote.
3. Every applied package ends by inserting its `schema_migration` row — no row, migration not done.
4. **No hand-typed ALTERs on a live instance. Ever.** An emergency fix still goes through a script + a decision (possibly retroactive same-day, but logged).
5. Version lag is legal (D-026) but must be visible: the [06 · Case Registry — Domain Packs](./06-case-registry.md) row for the instance carries its current version.

**Practical note from the first run through this discipline (sck, 2026-09-19).** `schema_migration.script_sha256` is `CHAR(64)` — one hash — but a package can be more than one script (v2.2 is Block 2 + Block 3). The convention adopted: hash the scripts **concatenated in apply order**, record that, and keep per-file hashes in a `CHECKSUMS.txt` beside them. A script also cannot contain its own hash, so the value is written by a final `UPDATE` in the package rather than being a literal inside the last script.

## Retention & tiering ops (D-043)

- Retention windows per (`source_code`, lane) live in per-instance deployment config. **Default = keep everything.** Setting a window is an instance-owner action, recorded in the instance's ops notes.
- Tier transitions: COLD = move object to archive storage + update `storage_uri` + set `tier`, `tier_changed_at` in one transaction. PURGED = delete object, set tier — **the row stays forever** (tombstone).
- Purge preconditions: every derived summary param from the file already exists as measurements; purge action logged as its own batch. Never automatic.
- Readers fail loudly on non-HOT tiers — that error reaching a user is working as designed; the fix is a restore-or-accept decision, not a retry loop.

## Guard operations (D-034)

**The advisory lock below is asserted by this page but implemented by no ratified DDL.** Verified 2026-09-19: Block 1, 2 and 3 on [01 · Schema Spec — v2.2 (Committed Baseline)](./01-schema-spec-v2.2.md) contain no `CREATE OR REPLACE FUNCTION link_guard()` carrying `pg_advisory_xact_lock` — Block 2 only describes it in a trailing comment. Measured on a throwaway instance running the ratified guard: two concurrent READ COMMITTED sessions inserting A→B and B→A **both commit**, and a cycle lands on an acyclic kind. With the lock, the second session blocks then raises. A fix is drafted (`INFRA/sql/udb/migrations/003_link_guard_advisory_lock.sql`) and held pending **D-051**; it blocks [07 · Field-Test Plan — Verification Mode](./07-field-test-plan.md) gate G6. Until then, read the first bullet as the intent, not as what any instance actually runs.

- `link_guard()` takes `pg_advisory_xact_lock(hashtext(NEW.link_kind))` before the cycle walk on acyclic kinds — closes the READ COMMITTED race. Serializes writers per kind; fine with one loader per kind.
- **Bulk loads:** `ALTER TABLE link DISABLE TRIGGER trg_link_guard` inside the load transaction, then ONE set-based validation before COMMIT: domain rules + cycle check + window-overlap check (running-max form — see [02 · Data Engineer Runbook — New Data Arrives, Do This](./02-data-engineer-runbook.md) for the SQL). Never commit unvalidated.
- **Nightly backstop verification** on every live instance: domain-rule violations (0 expected), cycles per acyclic kind (0), window overlaps (0), orphaned `batch_id`s (0), measurement rows violating `one_value_only` semantics vs `param_def.value_type` (0), file-lane params with measurement rows (0).
- **Parent-rename runbook:** a parent `local_id` change = single-transaction recursive `natural_key` recompute (fresh keys can't collide mid-transaction; extends I3's re-key mechanic).

**Writing the overlap check.** When every `link` row has NULL windows, `ts_window` is `tstzrange(NULL, NULL, '[)')` — unbounded, so it overlaps everything and a naive self-join reports a false positive on every pair. Restrict the check to rows that actually declare a window (`window_start IS NOT NULL OR window_end IS NOT NULL`) on both sides. A row with no declared window has no occupancy to conflict over.

## Partition ops (D-035)

- `observation` is `PARTITION BY RANGE (ts)`, monthly by default. Create partitions ahead (at least next month); a missing partition fails inserts loudly — that's the correct failure, fix by creating the partition, not by widening one.
- Partition creation/drop is ops config, not schema change — no decision needed. Dropping a partition IS data deletion: same preconditions as purge (D-043).
- Because it is ops config, keep it **out of the numbered migration scripts**, whose hash must attest to the ratified block text alone. `sck` and `innotek` both carry `observation_2026_09` … `observation_2026_12`, created from `INFRA/sql/udb/ops/observation_partitions.sql`.

## Backups & the derived layer

- Backup scope = the 18 base tables + the Parquet object store. The derived layer (wide tables, views, graph projection) is **not** backed up — it regenerates from truth (I8). If restoring a backup, rebuild derived artifacts; check dataset manifests (D-046) for which watermark downstream consumers pinned.

## Instance version board

| Instance | Schema version | Note |
| --- | --- | --- |
| `sck` | **v2.2** | Migrated from v2.0 on  (Block 2 then Block 3; Block 1 not re-run), scope and path per [D-050 — Full Record: Test 1's oracle and scope superseded by the measured v2.0 baseline (ratified 2026-09-01)](./05-D050-full-record.md). [07 · Field-Test Plan — Verification Mode](./07-field-test-plan.md) Test 1: 7 of 8 gates green, G6 pending D-051. `script_sha2566af0eccb…21dae2`. |
| `sck_raw` | v2.0 | **Rollback, not a live instance.** The pre-migration database, left untouched under its original name; `sck` is the restore of its dump. Retire it once Test 1 is fully green and the phase closes. |
| `semes` | — (pack Phase 0) | Stands up on v2.2 directly |
| `innotek` | v2.2 | Created empty on v2.2; no pack rows loaded yet |
| `kogas` | not yet created | Stands up on v2.2 directly |

Keep this board current — it is the one place "which version is where" is written down outside the databases themselves.
