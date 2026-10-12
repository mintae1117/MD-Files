# D-050 — Full Record: Test 1's oracle and scope superseded by the measured v2.0 baseline (ratified 2026-09-01)
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/399179813/D-050+Full+Record+Test+1+s+oracle+and+scope+superseded+by+the+measured+v2.0+baseline+ratified+2026-09-01
> pageId: 399179813 · lastModified: 9월 01, 2026 · saved: 2026-10-12

**Ratified** 2026-09-01 by the project owner · **Recorded by** claude-pm-cowork · **Summary row:** the D-050 entry on [05 · Decision Log](./05-decision-log.md) · **Supersedes:** Test 1's G1 criterion and scope line on [07 · Field-Test Plan](./07-field-test-plan.md)

Page 07 requires that "a flaw found here is fixed by a **superseding decision** on 05 · Decision Log, never a silent edit." This is that decision.

## The decision

Test 1's count oracle and its scope statement are replaced by the **measured v2.0 baseline** below. Four limbs:

1. **The oracle** — G1 checks the full 23-key set the loader harness already computes, not a hand-listed subset.
2. **The scope** — Test 1 is **v2.0 → v2.2**, not v1.1 → v2.2.
3. **Coverage** — every `entity.type_code` and every `link_kind` present at the pinned pre-migration state.
4. **The migration path** — ALTER in place: dump to the target name, then Block 2 and Block 3. **Block 1 is never re-run.**
5. **The pack gets a home** — the SCK domain pack is published as a child of [06 · Case Registry](./06-case-registry.md).

## Root cause — one defect, three symptoms

G1 was transcribed from the Notion SCK Case Registry as-built figures and never re-derived after the v2.0 rebuild. `expected_counts.json` states its own provenance: *"Notion SCK Case Registry as-built figures (2026-07-06, v1.1 load) plus this project's own profiling of all 26 files for the rows the old loader did not create."*

Its `_deltas_vs_notion` block explains every discrepancy, and each reconciles arithmetically: `param_def` 3415 + 106 = 3521 (the L-SCK-04 fix splits 105 colliding test numbers into 211 disambiguated native\_ids); `param_spec` 2786 + 27 = 2813; `TESTED_ON` 26 RUNs × 3 equipment links = 78. PIN, PIN\_GROUP and MEMBER\_OF are *"new in this build — the old loader dropped PMR/PGR"*; HANDLER and LOADBOARD are *"new in this build (D1)"*.

So the seven entity types and two link kinds G1 omits are not an oversight — **they did not exist at v1.1.** The two wrong figures, the wrong scope line and the incomplete enumeration are one defect with three symptoms, which is why the oracle is superseded wholesale rather than patched figure by figure.

## The measured v2.0 baseline

Read from `sck_raw` on the running `postgres` container, 2026-08-31 and 2026-09-01, via `docker exec -i postgres sh -c 'psql -U "$POSTGRES_USER" -d sck_raw -Atc "<query>"'`. Every value matches `expected_counts.json`.

| Key | Value | Query |
| --- | --- | --- |
| LOT | 10 | `select type_code, count(*) from entity group by 1` |
| RUN | 26 | ″ |
| UNIT | 211593 | ″ |
| FAB\_LOT | 10 | ″ |
| WAFER | 98 | ″ |
| DIE | 206501 | ″ |
| PIN | 146 | ″ |
| PIN\_GROUP | 110 | ″ |
| TESTER | 2 | ″ |
| HANDLER | 2 | ″ |
| LOADBOARD | 3 | ″ |
| TEST\_PROGRAM | 1 | ″ |
| TEST\_PROG\_VER | 1 | ″ |
| link ASSEMBLY | 210890 | `select link_kind, is_active, count(*) from link group by 1,2` |
| link MEMBER\_OF | 865 | ″ |
| link TESTED\_ON | 78 | ″ |
| measurement | 634779 | `select count(*) from measurement` |
| param\_def | 3521 | `select count(*) from param_def` |
| param\_spec | 2813 | `select count(*) from param_spec` |
| file\_layout\_set | 1 | `select count(*) from file_layout_set` |
| file\_layout | 3518 | `select count(*) from file_layout` |
| file\_ref | 26 | `select count(*) from file_ref where lane='SCALAR_BULK'` |
| multi-pass dies | 4249 | ASSEMBLY links grouped by src, `having count(distinct u.parent_id) > 1` |

`entity` totals **418503**, the sum of all thirteen types. `link` totals **211833**, the sum of all three kinds. Both reconcile to the row. All `link` rows are `is_active = t`.

**Two figures change from G1 as written:** `param_def` 3,415 → **3,521** and limits 2,786 → **2,813**. The other nine G1 figures were correct and are unchanged.

**Note the **`file_ref` query is lane-filtered. An unfiltered `count(*)` agrees only because every `file_ref` row is `SCALAR_BULK` (measured: `SCALAR_BULK|26`). A future instance with a second lane would diverge.

## Limb 2 — the scope, and the four pages that state it wrongly

The live instance is **v2.0**: eleven base tables in `public`, and `schema_migration` absent — which page 08's own rule reads as "if the table is absent, the instance is pre-v2.2". The eleven tables, read by name, are exactly Block 1's: `entity`, `entity_key_def`, `entity_type`, `file_layout`, `file_layout_set`, `file_ref`, `link`, `link_type`, `measurement`, `param_def`, `param_spec`.

