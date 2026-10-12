# 06a · SCK Domain Pack — entity types, key definitions and link kinds
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/399966230/06a+SCK+Domain+Pack+entity+types+key+definitions+and+link+kinds
> pageId: 399966230 · lastModified: 9월 01, 2026 · saved: 2026-10-12

> **The SCK domain pack, as it exists in the running instance.** Published under [D-050](./05-D050-full-record.md) limb 5, which rules that the pack gets a home. Every row below was read from `sck_raw` on the running `postgres` container on **2026-09-01** by the command printed beside it. Nothing here is inferred from another document.
> 
> **This page is INCOMPLETE against the ten-section pack template on **[**06 · Case Registry**](./06-case-registry.md)**.** Sections 2, 3 and 8 are published from measurement. Sections 1, 4, 5, 6, 7, 9 and 10 are **NOT YET PUBLISHED**; each states what reading or document would fill it. A section marked not-yet-published is a gap on the record, not an assertion that the pack has nothing there.

**Why this pack matters operationally.** D-050 limb 4 rules that the `sck` migration is **ALTER in place** — `pg_dump sck_raw` into the target name, then Block 2, then Block 3, and **Block 1 is never re-run**. The reason is on this page: `entity.type_code` is FK-constrained to `entity_type` and `link.link_kind` to `link_type`, and **none of the thirteen SCK type\_codes, and not **`MEMBER_OF`, appears in Block 1's generic seed. A fresh instance built from Block 1+2+3 and then restored into fails on those foreign keys. The pack travels with the dump; that is what makes the ruled path work.

## 1 · Snapshot — NOT YET PUBLISHED

What is on the record elsewhere, cited rather than restated: source code `SCK`; post-package test (final test + wafer sort) from STDF; 26 files, \~40 GB loaded — all from the SCK row on [06 · Case Registry](./06-case-registry.md).

**Missing:** the anchor partner, the cadence, and a volumes figure that reconciles. See *Known limits hit* below for the row-count discrepancy this section cannot resolve. Fills from `TESTS/sck_test/loader/CASE_REGISTRY_UPDATE.md`, which is written but unpublished.

## 2 · Item types — `entity_type` rows

```
docker exec -i "$PG" sh -c "psql -U \"\$POSTGRES_USER\" -d $DB -Atc \
  \"select type_code, parent_type, domain from entity_type order by 1\""
```

**17 rows returned. Four are Block 1's generic seed; thirteen are the SCK pack.** The thirteen are exactly the thirteen `type_code`s D-050 measures entity counts for.

**The pack's thirteen:**

| `type_code` | `parent_type` | `domain` | Entities (D-050) |
| --- | --- | --- | --- |
| `LOT` | — | PRODUCT | 10 |
| `RUN` | `LOT` | PRODUCT | 26 |
| `UNIT` | `RUN` | PRODUCT | 211,593 |
| `FAB_LOT` | — | PRODUCT | 10 |
| `WAFER` | `FAB_LOT` | PRODUCT | 98 |
| `DIE` | `WAFER` | PRODUCT | 206,501 |
| `TESTER` | — | EQUIPMENT | 2 |
| `HANDLER` | — | EQUIPMENT | 2 |
| `LOADBOARD` | — | EQUIPMENT | 3 |
| `TEST_PROGRAM` | — | PROCESS | 1 |
| `TEST_PROG_VER` | `TEST_PROGRAM` | PROCESS | 1 |
| `PIN` | `TEST_PROG_VER` | PROCESS | 146 |
| `PIN_GROUP` | `TEST_PROG_VER` | PROCESS | 110 |

**The four generic seeds, present and unchanged from Block 1:** `TOOL` (—, EQUIPMENT) · `CHAMBER` (`TOOL`, EQUIPMENT) · `RECIPE` (—, PROCESS) · `RECIPE_VERSION` (`RECIPE`, PROCESS). Compared field by field against Block 1's seed INSERT on [01 · Schema Spec](./01-schema-spec-v2.2.md) — identical. **All four carry zero entities in this instance**, which is why the thirteen pack types alone sum to the whole `entity` table.

