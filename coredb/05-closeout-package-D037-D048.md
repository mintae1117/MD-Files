# Close-out Package D-037…D-048 — Full Record (ratified 2026-08-11)
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/377028674/Close-out+Package+D-037+D-048+Full+Record+ratified+2026-08-11
> pageId: 377028674 · lastModified: 8월 13, 2026 · saved: 2026-10-12

> **This is the full record the D-037…D-048 log entries cite** — trigger, design, decision-log regression scan, consolidated I1–I14 requirements walk, migration impact, and validation gates for each decision. Ratified 2026-08-11 by project owner. The same text was appended to the frozen Notion decision log as its final entries. Schema v2.1 → v2.2 (16 → 18 tables). All DDL below was executed and verified on PostgreSQL 16.13 on 2026-08-11 **before** ratification.

## What this package did

- Closed **Milestone 1**: every coverage-backlog item (C-01…C-06), watch-list item and open question (Q-002…Q-008) is now resolved by design, resolved by convention, or formally deferred with a named reopen trigger.
- **Milestone 2** status: the architecture verdict was already ratified (D-033); the verification-mode benchmarks remain scheduled as field tests.
- Produced **schema v2.2** — additive only, no key reshapes — so the pending `sck` migration jumps v1.1 → v2.2 in one hop (the same argument D-034 used).
- Moved the single source of truth to this Confluence space (D-037); the five Notion pages are frozen with pointer banners.

## Decision summaries with full reasoning

### D-037 — Phase change; truth moves to Confluence

Verification & field-test mode. Confluence COREDB = single source of truth (curated engineer set); Notion frozen, nothing deleted; append-only log continues here at D-049; ratify-then-validate and supersede-never-rewrite carry over. *Rejected:* keeping Notion as truth; parallel maintenance (drift, two truths); 1:1 migration of all history (noise).

### D-038 — Resolve C-04: the condition model (+1 table)

Execution-level conditions stay at RUN grain. Measurement-level conditions: one `param_def` per condition (native\_id suffix, SCK `t1500.03` precedent), tied by one `param_concept`; new `param_condition` (param\_id, condition\_key, value\_num/value\_text, one-value CHECK) makes the condition machine-readable; per-corner limits are ordinary `param_spec` rows. *Regression scan:* D-012's rejected "multiple rows per (entity,param)" stays rejected — `measurement` PK untouched; composes with D-036; no second catalog (I9); consistent with I14/L-SCK-01. *Rejected:* condition in `measurement` PK (reshapes largest table, weakens I11 model-wide); condition sub-entities (phantom items, I1); `attrs` markers (I14). *Gate:* SCK corner-file load — corner-curve query, per-corner limits, non-corner params unaffected.

### D-039 — Close C-05 residual by convention (no schema)

Source gives a discriminator → new RUN (folded into RUN `local_id`; SCK filename-CRC precedent). Source gives nothing → two visits are indistinguishable by construction; the idempotent same-row landing (I12) is correct behavior. No schema can distinguish what the source doesn't report. Formalizes exactly where D-034 left the residual. *Rejected:* window/`captured_at` in the link key (I1, already rejected); synthetic event counters (invented identity, breaks idempotent re-sends).

### D-040 — Unit-conversion mechanics (columns on `param_def`)

`to_canonical_scale`/`to_canonical_offset` pair (pair-or-neither CHECK); canonical = value × scale + offset; declared at concept-mapping time, mandatory when unit ≠ canonical\_unit; values stored as-received, converted on read. Decided **ahead of** the Innotek dual-source validation D-036 reserved it for — disclosed, D-024 pattern; that validation stays the gate. *Why not a unit-string registry:* units are free text — string-keyed conversion resolves silently or wrongly; the per-param pair is deliberate and auditable. SCK: STDF values are already base-unit (src\_unit/src\_scale = provenance) — most map as identity. *Rejected:* unit-string registry; storing converted values (two truths). *Reopen trigger:* first real non-linear case → conversion-function registry.

### D-041 — Timestamped / irregular traces (`file_layout_set.time_column`)

NULL = uniformly sampled (`file_ref.sampling_hz`); set = the in-file timestamp column. The layout is the home because timing shape is a property of the file format shared by all files of the layout (D-014 pattern). Irregular traces were storable but not describable — this closes the gap without touching lanes. *Rejected:* per-file column on `file_ref` (duplicates a layout property); a new lane. *Gate:* first irregular-trace case (likely KOGAS).

### D-042 — Time-window bridge on `link` (adopts the 2026-08-08 exploration)

