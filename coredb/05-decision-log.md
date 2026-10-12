# 05 · Decision Log — D-001…D-050 (append-only)
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/377061389/05+Decision+Log+D-001+D-050+append-only
> pageId: 377061389 · lastModified: 9월 01, 2026 · saved: 2026-10-12

> **Append-only.** Never edit a ratified decision — supersede it with a new one. The log continues **here** at D-051. Full rationale + alternatives-rejected for D-001…D-048 are preserved verbatim in the frozen Notion page ("Decision history & open questions") — link below; the full record of D-037…D-048 is the child page **Close-out Package D-037…D-048**. This summary is the working index; the anti-relitigation rule stands: before proposing anything that resembles a rejected alternative, read the original entry and address its rationale head-on.

## How to add a decision

1. Next `D-###`, today's date, one-line decision + rationale + alternatives rejected + status — appended to the table below (fuller record as a child page when the decision is large).
2. Never edit or delete a ratified row. To change one: append a new entry "supersedes D-0xx", mark the old row Superseded.
3. A decision that would break any invariant I1–I14 must arrive as a proposal with a full requirements walk (satisfied as-is / differently / consciously dropped with the failure mode named) — per D-029.
4. Ratified schema changes ship as numbered migrations through the \[08 · Operations\] discipline, and usually need a consistency sweep of the other pages here.

## The log

