# 00a · Table and Column Reference — every table, every column (v2.2)
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/380174341/00a+Table+and+Column+Reference+every+table+every+column+v2.2
> pageId: 380174341 · lastModified: 8월 15, 2026 · saved: 2026-10-12

**Every table, every column, with an example — schema v2.2 (18 tables).**

> Companion to [00 · Start Here](./00-start-here.md). This page is a **reading aid built from the canonical v2.2 DDL** on [01 · Schema Spec](./01-schema-spec-v2.2.md) (three blocks, verified on PostgreSQL 16.13, 2026-08-11). Column names, types, keys and CHECKs come from that DDL; the examples are illustrative, drawn from the SCK and SEMES packs where possible. **If this page and page 01 ever disagree, page 01 wins** — and that is a bug on this page worth reporting. Built 2026-08-14.

## How to read a column

- **PK / FK** — a PK column is part of the row's identity; an FK column points at another table's PK. Join on ids, never on names (I4).
- **CHECK / UNIQUE** — rules the database enforces itself. When one fires you get a loud error, not a silently wrong row. That is the design.
- **Trigger-enforced** — a rule too complex for a CHECK, enforced by a fail-loud guard function (domain rules, lanes, cycles, spec contexts).
- **Nullable** — legal to leave empty. Where "NULL means something" the meaning is spelled out in the column's row.

## The three domains

Every `entity_type` row carries a `domain`, and every entity inherits it from its type. The column is `VARCHAR(16) NOT NULL DEFAULT 'PRODUCT'` — three values, fixed by convention.

| Domain | What belongs in it | Example types |
| --- | --- | --- |
| **PRODUCT** | What is made, measured and shipped | `LOT · WAFER · SITE · RUN · DIE · FAB_LOT · TRAY · MODULE · UNIT` |
| **EQUIPMENT** | What does the making or the measuring | `TOOL · CHAMBER · TESTER · pipeline segment` |
| **PROCESS** | The instructions and their versions | `RECIPE · RECIPE_VERSION · RECIPE_STEP · TEST_PROGRAM · TEST_PROG_VER` |

**The domain decides three things** — this is the whole reason it exists:

1. **Tree containment.** `entity_type.parent_type` stays inside one domain, so a wafer's parent can never be a chamber (I10).
2. **Which links are legal.** Every `link_type` rule is written in domains — see the next section.
3. **Where specs can hang.** `param_spec.context_entity_id` must resolve to a PROCESS-domain entity, enforced by a trigger (D-023).

**Choosing a domain for a new type**, in one question: is it *made or measured* (PRODUCT), does it *do the making or the measuring* (EQUIPMENT), or is it *the instruction being followed* (PROCESS)?

> **Honest note.** The three values are a closed convention, not a CHECK constraint — the DDL does not restrict the column's text. A typo'd domain would be accepted by `entity_type` and then show up as a fail-loud guard error on the first link that touches it, not at type registration. Registering types is a reviewed step, so this has never bitten; if you want it hardened, that is a decision to raise, not a silent patch.

## Link kinds are NOT free-form

`link.link_kind` is a **foreign key to **`link_type`. A kind that is not registered there cannot be inserted — the FK rejects it. And each registered kind carries its own rules, which one generic guard enforces on every INSERT and UPDATE:

| link\_kind | src domain | dst domain | context domain | acyclic | Meaning |
| --- | --- | --- | --- | --- | --- |
| `ASSEMBLY` | PRODUCT | PRODUCT | — | TRUE | many components → one assembled item |
| `MERGE` | PRODUCT | PRODUCT | — | TRUE | many source lots → one merged lot |
| `SPLIT` | PRODUCT | PRODUCT | — | TRUE | one source → many outputs, ONLY when the output is not named under the source |
| `PROCESSED_IN` | PRODUCT | EQUIPMENT | PROCESS | FALSE | product processed in equipment; context = the RECIPE\_VERSION that ran |
| `TESTED_ON` | PRODUCT | EQUIPMENT | PROCESS | FALSE | product tested on equipment; context = the TEST\_PROG\_VER that ran |

**What the guard checks, every time:**

- the source entity's domain equals `src_domain`, and the target's equals `dst_domain` — otherwise a loud error naming both;
- if a context is supplied, the kind must declare a `context_domain` (**NULL means context is forbidden for that kind**) and the context entity must be in it;
- if `acyclic` is TRUE, it walks the existing graph for that kind and rejects any edge that would close a cycle — so an assembly can never contain itself.