**Two checks that pass on these rows.**

- **I10 — dimensions never cross via the tree.** Every non-null `parent_type` sits in the same `domain` as its child: `RUN`→`LOT`, `UNIT`→`RUN`, `WAFER`→`FAB_LOT`, `DIE`→`WAFER` all PRODUCT; `TEST_PROG_VER`→`TEST_PROGRAM`, `PIN`→`TEST_PROG_VER`, `PIN_GROUP`→`TEST_PROG_VER`, `RECIPE_VERSION`→`RECIPE` all PROCESS; `CHAMBER`→`TOOL` EQUIPMENT. No crossing. This is a check at the **type** level and does not prove the same of any `entity.parent_id` row.
- **The entity total reconciles.** 10 + 26 + 211,593 + 10 + 98 + 206,501 + 2 + 2 + 3 + 1 + 1 + 146 + 110 = **418,503**, the `entity` total D-050 records.

**Note on **`PIN` / `PIN_GROUP`. Both are PROCESS-domain children of `TEST_PROG_VER` — pins belong to the test program version that defines them, not to the product. This is what makes `MEMBER_OF` a PROCESS→PROCESS kind in section 8.

## 3 · Naming recipes — `entity_key_def` rows

```
docker exec -i "$PG" sh -c "psql -U \"\$POSTGRES_USER\" -d $DB -Atc \
  \"select type_code, source_code, seq, field_name, prefix, canon_rule
    from entity_key_def order by 1, seq\""
```