| ID | Date | Decision (summary) | Status |
| --- | --- | --- | --- |
| D-001 | 2026-06-14 | Long-format core: new sensors/item-types/naming = rows, never schema changes | Ratified (baseline) |
| D-002 | 2026-06-14 | Storage classes behind one `param_id`: SCALAR rows, TRACE Parquet, TRACE+SCALAR | Ratified (baseline) |
| D-003 | 2026-06-14 | Identity from stable business codes only; reprocessing at RUN level; time/equipment in `attrs` | Ratified (baseline) |
| D-004 | 2026-06-14 | `entity_key_def` append-only + canonicalization; identity is a configurable combination of codes | Ratified (baseline) |
| D-005 | 2026-06-14 | All joins on `entity_id`; `natural_key` derived, display/resolve/prefix only; no `/` in `local_id` | Ratified (baseline) |
| D-006 | 2026-06-14 | Roll-ups computed at parent grain, never stored as parent measurements | Ratified (baseline) |
| D-007 | 2026-06-14 | Wide tables derived & disposable; never a source of truth | Ratified (baseline) |
| D-008 | 2026-06-14 | One SVID, one definition — meaning only in `param_def`; layouts store no names/units | Ratified (baseline) |
| D-009 | 2026-06-14 | Soft-delete in registries; never hard-delete rows with measurements | Ratified (baseline) |
| D-010 | 2026-06-14 | Canonicalization closed & immutable (AS\_IS/UPPER/DIGITS\_PAD2/DIGITS\_PAD3); missing recipe fails loudly | Ratified (baseline) |
| D-011 | 2026-06-14 | Many sources share one schema via `source`/`source_code` namespacing; onboarding = INSERT rows | Ratified (baseline) |
| D-012 | 2026-06-14 | Polymorphic value storage; PK (`entity_id`,`param_id`) = one scalar per (item, param) | Ratified (baseline) |
| D-013 | 2026-06-14 | UNIQUE NULLS NOT DISTINCT on entity locals; idempotent resolve-or-create | Ratified (baseline) |
| D-014 | 2026-06-14 | Trace file granularity & path mirror `natural_key`; shared channel map per layout | Ratified (baseline) |
| D-015 | 2026-06-29 | Equipment as a second dimension: `entity_type.domain`, TOOL→CHAMBER; `parent_id` never crosses domains | Ratified |
| D-016 | 2026-06-29 | Product↔equipment routing via a surrogate-keyed bridge (shape later unified by D-030) | Ratified; shape superseded by D-030 |
| D-017 | 2026-06-29 | Process recipes as a third dimension (PROCESS): RECIPE → RECIPE\_VERSION, versioned identity | Ratified |
| D-018 | 2026-06-29 | Recipe wired into the routing event (now `link.context_entity_id`); event = product × equipment × recipe | Ratified; shape superseded by D-030 |
| D-019 | 2026-06-30 | Many-to-many genealogy beside the tree (ASSEMBLY/MERGE/SPLIT); never feeds identity; resolves C-01 | Ratified; shape superseded by D-030 |
| D-020 | 2026-06-30 | SCALAR\_BULK lane for high-volume homogeneous scalar (STDF PTR): columnar Parquet, zero measurement rows | Ratified |
| D-021 | 2026-06-30 | `param_spec` for per-(param, context) limit/setpoint bands; resolves C-06 | Ratified |
| D-022 | 2026-07-06 | Equipment-grain `measurement` = current-value snapshots only; standing-asset history reserved for a real-data decision (→ D-035) | Ratified |
| D-023 | 2026-07-06 | `param_spec` context must be PROCESS-domain (guard); per-product contexts out of scope | Ratified (reaffirmed D-032, D-048) |
| D-024 | 2026-07-06 | Batch-ratify D-019–D-023 ahead of real-data validation; validation lives in the case tests; flaws fixed by superseding decisions | Ratified |
| D-025 | 2026-07-06 | Layout catalogs unified into lane-tagged `file_layout_set`/`file_layout` (15 → 13 tables) | Ratified |
| D-026 | 2026-07-06 | Deployment: logically universal, physically per-project — one unforked schema, one instance per customer | Ratified |
| D-027 | 2026-07-06 | `type_code` ≤ 16 chars (`TEST_PROG_VER`); widening rejected | Ratified (reaffirmed D-032) |
| D-028 | 2026-07-06 | Invariant I14 — one `type_code`, one item kind (promoted from pack lesson L-SCK-01) | Ratified |
| D-029 | 2026-07-09 | Project rescope: L1 rows / L2 schema / L3 architecture; invariants = portable requirements; verdict-bound explorations; Milestone 2 | Ratified |
| D-030 | 2026-07-20 | Schema v2.0: one `link` + `link_type` for all relations; one `file_ref` for both file lanes (13 → 11 tables) | Ratified |
| D-031 | 2026-08-02 | Setpoints have one home: `param_spec` SETPOINT (context = RECIPE\_VERSION); curve setpoints via RECIPE\_STEP or TRACE file | Ratified |
| D-032 | 2026-08-02 | Outside review R-001 disposition: accepted items → P-002/P-003/P-004/D-031; renames REJECTED; D-023/D-027 reaffirmed; two items deferred with triggers | Ratified |
| D-033 | 2026-08-02 | Relations verdict: `link` reaffirmed sole truth; multi-hop traversal via derived graph projection (I8); graph-as-truth NOT ratified | Ratified |
| D-034 | 2026-08-02 | Schema v2.1: versioned `param_spec` bands; `link_id` + context-aware natural key; `ingest_batch` provenance + CORRECTION policy + `measurement_audit`; `file_ref` windows; TEXT widening; guard concurrency fixes | Ratified |
| D-035 | 2026-08-02 | TIMESERIES lane: `observation` (PK entity, param, ts; partitioned); resolves C-03; cadence-bounded (I5) | Ratified |
| D-036 | 2026-08-02 | Vocabulary layer: `param_concept` + `param_def.concept_id` + `param_alias`; resolves C-02; no SV role (I9/D-031) | Ratified |
| D-037 | 2026-08-11 | **Phase change:** verification & field-test mode; truth moves to this Confluence space; Notion frozen with pointers; log continues here at D-049 | Ratified |
| D-038 | 2026-08-11 | **Resolves C-04:** condition model — per-condition `param_def` + shared concept + new `param_condition` table; per-corner limits via `param_spec`; RUN grain for execution-level conditions | Ratified (gate: SCK corner load) |
| D-039 | 2026-08-11 | **Closes C-05 residual by convention:** source discriminator → new RUN; no discriminator → idempotent same-row landing is correct | Ratified |
| D-040 | 2026-08-11 | Unit conversion: `param_def.to_canonical_scale`/`_offset` pair; declared at mapping; values stored as-received, converted on read; non-linear deferred with trigger | Ratified (gate: Innotek dual-source) |
| D-041 | 2026-08-11 | Irregular/timestamped traces: `file_layout_set.time_column`; NULL = uniform sampling | Ratified (gate: first irregular trace) |
| D-042 | 2026-08-11 | Time-window bridge on `link`: `window_start`/`_end` + generated `ts_window` + composite GiST; windows never identity; set-based overlap validation (running-max, no EXCLUDE); differing window re-send = CORRECTION; guard-band attribution; clock-domain registry deferred. Adopts the 2026-08-08 exploration | Ratified (gates: real-data benchmark + guard-band demo) |
| D-043 | 2026-08-11 | Resolves Q-002: `file_ref.tier` (HOT/COLD/PURGED) + tombstone rule; retention = per-instance config; purge = explicit logged ops action | Ratified (gate: ops drill) |
| D-044 | 2026-08-11 | Resolves Q-003: `attrs` timestamps are UTC ISO-8601 `Z`, keys end `_at`; unparseable/ambiguous time fails the load | Ratified |
| D-045 | 2026-08-11 | Resolves Q-006: `schema_migration` table + numbered migrations, dev → verify → promote; instance version is one query | Ratified (gate: sck migration) |
| D-046 | 2026-08-11 | Resolves Q-007: dataset-manifest contract (instance, schema\_version, watermark\_batch\_id, built\_at, generator); DB stores nothing about derived datasets | Ratified (gate: first Core-Model extract) |
| D-047 | 2026-08-11 | Resolves Q-008: DDL is truth — `file_ref` stores no summary stats; source-provided summaries = TRACE+SCALAR params; derived summaries recomputed (I6) | Ratified |
| D-048 | 2026-08-11 | Per-product limit contexts: formal DEFER with binding trigger (real per-product-limits dataset); D-023 guard unchanged | Ratified (DEFER) |
| D-049 | 2026-08-14 | **Docs only:** add **00 · Start Here — New Engineer Onboarding Manual** (EN, with KO child page) as a non-normative learning aid for new data engineers; live pages 01–08 remain the sole source of truth and win on any disagreement; manual carries its verification date and must be re-checked after schema-changing decisions; no schema impact | Ratified (docs) |
| D-050 | 2026-09-01 | **Supersedes Test 1's G1 oracle and scope line** (07 · Field-Test Plan). Root cause: G1 was transcribed from the Notion v1.1 as-built figures and never re-derived after the v2.0 rebuild — one defect, three symptoms. **(1)** G1's oracle becomes the loader harness's 23-key set; all 23 verified against the live instance 2026-08-31/09-01, all matching; `param_def` 3,415 → **3,521** and limits 2,786 → **2,813**; the other nine figures were correct. **(2)** Test 1's scope is **v2.0 → v2.2**, not v1.1 → v2.2 — the live instance has eleven base tables in `public` and no `schema_migration`; the v1.1 instance is a separate stopped container with its own volume. The "on v1.1" statement is corrected on pages 01, 05, 07 and 08. **(3)** G1 covers **every** `entity.type_code` and **every** `link_kind` present at the pinned state, not a named subset. **(4)** Migration path ruled: dump to the target name, then Block 2 and Block 3 — **Block 1 is never re-run**, because its seed creates four generic types and five link kinds and would fail FK on all thirteen SCK types and on MEMBER\_OF. **(5)** The SCK domain pack is published as a child of 06 · Case Registry. Alternatives rejected: patching only the two figures; scoping G1 to six entity types. Deferred to its own decision: relocating the oracle and the two L-SCK-04 guard expectations out of `TESTS/`. Breaks no invariant I1–I14. Full record: child page `399179813` | Ratified |

**Schema version history:** v1.0 (15 tables, D-024 consolidation) → v1.1 (13 tables, D-025) → v2.0 (11 tables, D-030) → v2.1 (16 tables, D-034–D-036) → **v2.2 (18 tables, D-038–D-045)**. The loaded `sck` instance is on **v2.0** and migrates v2.0 → v2.2 by Block 2 then Block 3 (D-050, measured 2026-08-31; the earlier "on v1.1" reading described the separate `universal_db` container, which has been stopped since 2026-07).

## Archive links (frozen Notion — full history, read-only)

- Decision history & open questions (full rationale + alternatives for D-001…D-048, changelog Rev 1–20): `<NOTION_ARCHIVE_LINK>`
- Guideline v2.1 (superseded by \[01 · Schema Spec\] here; includes the full v2.1 column dictionary and dimension worked examples): `<NOTION_ARCHIVE_LINK>`
- Architecture & Schema Proposals sandbox (P-001…P-005 full proposals + outside review R-001): `<NOTION_ARCHIVE_LINK>`
- Coverage Project hub (H2 2026 working view): `<NOTION_ARCHIVE_LINK>`
- Case Registry database (full SCK/SEMES packs incl. pack log L-SCK-01): `<NOTION_ARCHIVE_LINK>`