**Adding a new relationship kind is one INSERT into **`link_type` — never a new bridge table. That is one of the universality tests on page 01: a new relation type must be *rows*, not schema. The rules are data, so changing a rule changes what every future link is allowed to do.

**Two families, one table.** Routing (`PROCESSED_IN`, `TESTED_ON`) crosses PRODUCT → EQUIPMENT and carries the PROCESS context that governed the event. Lineage (`ASSEMBLY`, `MERGE`, `SPLIT`) stays inside PRODUCT, takes no context, and is acyclic. Note `SPLIT` is only for outputs *not* named under their source — ordinary dicing, where the child is named under the parent, is tree containment (`parent_id`), not a link.

## The 18 tables at a glance

| Box | Table | One row is… |
| --- | --- | --- |
| ITEM | `entity_type` | a kind of item |
| ITEM | `entity_key_def` | ONE part of one type's naming recipe, for one source |
| ITEM | `entity` | one tracked item |
| VALUE | `param_def` | a kind of measurement — the Dictionary |
| VALUE | `param_condition` | one condition axis qualifying a parameter (D-038) |
| VALUE | `measurement` | one scalar value |
| VALUE | `observation` | one time-keyed telemetry point on a standing asset (D-035) |
| VALUE | `param_concept` | one canonical cross-source quantity (D-036) |
| VALUE | `param_alias` | one synonym that resolves to a parameter (D-036) |
| SPEC | `param_spec` | one spec band (limit or setpoint) for a parameter in a context |
| FILE | `file_layout_set` | one file-column layout, tagged with its lane |
| FILE | `file_layout` | one column (channel or test) of a layout |
| FILE | `file_ref` | a pointer to ONE Parquet file — the bridge from rows to bulk |
| LINK | `link` | one relationship between two items — a routing event or a lineage edge |
| LINK | `link_type` | one ALLOWED relationship kind, and the rules that police it |
| PROVENANCE | `ingest_batch` | one ingestion or correction batch — the provenance of every row it wrote |
| PROVENANCE | `measurement_audit` | one superseded value, archived by a CORRECTION batch (append-only) |
| PROVENANCE | `schema_migration` | one applied DDL package — the provenance of the schema itself (D-045) |

---

## Table by table, column by column

### ITEM

#### `entity_type` — one row is a kind of item

*PK: type\_code*

| Column | What it is | Example |
| --- | --- | --- |
| `type_code` | The kind's code. ≤16 chars (D-027). Generated views filter on this | 'WAFER' |
| `parent_type` | FK → entity\_type: which kind is its tree parent. NULL = root kind | 'LOT' (for WAFER) |
| `domain` | PRODUCT \| EQUIPMENT \| PROCESS. NOT NULL, defaults to PRODUCT | 'EQUIPMENT' for CHAMBER |
| `description` | Free text for humans | 'wafer inside a fab lot' |

- `parent_type` must stay inside one domain — that is the tree half of I10, checked when you design the mapping (no trigger enforces it).
- One `type_code` = one item kind (I14). Two kinds keyed from different source fields are two types.

#### `entity_key_def` — one row is ONE part of one type's naming recipe, for one source

*PK: (type\_code, source\_code, version, seq) · APPEND-ONLY (I3)*

| Column | What it is | Example |
| --- | --- | --- |
| `type_code` | Which kind this recipe names | 'DIE' |
| `source_code` | Which source it applies to. '\*' = default; a specific code wins | 'SCK' |
| `version` | Recipe version. New naming = new version + explicit re-key migration | 1 |
| `seq` | Position of this part in a composite key | 1, then 2 |
| `field_name` | The source field to read | 'DIE\_X' (fuse PTR t850021) |
| `prefix` | Literal glued in front of the canonicalized value | 'X' |
| `canon_rule` | Closed set (I13): AS\_IS \| UPPER \| DIGITS\_PAD2 \| DIGITS\_PAD3 | 'DIGITS\_PAD3' → 57 becomes '057' |

- SCK's DIE uses two rows — seq 1 ('X', PAD3) and seq 2 ('Y', PAD3) — which build `local_id` `X057Y028`.
- A missing recipe for an incoming type FAILS the load. Never guess (I13).

#### `entity` — one row is one tracked item

*PK: entity\_id · UNIQUE: natural\_key · UNIQUE NULLS NOT DISTINCT (type\_code, parent\_id, local\_id)*

