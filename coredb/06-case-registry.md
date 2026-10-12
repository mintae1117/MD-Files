# 06 · Case Registry — Domain Packs
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/376930345/06+Case+Registry+Domain+Packs
> pageId: 376930345 · lastModified: 9월 20, 2026 · saved: 2026-10-12

> One entry per onboarded (or planned) data source. A **pack** = the written mapping spec for one source: entity tree, naming recipes, parameter dictionary + lanes, specs, relations, known limits — plus its as-built load evidence and lessons (L-###). New field-test packs are authored **here** as child pages of this one, using the section list below. The full historical SCK/SEMES packs remain in the frozen Notion Case Registry (linked per row) as evidence for the decisions that cite them.

## Registry

| Case | Source code | Industry | Tree | Lanes | Status | Notes |
| --- | --- | --- | --- | --- | --- | --- |
| SEMES — semiconductor etcher | SEMES | Semiconductor etch | LOT → WAFER (SITE, RUN under WAFER) | SCALAR, TRACE, TRACE+SCALAR | **Outdated** — re-verify against v2.2 | Etch anomaly detection; 10 Hz × \~200ch traces; chamber routing + recipe versions. Pack written pre-v2.1 — do not run its SQL as-is. Full pack: frozen Notion archive — links on [05 · Decision Log — D-001…D-050 (append-only)](./05-decision-log.md). |
| SCK — final-test + wafer-sort (STDF) | SCK | Post-package test | Final test LOT → RUN → UNIT; die origin FAB\_LOT → WAFER → DIE; DIE→UNIT ASSEMBLY edges | SCALAR, SCALAR\_BULK | **Outdated** — pack re-verification still owed. **Instance now on v2.2**, migrated from v2.0 on  ([07 · Field-Test Plan — Verification Mode](./07-field-test-plan.md) Test 1: 7 of 8 gates green, G6 pending D-051). The database is `sck`; `sck_raw` is retained as the v2.0 rollback. | **Pack page: 399966230** — 06a · SCK Domain Pack (entity types, key definitions, link kinds), published 2026-09-01; incomplete against the template below, with each gap named on it. 26 files / \~40 GB loaded, 210,890 edges, idempotency proven — and re-proven post-migration by Test 1's G3. PTR/FTR columnar; bins SCALAR; limits in `param_spec` at TEST\_PROG\_VER grain. Corner data currently held out — loads next as Test 2 (D-038 gate), now unblocked. Pack lesson **L-SCK-01** → invariant I14. Full pack: frozen Notion archive — links on [05 · Decision Log — D-001…D-050 (append-only)](./05-decision-log.md). ── **The "6.53 M rows" figure this row used to carry has been removed as unreconciled.** Every one of the eleven v2.0 base tables sums to **1,275,035** rows (measured; see 06a §9). The likely explanation is that 6.53 M counted SCALAR\_BULK Parquet rows, which live in the object store and not in Postgres — but that is a hypothesis nobody has tested. Settle it by summing `file_ref.row_count` before quoting a volume figure again. |
| LG Innotek — camera-module tester | LG | Camera-module test | TRAY → MODULE; assembly genealogy (lens/sensor/PCB) via ASSEMBLY links | SCALAR | **Planned** — field-test case | Validates: D-019 assembly on real parts-list data, D-036/D-040 dual-source vocabulary + unit conversion, D-033 projection benchmark. An empty `innotek` instance already stands on v2.2. |
| KOGAS — gas-pipeline monitoring | KOGAS | Gas pipeline | Standing assets (pipeline segments) | SCALAR, TRACE, TIMESERIES | **Planned** — field-test case | Validates: D-035 TIMESERIES lane at real cadence, D-041 irregular traces, D-042 windows/attribution on entity-less telemetry. |

## Pack lessons (L-###)

| ID | Lesson | Promoted to |
| --- | --- | --- |
| L-SCK-01 | One `type_code` was overloaded with two item kinds (test lots + fab lots both as LOT, split by an `attrs` marker). Generated views filter on `type_code`, so the overload silently mixed two kinds into every downstream view; the naming registry was untruthful for half the rows. Fix: separate `FAB_LOT` type, entities retyped in place, spec corrected. | Invariant **I14** (D-028) |
| L-SCK-05 | A pack's tooling can rot while the pack sits idle. Test 1 found the SCK loader's `.env` carrying **two stale credentials** — a Postgres password and a MinIO secret, both left over from a container that has since been replaced — so `validate.py` could not connect at all, and the pack's own acceptance gate was unrunnable until they were repaired. The gate had not been exercised since the load. **Run a pack's validator on a schedule, not only at load time**; a gate that has not been run recently is not known to work. | Not yet promoted — candidate for the sign-off gates on [02 · Data Engineer Runbook — New Data Arrives, Do This](./02-data-engineer-runbook.md) |

New lessons land in this table with the pack that produced them. A lesson that generalizes gets promoted into [01 · Schema Spec — v2.2 (Committed Baseline)](./01-schema-spec-v2.2.md) by a ratified decision, like L-SCK-01 was.

## Pack template (sections every new pack page must have)

1. **Snapshot** — source, anchor partner, what the data is, volumes/cadence.
2. **Item types** — `entity_type` rows (domain, parent, ≤16-char codes, I14 check).
3. **Naming recipes** — `entity_key_def` rows per type; declared source timezone (D-044); RUN discriminator rule for this source (D-039).
4. **Storage routing** — every parameter family → lane, with the decision-tree reasoning ([02 · Data Engineer Runbook — New Data Arrives, Do This](./02-data-engineer-runbook.md)).
5. **Dictionary** — key `param_def` rows; conditions (`param_condition`, D-038); concepts + conversion pairs (D-040); aliases.
6. **Layouts & files** — `file_layout_set` (lane, `time_column` if irregular), file granularity, path scheme.
7. **Specs** — limits/setpoints and their governing PROCESS entities.
8. **Relations** — link kinds used; windows available? window\_end rule + guard band for this source (D-042).
9. **Known limits hit** — logged per [04 · Known Limits & Deferred Items — Read Before Field Testing](./04-known-limits-deferred.md); never worked around silently.
10. **As-built evidence** — load counts, idempotency re-run proof, sign-off gate results ([02 · Data Engineer Runbook — New Data Arrives, Do This](./02-data-engineer-runbook.md) §gates).