`window_start`/`window_end` (nullable, ordered CHECK) + STORED generated `ts_window tstzrange` + partial composite GiST `(dst_entity_id, link_kind, ts_window) WHERE is_active`. Bound rules: windows never in any key (`uq_link_natural` unchanged — the entire I1 risk surface, stated in writing); guard = set-based running-max overlap validation (`lag()` provably misses nested overlaps — re-verified: running-max 2/2, lag 1/2; EXCLUDE constraint rejected on a measured 32× bulk-insert penalty); a re-sent link with a different window = value conflict → CORRECTION batch (extends D-034; supersedes D-019's attrs-refresh wording for windows only); `window_end` never synthesized (NULL = open; attribution refuses open windows unless a per-source next-start rule is declared); attribution carries a ± guard band per source, in-band samples flagged AMBIGUOUS (measured: 3.4% of synthetic SCADA-cadence samples fall within 5 s of a boundary — without the band, typed windows make wrong answers faster); clock-domain registry DEFERRED (trigger: first cross-source mis-attribution or observed drift). *Why now:* `link` is the one row saying "product was in equipment" and its time was `attrs` JSON — a logged defect by the project's own rule; `file_ref` got identical columns in D-034; landing pre-`sck`-migration = one migration; unlocks entity-less source ingestion (SCADA/alarms/meters → EQUIPMENT `observation`, product attribution becomes a query). Perf numbers (0.18 s attribution at 400k links with the composite GiST; single-column GiST 3.3 s; no index 24.6 s) are synthetic-data directional pending the real-data gate. *Rejected:* EXCLUDE as primary guard; window in the natural key; a universal time spine on non-event tables; leaving time in `attrs`.

### D-043 — Raw-file retention & tiering (resolves Q-002)

`file_ref.tier` (HOT/COLD/PURGED, CHECK, default HOT) + `tier_changed_at`. Retention windows per (`source_code`, lane) are per-instance deployment config. COLD = archived object, `storage_uri` updated in-transaction. PURGED = object deleted, **row becomes a tombstone, never deleted** (I7 extended). Readers fail loudly on non-HOT. Purge is an explicit logged ops action requiring pre-existing derived summaries; never a background default. **Consciously accepted failure mode, named:** purged raw data cannot be re-derived. *Rejected:* hard-deleting rows (I7); background auto-purge; retention in schema.

### D-044 — Time-in-`attrs` convention (resolves Q-003)

All `attrs` timestamps: RFC 3339 / ISO-8601 UTC with `Z`; keys end `_at` (`start_time` grandfathered read-side); source-local times converted via pack-declared zone; unparseable/ambiguous fails the load (I13 applied to time). *Rejected:* epoch integers; storing source-local time; best-effort parsing.

### D-045 — Schema-version registry & migration discipline (+1 table; resolves Q-006)

`schema_migration` (version PK, applied\_at, script\_sha256, decision\_ids, note) — PROVENANCE of the schema itself. Every DDL package inserts one row; instance version is one query. Migrations = numbered repo scripts tied to D-### ids, applied dev → verify → promote with acceptance gates; no hand-typed ALTERs on live instances. *Why:* "confirm which version THAT instance is on" had no queryable answer — the sck v1.1 lag was tribal knowledge. *Gate:* the sck v1.1 → v2.2 migration is the first disciplined run.

### D-046 — Dataset-manifest contract (resolves Q-007)

Every derived dataset carries a manifest (Parquet KV metadata or sidecar JSON): `instance`, `schema_version`, `watermark_batch_id` (= max `ingest_batch.batch_id` included), `built_at`, `generator`. DB stores nothing about derived datasets (I8). Reproduce = same instance + version + `WHERE batch_id <= watermark`. *Rejected:* registering datasets in the DB (second truth); pinning by timestamp (unstable under corrections). *Gate:* first Core-Model extract ships a manifest; re-extract at the same watermark is row-count-identical.

### D-047 — Q-008 doc fix: the DDL is truth

`file_ref` stores no summary statistics. Source-provided per-trace summaries = TRACE+SCALAR params (SEMES pattern); derived summaries recomputed, never stored (I6; SCK TSR precedent "recompute from PTR"). The Confluence spec is written from the DDL, so the discrepancy dies here.

### D-048 — Per-product limit contexts: formal DEFER

`param_spec` context stays PROCESS-domain (D-023 guard unchanged). Binding reopen trigger (from D-032): a real dataset where one test program carries different limits per product/package. Designing without a case would re-propose what D-023 rejected with no new evidence — the no-regression rule forbids exactly that; deferral-with-trigger is the decision Milestone 1 requires.

## Consolidated requirements walk (I1–I14)

| Invariant | Verdict |
| --- | --- |
| I1 | AS-IS — D-042 windows never in any key; D-038 conditions are param data; D-039 discriminators are business codes |
| I2, I3, I4 | AS-IS — untouched |
| I5 | AS-IS — D-041 describes in-file timing, nothing unpivoted; D-042 does not relax the cadence rule |
| I6 | AS-IS — D-047 reaffirms; attribution outputs are derived reads |
| I7 | AS-IS, extended — D-043 tombstones; conscious retention loss named + mitigated |
| I8 | AS-IS — D-046 is I8's missing paperwork |
| I9 | AS-IS, strengthened — per-condition params each one definition, concept ties them; condition explicit |
| I10 | AS-IS — no new crossings |
| I11 | AS-IS — `measurement` PK untouched; corners are distinct params |
| I12 | SATISFIED DIFFERENTLY (extended) — differing window re-send = CORRECTION (D-042) |
| I13 | AS-IS — D-044 applies the same philosophy to time |
| I14 | AS-IS — RUN discipline consistent with L-SCK-01 |

Nothing consciously dropped except the D-043 retention loss, named above. Case Registry pack-log scan: L-SCK-01 is the only pack lesson; no conflicts (D-038/D-039 build on SCK precedents).

## Migration & coexistence

All v2.2 changes are additive — no PK reshapes, renames, or type changes; every v2.1 query and loader runs unchanged. `sck` retargets v1.1 → v2.2 in one hop; all P-001 + P-002 §6 gates reused unchanged, plus the corner-file load and windows-present-and-NULL checks. `schema_migration` makes per-instance version state queryable.

## Verification performed before ratification (PostgreSQL 16.13, 2026-08-11)

v2.0 base + v2.1 delta + v2.2 delta run clean in order → 18 tables. `param_condition` CHECK fires; corner-curve query correct. Conversion-pair CHECK fires. `uq_link_natural` still rejects duplicates with windows present; inverted window rejected. Running-max overlap validation finds 2/2 nested overlaps; `lag()` finds 1/2 (wrong). Attribution join `ts_window @> ts` returns the correct product. Performance figures remain synthetic-data directional until the field-test re-run.