| Column | What it is | Example |
| --- | --- | --- |
| `entity_id` | Meaningless BIGINT identity. JOIN ON THIS, ALWAYS (I4) | 40217 |
| `type_code` | FK → entity\_type: what kind of thing it is | 'WAFER' |
| `parent_id` | FK → entity (self): the single containment/naming parent. NULL = root | the LOT's entity\_id |
| `local_id` | Name inside the parent. CHECK: must not contain '/' | 'W07' |
| `natural_key` | Derived: parent.natural\_key \|\| '/' \|\| local\_id. Resolver-only (I2) | 'LOT2291/W07' |
| `attrs` | JSONB descriptive bag — NEVER identity. Times UTC ISO-8601, keys end \_at (D-044) | {"started\_at":"2026-08-11T02:14:00Z"} |
| `created_at` | When the ROW was created (DB clock) — not when the event happened | 2026-08-11 05:00Z |

- `UNIQUE NULLS NOT DISTINCT` on (type\_code, parent\_id, local\_id) means two ROOT items with the same name collide instead of silently duplicating.
- Nothing in the DDL forces a child's parent to match `entity_type.parent_type` — the resolver is responsible for building the tree correctly.

### VALUE

#### `param_def` — one row is a kind of measurement — the Dictionary

*PK: param\_id · UNIQUE (source, native\_id)*

| Column | What it is | Example |
| --- | --- | --- |
| `param_id` | Identity PK — what measurement/file\_layout rows point at | 102 |
| `source` | Source-system namespace | 'SEMES' |
| `native_id` | What the source calls it. UNIQUE together with source | 'SVID14', 't1001', 'PRR.HARD\_BIN' |
| `name` | Human-readable name | 'Chamber Pressure' |
| `unit` | Unit as received from the source | 'mTorr' |
| `grain` | FK → entity\_type: the entity level values attach to. CHECK THIS FIRST | 'WAFER' / 'SITE' / 'UNIT' |
| `value_type` | FLOAT \| STRING \| BOOL \| CATEGORICAL | 'FLOAT' |
| `storage_class` | THE LANE: SCALAR \| SCALAR\_BULK \| TRACE \| TRACE+SCALAR \| TIMESERIES | 'TRACE+SCALAR' |
| `is_active` | Soft-retire here; never DELETE a param with measurements (I7) | TRUE |
| `concept_id` | FK → param\_concept: the cross-source quantity this is (D-036) | 7 = CHAMBER\_PRESSURE |
| `to_canonical_scale / _offset` | Linear transform to the concept's canonical unit. Both or neither (CHECK, D-040) | 0.133322 / 0  (mTorr → Pa) |

- One SVID, one definition (I9). Aliases and concepts POINT AT this row; they never copy its meaning.
- `grain` and `storage_class` together decide every future query. Choose them deliberately at mapping review.

#### `param_condition` — one row is one condition axis qualifying a parameter (D-038)

*PK: (param\_id, condition\_key)*

| Column | What it is | Example |
| --- | --- | --- |
| `param_id` | FK → param\_def: the per-condition parameter this qualifies | the 't1001@125C' param |
| `condition_key` | The condition axis | 'TEMP\_C', 'VDD\_V', 'FREQ\_HZ' |
| `value_num / value_text` | The condition's value — exactly ONE is populated (CHECK) | 125 |

- Two axes = two rows (TEMP\_C=125 and VDD\_V=1.8) on the same `param_id`.
- Corners of one quantity are separate `param_def` rows sharing one `param_concept` — never one param with a mutable condition.

#### `measurement` — one row is one scalar value

*PK: (entity\_id, param\_id) — NO TIME IN THE KEY (I11)*

| Column | What it is | Example |
| --- | --- | --- |
| `entity_id` | FK → entity: the item this value belongs to, at the param's grain | SITE 'LOT2291/W07/S01' |
| `param_id` | FK → param\_def: which measurement this is | SVID31 (Film Thickness) |
| `value_num / value_text / value_bool` | The value — exactly ONE column is populated (CHECK num\_nonnulls = 1) | 512.3 |
| `captured_at` | When the SOURCE measured it. Nullable; not part of the key | 2026-08-11T02:15:03Z |
| `batch_id` | FK → ingest\_batch: which load wrote this row. One join to the file | 884 |

- A re-send with an identical value lands on this same row. A DIFFERENT value fails loudly unless the batch is CORRECTION (I12).
- Want the same value again over time? On a standing asset that's `observation`; on product it means you are missing an entity level (RUN).