The v1.1 instance is a **separate stopped container** — `universal_db`, `postgres:17`, `Exited (128) 5 weeks ago`, with its own volume `universal_db_data`. It shares no data with `sck_raw`.

**Four pages state that the **`sck` instance is on v1.1, and all four are corrected by this decision: 01 · Schema Spec (status snapshot), 05 · Decision Log (schema version history line), 07 · Field-Test Plan (Test 1 scope), 08 · Operations (instance version board).

The UDB tables live in `public` — the sole non-system schema. This was unruled anywhere previously; it is settled here by measurement, and it means no `SET search_path` is required before any DDL block.

## Limb 4 — why Block 1 is never re-run

`entity.type_code` is FK-constrained to `entity_type` and `link.link_kind` to `link_type`. Block 1's seeds create **four generic entity types** (TOOL, CHAMBER, RECIPE, RECIPE\_VERSION) and **five link kinds** (ASSEMBLY, MERGE, SPLIT, PROCESSED\_IN, TESTED\_ON).

**None of SCK's thirteen type\_codes is in that seed, and MEMBER\_OF is not among the five kinds.** This is correct and intended — page 01 states the seeds are *"Generic dimension seeds (rows, not schema — adjust per deployment)"* and that "onboarding a new domain or customer requires writing rows (a domain pack), not editing the schema". SCK's types are a domain pack, which is the mechanism the schema is designed around.

The operational consequence is what this limb rules:

- **ALTER in place** — `pg_dump sck_raw | psql <target>`, then Block 2 then Block 3. The domain pack travels with the dump. **This is the path.**
- **Fresh instance from Block 1+2+3 then restore** — fails on FK violation for all thirteen type\_codes and for MEMBER\_OF.

The wrong path looks cleaner and fails late, which is why it is ruled rather than left to judgement.

Both PK reworks in Block 2 were checked against the data and are safe. `link` moves to a surrogate `PRIMARY KEY (link_id)` with the natural key demoted to `uq_link_natural UNIQUE NULLS NOT DISTINCT (src, dst, link_kind, context_entity_id)`; `context_entity_id` joins a UNIQUE, not the PK, so its 99.96% NULL rate (measured: ASSEMBLY 210890/210890 NULL, MEMBER\_OF 865/865, TESTED\_ON 0/78) is no obstacle — `NULLS NOT DISTINCT` is the deliberate design for it. The existing three-column PK already guarantees the new constraint holds. `param_spec` widens its natural key with `valid_from`, and its old PK implies the new partial unique index a fortiori.

## Alternatives rejected

**Patch only the two wrong figures.** Leaves the stale scope line and the incomplete enumeration in place, and treats one symptom of a single root cause as if it were the whole defect. The next stale symptom would be found the same way, by accident.

**Scope G1 explicitly to six entity types and ASSEMBLY only,** recording the other 1,208 rows as deliberately outside the gate. Rejected: `expected_counts.json` already computes all 23 keys, so this would publish a gate narrower than the harness that already runs, and would require justifying on the record why the test-equipment topology and the MEMBER\_OF and TESTED\_ON edges need no check.

**Relocate the oracle and the guard expectations out of **`TESTS/` to a versioned location. Not rejected — **deferred to its own decision**, because it turns on whether aggregate counts may leave a confidential-data directory, which is a boundary question this decision has no business settling in passing.

## Invariants

This decision changes no table, column, key or guard. **It breaks no invariant I1–I14**, and therefore appends as a plain decision rather than arriving as a D-029 proposal with a full requirements walk.

## What this does NOT settle

- **The oracle's home.** `expected_counts.json` is the only correct copy and it lives in `TESTS/`, which is unversioned and outside every agent's reach. Citing it fixes the drift but leaves the gate with an unauditable authority. Own decision, named above.
- **The two L-SCK-04 guard expectations.** `validate.py` computes `param_def name collisions` and `disambiguated params (#)`, and neither has a recorded expected value — 0 and 211 respectively, derived from `expected_counts.json`'s delta note rather than read from the instance. A guard whose passing value is written nowhere is not a guard. Both belong in the oracle when it gains a home.
- **D6 — **`MEMBER_OF` promotion. Pin-group membership is not SCK-specific. Whether it belongs in the canonical Block 1 `link_type` seed rather than in every customer's domain pack is a real question, now with 865 rows behind it.
- `TEST_PROG_VER` is named but never seeded. Page 01's `param_spec.context_entity_id` comment reads *"governing node: TEST\_PROG\_VER / RECIPE\_VERSION"* and the `TESTED_ON` seed description says *"context = the TEST\_PROG\_VER that ran"*, yet no seed creates that `entity_type`. Any deployment using `param_spec` for LIMITs must add it.
- **The canonical DDL's verification scope.** Page 01 records it as *"verified end-to-end on PostgreSQL 16.13"*; the live container is `postgres:17`. Nothing obviously breaks, but the conformance claim's scope is 16.13 and should either be re-verified or stated as a gap.
- **The MinIO side.** Bucket contents, and whether `file_ref.storage_uri` points at `sck-raw` for all 26 rows, are unmeasured. They bear on the bucket naming choice, not on this decision.

## Provenance of this record

Every figure above was produced by the command printed beside it and read from the live instance on the dates stated. Nothing here is inferred from another document. Four earlier framings by the recording agent — a soft-delete hypothesis, two successive claims that the `link` PK rework endangered the data, and a claim that the domain-pack rows were a conformance defect — were each corrected by reading the artefact rather than reasoning about it, and none of them survives into this record.