**18 rows returned. Four carry **`source_code = '*'` (Block 1's seed); fourteen carry `source_code = 'SCK'` — one per pack type except `DIE`, which takes two. 12 + 2 = 14.

| `type_code` | `seq` | `field_name` | `prefix` | `canon_rule` |
| --- | --- | --- | --- | --- |
| `DIE` | 1 | `DIE_X` | `X` | `DIGITS_PAD3` |
| `DIE` | 2 | `DIE_Y` | `Y` | `DIGITS_PAD3` |
| `FAB_LOT` | 1 | `LOT_ID_CHAR` | — | `UPPER` |
| `HANDLER` | 1 | `HAND_ID` | — | `UPPER` |
| `LOADBOARD` | 1 | `LOAD_ID` | — | `UPPER` |
| `LOT` | 1 | `LOT_ID` | — | `UPPER` |
| `PIN` | 1 | `LOG_NAM` | — | `UPPER` |
| `PIN_GROUP` | 1 | `GRP_NAM` | — | `UPPER` |
| `RUN` | 1 | `RUN_CODE` | — | `AS_IS` |
| `TESTER` | 1 | `NODE_NAM` | — | `UPPER` |
| `TEST_PROGRAM` | 1 | `JOB_NAM` | — | `UPPER` |
| `TEST_PROG_VER` | 1 | `JOB_REV` | — | `UPPER` |
| `UNIT` | 1 | `PART_ID` | — | `UPPER` |
| `WAFER` | 1 | `WAFERID` | `W` | `DIGITS_PAD2` |

**The four generic rows, unchanged from Block 1:** `TOOL`/`tool_id`/no prefix/`UPPER` · `CHAMBER`/`chamber_no`/`C`/`DIGITS_PAD2` · `RECIPE`/`recipe_name`/no prefix/`UPPER` · `RECIPE_VERSION`/`version`/`V`/`DIGITS_PAD2`. Compared field by field against Block 1's seed INSERT — identical.

**Checks.**

- **I13 — canonicalization is closed.** Every `canon_rule` used across all 18 rows is one of the four immutable names: `AS_IS`, `UPPER`, `DIGITS_PAD2`, `DIGITS_PAD3`. No rule outside the closed set appears.
- `DIE` is the only composite key in the pack — two codes, `X`-prefixed row then `Y`-prefixed, both `DIGITS_PAD3`, giving a die coordinate. Every other type keys from a single source field.
- `RUN` is the only type using `AS_IS` — its `RUN_CODE` is written through without normalisation. Note that `entity.local_id` carries `CHECK (local_id !~ '/')`, so an `AS_IS` field is the one place in this pack where an unnormalised source value meets that constraint directly.

**What this section does NOT establish**, and the template asks for: the **declared source timezone** (D-044) and the **RUN discriminator rule for this source** (D-039). Neither is readable from `entity_key_def`. `RUN_CODE`/`AS_IS` shows the discriminator arrives pre-formed in the source field, but the rule that forms it is not on this page and is not inferable from these rows.

## 4 · Storage routing — NOT YET PUBLISHED

Needs the per-parameter-family lane mapping with its decision-tree reasoning ([02 · Runbook](./02-data-engineer-runbook.md)). Two facts are on the record and cited, not restated: the pack uses the **SCALAR** and **SCALAR\_BULK** lanes ([06 · Case Registry](./06-case-registry.md) row), and **every **`file_ref` row in the instance is `SCALAR_BULK` — measured `SCALAR_BULK|26` (D-050). Fills from `select storage_class, count(*) from param_def group by 1` plus `CASE_REGISTRY_UPDATE.md`.

## 5 · Dictionary — NOT YET PUBLISHED

`param_def` totals **3,521** and `param_spec` **2,813** (D-050); the individual rows are not published. **The two figures that changed under D-050 live here** — the superseded G1 criterion carried `param_def` 3,415 and limits 2,786, and the delta is the L-SCK-04 fix splitting 105 colliding test numbers into 211 disambiguated `native_id`s. Fills from `param_def`, `param_condition`, `param_concept` and `param_alias` readings.

## 6 · Layouts and files — NOT YET PUBLISHED

`file_layout_set` = **1**, `file_layout` = **3,518**, `file_ref` = **26** (D-050). Lane, `time_column`, file granularity and path scheme are not published. Fills from `select * from file_layout_set` and a `file_ref` sample.

## 7 · Specs — NOT YET PUBLISHED

`param_spec` totals **2,813** (D-050), and the registry row states limits sit at `TEST_PROG_VER` grain. The instance holds exactly **one** `TEST_PROG_VER` entity (section 2), so every governing PROCESS entity in this pack is that one node — but which `spec_kind` values are present is not read. Fills from `select spec_kind, count(*) from param_spec group by 1`.

## 8 · Relations — `link_type` rows

```
docker exec -i "$PG" sh -c "psql -U \"\$POSTGRES_USER\" -d $DB -Atc \
  \"select link_kind, src_domain, dst_domain, context_domain, acyclic,
    description from link_type order by 1\""
```

**6 rows returned. Five are Block 1's seed; one — **`MEMBER_OF` — is the pack's.

| `link_kind` | `src_domain` | `dst_domain` | `context_domain` | `acyclic` | Rows (D-050) |
| --- | --- | --- | --- | --- | --- |
| `ASSEMBLY` | PRODUCT | PRODUCT | — | `t` | 210,890 |
| `MEMBER_OF` | PROCESS | PROCESS | — | `t` | 865 |
| `MERGE` | PRODUCT | PRODUCT | — | `t` | 0 |
| `PROCESSED_IN` | PRODUCT | EQUIPMENT | PROCESS | `f` | 0 |
| `SPLIT` | PRODUCT | PRODUCT | — | `t` | 0 |
| `TESTED_ON` | PRODUCT | EQUIPMENT | PROCESS | `f` | 78 |

`MEMBER_OF` is the pack's one added kind. Its description reads: pin is a member of a pin group (STDF PGR). PROCESS→PROCESS, acyclic, no context permitted. D-050's deferred question **D6** asks whether this belongs in the canonical Block 1 seed rather than in every customer's pack; that question stays open and is not settled here.

**Three registered kinds carry no rows** — `MERGE`, `SPLIT` and `PROCESSED_IN`. They are present because Block 1 seeded them, not because SCK uses them.

**Checks.**

- **The link total reconciles.** 210,890 + 865 + 78 = **211,833**, the `link` total D-050 records. The three unused kinds contribute nothing, which is why three kinds sum to the whole table.
- **Context nullability agrees with the guard.** `ASSEMBLY` and `MEMBER_OF` declare no `context_domain`, so `link_guard()` rejects any context on them — and D-050 measures both at 100% NULL context (210,890/210,890 and 865/865). `TESTED_ON` declares `context_domain = PROCESS` and D-050 measures 0/78 NULL, i.e. all 78 carry a context. Section 2 shows exactly one `TEST_PROG_VER` entity exists, so all 78 resolve to that single node.
- `acyclic` flags match Block 1 for all five seeded kinds.

**One divergence from the canonical seed, recorded not fixed.** `ASSEMBLY`'s description in this instance reads *many components -\> one assembled item (lens+sensor+pcb -\> module)*. Block 1 on [01 · Schema Spec](./01-schema-spec-v2.2.md) seeds it as *many components -\> one assembled item*, with no parenthetical. The other four seeded descriptions are byte-identical to Block 1. **The appended example is a camera-module one** — lens/sensor/PCB is the LG Innotek case's genealogy as [06 · Case Registry](./06-case-registry.md) describes it, not SCK's DIE→UNIT assembly. How it came to be in the `sck` instance is not established by anything read here. It is a description string, load-bearing on nothing, and it will travel with the dump under the ruled ALTER-in-place path. Flagged for a reader, not corrected, and not raised as a decision.

**What this section does NOT establish**, and the template asks for: whether **windows** are available for this source, the `window_end` rule, and the guard band (D-042). `link.window_start`/`window_end` do not exist at v2.0 — they arrive in Block 3 — so the question cannot be answered against this instance at all until the migration runs.

## 9 · Known limits hit — NOT YET PUBLISHED as a list, but one discrepancy is recorded here

**The SCK row on **[**06 · Case Registry**](./06-case-registry.md)** states 6.53 M rows loaded. That does not reconcile with the measured instance.** Summing every table in the v2.0 instance:

- From D-050: `entity` 418,503 + `link` 211,833 + `measurement` 634,779 + `param_def` 3,521 + `param_spec` 2,813 + `file_layout_set` 1 + `file_layout` 3,518 + `file_ref` 26 = **1,274,994**.
- From this page: `entity_type` 17 + `entity_key_def` 18 + `link_type` 6 = **41**.
- All eleven v2.0 base tables: **1,275,035 rows.**

That is roughly 5.25 M short of 6.53 M. **The likely explanation is that 6.53 M counts SCALAR\_BULK Parquet rows, which live in the object store and not in Postgres** — but that is a hypothesis, not a measurement, and nothing read here tests it. Recorded so the next person does not rediscover it by accident. Not fixed on 06, and not raised as a decision.

The pack lesson already promoted from this case, **L-SCK-01** (one `type_code` overloaded with two item kinds → invariant I14), is on 06 and is not restated here. **L-SCK-04**, cited by D-050 as the `param_def` disambiguation fix, is referenced in section 5 but has no published record on these pages that I read.

## 10 · As-built evidence — NOT YET PUBLISHED

D-050's measured v2.0 baseline is the closest thing on the record and is cited there, not copied here — 23 keys, read 2026-08-31 and 2026-09-01, every value matching `expected_counts.json`. Idempotency is asserted as proven on 06's registry row but its evidence is not published. Fills from the loader's run log and `CASE_REGISTRY_UPDATE.md`.

**Open, and deliberately left open.** D-050 defers the oracle's home to its own decision: `expected_counts.json` is the only correct copy and it lives in `TESTS/`, which is unversioned and outside every agent's reach. Until that decision closes, the gate for this pack has an authority no agent can audit.

## Provenance

Three readings, run by the owner on the DGX on **2026-09-01** and pasted verbatim into the session that wrote this page; the commands are printed above each section. `PG=postgres`, `DB=sck_raw`. Counts attributed to D-050 are cited from [that record](./05-D050-full-record.md) and were not re-measured here.

**Two limits on this page's authority.** The `entity_key_def` query did not select `version`, so **no claim is made about which key-definition version any row carries** — I3 makes that table append-only, so more than one version may exist per type and these readings cannot see it. And every row here is a snapshot at 2026-09-01; nothing keeps this page current as the instance changes.

Published under CORE-194 executing D-050. No credential value appears on this page.