#### `observation` — one row is one time-keyed telemetry point on a standing asset (D-035)

*PK: (entity\_id, param\_id, ts) · PARTITION BY RANGE (ts), monthly*

| Column | What it is | Example |
| --- | --- | --- |
| `entity_id` | FK → entity: the standing asset being monitored | CHAMBER 'ETCH-03/C03' |
| `param_id` | FK → param\_def, whose storage\_class is TIMESERIES | pipeline pressure |
| `ts` | The sample time — IN THE KEY. This is the whole point of the table | 2026-08-11T02:15:00Z |
| `value_num` | NOT NULL. Numeric only in v1 — status/boolean channels are deferred | 4.21 |
| `batch_id` | FK → ingest\_batch: provenance, same as everywhere else | 902 |

- Cadence boundary: SCADA-rate and slower belongs here; anything faster stays TRACE in Parquet (I5).
- Missing partition = fail-loud error. Creating it is the designed action, not a workaround.

#### `param_concept` — one row is one canonical cross-source quantity (D-036)

*PK: concept\_id · UNIQUE: concept\_code*

| Column | What it is | Example |
| --- | --- | --- |
| `concept_id` | Identity PK — what param\_def.concept\_id points at | 7 |
| `concept_code` | The canonical code, UNIQUE | 'CHAMBER\_PRESSURE' |
| `name` | Human-readable name | 'Chamber pressure' |
| `canonical_unit` | The unit every mapped parameter converts INTO | 'Pa' |
| `description` | Free text | 'absolute pressure in the process chamber' |

- This is what lets one query return SEMES SVID14 and KOGAS PT-201 together, unit-consistent, with zero per-source SQL.
- A concept never carries a value or a setpoint — it only names the quantity (I9 / D-031).

#### `param_alias` — one row is one synonym that resolves to a parameter (D-036)

*PK: (param\_id, alias)*

| Column | What it is | Example |
| --- | --- | --- |
| `param_id` | FK → param\_def: what this synonym resolves to | 102 |
| `alias` | The synonym users actually type or the source actually sends | '압력', 'PRESS', 'SVID12' |
| `kind` | SYNONYM \| LOCALE \| LEGACY | 'LOCALE' for '압력' |

- Aliases resolve; they never define. The meaning stays in `param_def` (I9).
- Field-test gate: '압력' / 'PRESS' / the native id must all resolve to the same `param_id`.

### SPEC

#### `param_spec` — one row is one spec band (limit or setpoint) for a parameter in a context

*PK: (param\_id, context\_entity\_id, spec\_kind, valid\_from) · one ACTIVE band per (param, context, kind)*

| Column | What it is | Example |
| --- | --- | --- |
| `param_id` | FK → param\_def: which parameter is bounded | t1001 (Continuity VDD) |
| `context_entity_id` | FK → entity: the governing node. MUST be PROCESS domain — trigger-enforced (D-023) | TEST\_PROG\_VER entity |
| `spec_kind` | LIMIT (a test limit) \| SETPOINT (the SV, D-031) | 'LIMIT' |
| `lo_num / hi_num` | Band bounds. CHECK: lo ≤ hi when both present | 0.2 / 0.8 V |
| `target_num` | The target value — this is where a setpoint's number lives | 45 (mTorr setpoint) |
| `src_unit / src_scale` | Unit and scale as received, kept for provenance | 'V' / 0 |
| `valid_from` | When this band took effect. Part of the PK, so history is kept (D-034) | 2026-07-01T00:00Z |
| `is_active` | Exactly one active band per (param, context, kind) — partial UNIQUE index | TRUE |
| `captured_at / batch_id` | Source time / which load wrote it | 884 |

- A revision is ONE transaction: UPDATE the old row `is_active = FALSE`, then INSERT the new band (D-034). Never edit a band in place.
- Two sites with different limits for one test program = two TEST\_PROG\_VER contexts, two rows. No schema change.

### FILE

#### `file_layout_set` — one row is one file-column layout, tagged with its lane

*PK: layout\_id*

| Column | What it is | Example |
| --- | --- | --- |
| `layout_id` | The layout's id (assigned INT), referenced by file\_ref and file\_layout | 3 |
| `lane` | TRACE \| SCALAR\_BULK (CHECK). Immutable after insert — trigger-frozen | 'SCALAR\_BULK' |
| `time_column` | In-file timestamp column for irregular traces. NULL = uniformly sampled (D-041) | NULL, or 'ts' |
| `description` | Free text | 'SCK program STTV7X columnar layout' |

- One layout per file SHAPE, not per file. SCK uses one layout per TEST\_PROG\_VER; its columns accumulate as the union across that program's datalogs.
- If `time_column` is NULL the timing comes from `file_ref.sampling_hz` instead.

#### `file_layout` — one row is one column (channel or test) of a layout

*PK: (layout\_id, param\_id) · UNIQUE (layout\_id, column\_name)*

| Column | What it is | Example |
| --- | --- | --- |
| `layout_id` | FK → file\_layout\_set: which layout this column belongs to | 3 |
| `param_id` | FK → param\_def: which parameter this physical column carries | t1001's param\_id |
| `column_name` | The physical column name inside the Parquet file | 't1001' |

- UNIQUE (layout\_id, column\_name): two parameters can never share one physical column.
- This table stores NO names and NO units — that meaning lives only in `param_def` (I9). The file is dumb; the Dictionary is smart.

#### `file_ref` — one row is a pointer to ONE Parquet file — the bridge from rows to bulk

*PK: storage\_uri · CHECK file\_ref\_lane\_shape (lane-only columns must match the lane)*

| Column | What it is | Example |
| --- | --- | --- |
| `storage_uri` | Where the file is. This is the PK — one row per file | 'file://…/sck/\<LOT\>/\<RUN\>.parquet' |
| `lane` | TRACE \| SCALAR\_BULK. Must match its layout's lane — trigger-enforced | 'SCALAR\_BULK' |
| `anchor_entity_id` | What the file hangs off: TRACE = the traced entity; SCALAR\_BULK = the parent | the RUN entity |
| `layout_id` | FK → file\_layout\_set: how to read its columns | 3 |
| `row_grain / row_key_column / row_count` | SCALAR\_BULK only: what one row is, its id column, how many rows | 'UNIT' / 'entity\_id' / 8141 |
| `sampling_hz / sample_count` | TRACE only: the sampling rate and how many samples | 10 / 18000 |
| `window_start / window_end` | Time span the file covers. CHECK: ordered | 02:14Z → 02:44Z |
| `tier` | HOT \| COLD \| PURGED (CHECK). Readers FAIL LOUDLY on non-HOT (D-043) | 'HOT' |
| `tier_changed_at` | When the tier last changed | 2026-08-01T00:00Z |
| `captured_at / batch_id` | Source time / which load wrote it | 884 |

- Check `tier` BEFORE reading. A PURGED file keeps its row forever as a tombstone — the provenance survives even though the bytes are gone (D-043).
- The lane-shape CHECK means you physically cannot set `sampling_hz` on a SCALAR\_BULK row, or `row_grain` on a TRACE row.

### LINK

#### `link` — one row is one relationship between two items — a routing event or a lineage edge

*PK: link\_id · uq\_link\_natural UNIQUE NULLS NOT DISTINCT (src, dst, link\_kind, context) ← the loader's ON CONFLICT target*

| Column | What it is | Example |
| --- | --- | --- |
| `link_id` | Surrogate identity PK (added by D-034) | 55021 |
| `src_entity_id` | FK → entity: the 'from' item. Its domain must match link\_type.src\_domain | the DIE / the WAFER |
| `dst_entity_id` | FK → entity: the 'to' item. CHECK: src ≠ dst | the UNIT / the CHAMBER |
| `link_kind` | FK → link\_type: MUST be a registered kind. Unregistered = insert fails | 'ASSEMBLY', 'PROCESSED\_IN' |
| `context_entity_id` | FK → entity: the governing PROCESS node. Allowed ONLY if the kind declares a context\_domain | 'ETCH-A/V03' |
| `attrs` | JSONB: qty, slot, … Never identity (I1) | {"slot": 7} |
| `is_active` | Soft-retire a wrong edge; never DELETE (I7). Filter on this in queries | TRUE |
| `window_start / window_end` | Occupancy window — CONTEXT, never identity (D-042). CHECK: ordered | 02:14Z → 02:44Z |
| `ts_window` | STORED generated tstzrange(start, end, '\[)'). GiST-indexed with (dst, kind) | \['02:14Z','02:44Z') |
| `captured_at / batch_id` | Source time / which load wrote it | 884 |

- The window is NOT in the unique key — so a re-send with a different window is a value conflict and fails loudly, needing a CORRECTION batch (D-042).
- One generic trigger validates src/dst/context domains and per-kind acyclicity on every insert and update.

#### `link_type` — one row is one ALLOWED relationship kind, and the rules that police it

*PK: link\_kind — this table is why link kinds are not free-form*

| Column | What it is | Example |
| --- | --- | --- |
| `link_kind` | The kind's code, ≤16 chars. `link.link_kind` is a FK to this | 'PROCESSED\_IN' |
| `src_domain` | Which domain the source entity MUST be in | 'PRODUCT' |
| `dst_domain` | Which domain the target entity MUST be in | 'EQUIPMENT' |
| `context_domain` | Which domain the context must be in. NULL = context FORBIDDEN for this kind | 'PROCESS' (NULL for ASSEMBLY) |
| `acyclic` | TRUE = the guard walks the graph and rejects any edge that would close a cycle | TRUE for ASSEMBLY |
| `description` | What the kind means, in words | 'product processed in equipment; context = the RECIPE\_VERSION' |

- A NEW relationship kind is ONE ROW here — never a new bridge table. That is one of the universality tests on page 01.
- The rules are DATA: the guard reads this row and enforces it. Change the rule, and every future link obeys the new one.

### PROVENANCE

#### `ingest_batch` — one row is one ingestion or correction batch — the provenance of every row it wrote

*PK: batch\_id · stamped on measurement, link, file\_ref, param\_spec, observation*

| Column | What it is | Example |
| --- | --- | --- |
| `batch_id` | Identity PK. Every row the loader writes carries this | 884 |
| `source_code` | Which source system this batch came from | 'SCK' |
| `source_uri` | The file or feed of origin — this is what answers 'which file did this come from' | 'file://…/STTV7X\_1\_20260711.stdf' |
| `kind` | INGEST (normal) \| CORRECTION (the only legal way to overwrite a value) | 'INGEST' |
| `started_at / finished_at` | Opened when the load begins; finished\_at is set on commit | 05:00Z / 05:42Z |
| `note` | Free text for the operator | 'corner load, second attempt' |

- Open the batch FIRST, before writing anything else. Provenance is not an afterthought — it is step 1 of the loader contract.
- An unfinished batch (finished\_at IS NULL) is how you spot a load that died mid-flight.

#### `measurement_audit` — one row is one superseded value, archived by a CORRECTION batch (append-only)

*No PK — an append-only archive log*

| Column | What it is | Example |
| --- | --- | --- |
| `entity_id / param_id` | Which measurement was overwritten | (SITE id, SVID31) |
| `value_num / value_text / value_bool` | The OLD value — what the row held before the correction | 512.3 |
| `captured_at` | The old row's capture time | 2026-08-11T02:15:03Z |
| `batch_id` | The batch that originally wrote the old value | 884 |
| `superseded_at` | When it was archived. Defaults to now() | 2026-08-13T09:11Z |
| `superseded_by_batch` | FK → ingest\_batch: the CORRECTION batch that overwrote it | 991 |

- The archive row is written BEFORE the overwrite. If this table is empty, no value has ever been silently changed.
- This is the table a field-test gate reads: mutate a value, watch the load fail, run a CORRECTION, then find the old value here.

#### `schema_migration` — one row is one applied DDL package — the provenance of the schema itself (D-045)

*PK: version · instance version = latest row by applied\_at*

| Column | What it is | Example |
| --- | --- | --- |
| `version` | The schema version this package brought the instance to | 'v2.2' |
| `applied_at` | When it was applied. The LATEST row is the instance's version | 2026-08-11T00:00Z |
| `script_sha256` | Hash of exactly what ran — no ambiguity about which script | '9f2c…' |
| `decision_ids` | Which ratified decisions this package implements | 'D-038, D-040, D-041, D-042, D-043, D-045' |
| `note` | What the package did, in words | 'close-out package' |

- THE TABLE BEING ABSENT means the instance is pre-v2.2. That is Check 1 before you write any SQL.
- Rows only ever arrive from numbered migration scripts. A hand-typed ALTER leaves no row here — and that is exactly why it is banned.

---

## When this page is wrong

It is a reading aid, so it can drift. The live DDL on page 01 is the truth. If you find a mismatch, say so — it gets fixed here as a wording edit, and if the *spec* is what is wrong, that is a decision on [05 · Decision Log](./05-decision-log.md), never a silent patch.
