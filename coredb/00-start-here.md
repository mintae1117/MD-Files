# 00 · Start Here — New Engineer Onboarding Manual
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/378798159/00+Start+Here+New+Engineer+Onboarding+Manual
> pageId: 378798159 · lastModified: 8월 15, 2026 · saved: 2026-10-12

**From zero to your first safe query and your first data source.**

> **Read me first.** This manual is a *learning aid*, not the source of truth. The truth lives on the COREDB pages [01 · Schema Spec](./01-schema-spec-v2.2.md) · [02 · Runbook](./02-data-engineer-runbook.md) · [03 · Query Conventions](./03-backend-query-conventions.md) · [04 · Known Limits](./04-known-limits-deferred.md) · [05 · Decision Log](./05-decision-log.md) · [06 · Case Registry](./06-case-registry.md) · [07 · Field-Test Plan](./07-field-test-plan.md) · [08 · Operations](./08-operations.md). Whenever this manual and a live page disagree, **the live page wins**. Checked against the live pages on 2026-08-14 (schema v2.2). Column-level detail for all 18 tables is on the child page [00a · Table and Column Reference](./00a-table-column-reference.md). The **Real pack** examples come from the SCK and SEMES packs (page 06 + frozen Notion archive); those packs are pre-v2.1 — structures here are translated to v2.2 names, and load counts are as-built on the v1.1 `sck` instance, re-verified in field-test Test 1. 한국어판은 이 페이지의 하위 페이지에 있습니다.

## 0. Who this is for, and what you'll be able to do

You just joined as a data engineer. You know SQL and some Python, but you have never seen this database. After working through this manual you will be able to:

1. Explain in one minute why the Universal DB looks the way it does.
2. Find any value in the database (or in the Parquet files next to it) without guessing.
3. Write your first queries without breaking any of the house rules.
4. Follow the runbook when a new data file lands on your desk.
5. Know exactly what to do when something *doesn't* fit — which is the most important skill here.

Estimated time: read this in half a day, then spend your first two weeks on the exercises in Section 12.

## 1. The problem this database solves

AIBIZ works with manufacturing data from completely different worlds: a semiconductor etcher (SEMES), post-package test with STDF files (SCK), camera-module testers (LG Innotek), gas pipelines (KOGAS) — and whatever comes next.

The classic approach is one schema per customer: a `wafer` table here, a `test_result` table there, a `pipeline_sensor` table somewhere else. That works for exactly one project. Then the next customer arrives, and you fork everything: new tables, new loaders, new queries, new bugs. Five customers later you maintain five slightly different databases and nothing is reusable.

The Universal DB takes the opposite bet: **one schema that never changes per customer**. A new sensor, a new item type, a new naming scheme, a new relationship kind — all of these arrive as **rows in registry tables**, never as new columns or new tables. When onboarding a new domain requires only writing rows, the schema is doing its job. When a new case *forces* a schema change, that is called a **coverage defect** and gets logged and fixed through a formal decision — never patched quietly.

One thing to keep straight (decision D-026): the schema is *logically* universal but *physically* per-project. Every customer runs their own database instance (`sck`, `semes`, `innotek`, `kogas`, …), but every instance runs the **same unforked schema**.

## 2. The big idea in one paragraph

Instead of tables named after factory things (`wafer`, `lot`, `chamber`), the database stores factory things as **rows** in a small set of generic tables. *What kinds of things exist* is data (`entity_type`). *How each thing is named* is data (`entity_key_def`). *What gets measured* is data (`param_def`). *How things relate* is data (`link_type`). The schema itself only knows six concepts: **items, values, specs, files, links, and provenance**. That's why 18 tables can absorb an etcher, a test floor, and a gas pipeline without changing shape.

## 3. Six boxes, eighteen tables

Don't try to memorize 18 tables. Memorize six boxes; the tables sort themselves.

| Box | Question it answers | Tables |
| --- | --- | --- |
| **ITEM** | What things exist, and how are they named? | `entity_type`, `entity_key_def`, `entity` |
| **VALUE** | What was measured, and what is its value? | `param_def`, `param_condition`, `measurement`, `observation`, `param_concept`, `param_alias` |
| **SPEC** | What *should* the value be (limits, setpoints)? | `param_spec` |
| **FILE** | Where are the big raw files, and how do I read them? | `file_layout_set`, `file_layout`, `file_ref` |
| **LINK** | How do things relate across trees (routing, genealogy)? | `link`, `link_type` |
| **PROVENANCE** | Where did every row come from? Which schema is on disk? | `ingest_batch`, `measurement_audit`, `schema_migration` |

Three tables carry most of your daily work: `entity` (the things), `param_def` (the dictionary of measurements), `measurement` (the values). Start there.

### All 18 tables — what one row is

Read this once for orientation; you do not need to memorize it. **Every column of every table, with an example, is on the companion page **[**00a · Table and Column Reference**](./00a-table-column-reference.md) — that is where you look things up while writing SQL.

| Box | Table | One row is… | Primary key |
| --- | --- | --- | --- |
| ITEM | `entity_type` | a kind of item | `type_code` |
| ITEM | `entity_key_def` | ONE part of one type's naming recipe, for one source | (`type_code`, `source_code`, `version`, `seq`) |
| ITEM | `entity` | one tracked item | `entity_id` |
| VALUE | `param_def` | a kind of measurement — the Dictionary | `param_id` |
| VALUE | `param_condition` | one condition axis qualifying a parameter | (`param_id`, `condition_key`) |
| VALUE | `measurement` | one scalar value | (`entity_id`, `param_id`) |
| VALUE | `observation` | one time-keyed telemetry point on a standing asset | (`entity_id`, `param_id`, `ts`) |
| VALUE | `param_concept` | one canonical cross-source quantity | `concept_id` |
| VALUE | `param_alias` | one synonym that resolves to a parameter | (`param_id`, `alias`) |
| SPEC | `param_spec` | one spec band (limit or setpoint) for a param in a context | (`param_id`, `context_entity_id`, `spec_kind`, `valid_from`) |
| FILE | `file_layout_set` | one file-column layout, tagged with its lane | `layout_id` |
| FILE | `file_layout` | one column (channel or test) of a layout | (`layout_id`, `param_id`) |
| FILE | `file_ref` | a pointer to one Parquet file | `storage_uri` |
| LINK | `link` | one relationship between two items | `link_id` |
| LINK | `link_type` | one **allowed** relationship kind + its rules | `link_kind` |
| PROVENANCE | `ingest_batch` | one ingestion or correction batch | `batch_id` |
| PROVENANCE | `measurement_audit` | one superseded value, archived by a CORRECTION | *(none — append-only log)* |
| PROVENANCE | `schema_migration` | one applied DDL package | `version` |

Two things to notice already. Every table whose rows describe *what may exist* — `entity_type`, `entity_key_def`, `param_def`, `link_type`, `file_layout_set` — is a **registry**: onboarding a new source writes rows here and nothing else. And five tables carry a `batch_id` (`measurement`, `link`, `file_ref`, `param_spec`, `observation`), which is how any number in the database traces back to the file it came from.

## 4. The mental model: trees, domains, and links

### 4.1 Everything is an entity in a tree

Every tracked *thing* — a lot, a wafer, a test unit, a tool, a chamber, a recipe — is one row in `entity`. Entities form trees through `parent_id`: a wafer's parent is its lot, a chamber's parent is its tool.

Each entity has:

- `entity_id` — a meaningless number. **This is what you join on. Always.**
- `type_code` — what kind of thing it is (`LOT`, `WAFER`, `CHAMBER`, …). The list of valid types is itself data, in `entity_type`.
- `local_id` — its name *within* its parent ("W07").
- `natural_key` — the full human-readable path ("LOT2291/W07"). **Derived by the system — you never write it by hand.** Use it to *resolve* a human name to an `entity_id`, to display, or to prefix-filter (`LIKE 'LOT2291/%'`). Never to join.
- `attrs` — a JSON bag for descriptive stuff (start time, operator, equipment name). **Attrs are never identity.** More on this rule later, because it is the rule people break first.

> **Real pack (SCK).** A die's identity is built from two key parts — fuse-read X and Y coordinates, canon rule `DIGITS_PAD3` each — giving `local_id` `X057Y028` and full path `STTV7X/W11/X057Y028`. Positional, stable, no time or equipment anywhere. The part's electronic serial number is burned *at test* and differs between passes — which is exactly why it must not be identity; only the die coordinate links passes. And a naming subtlety that earns its keep: SCK's RUN code is kept `AS_IS` verbatim (`TEST_COD#<sessionTS>-<crc(filename)>`) because a lot can hold a base datalog and its `_rep` replicate with the same timestamp and pass — without the filename CRC in the key, one would silently overwrite the other.

### 4.2 Three separate trees (domains)

Entities live in exactly one of three **domains**. The domain is set once per *type*, on `entity_type.domain` (`VARCHAR(16) NOT NULL DEFAULT 'PRODUCT'`), and every entity inherits it from its type:

| Domain | What belongs in it | Example types |
| --- | --- | --- |
| **PRODUCT** | What is made, measured and shipped | LOT · WAFER · SITE · RUN · DIE · FAB\_LOT · TRAY · MODULE · UNIT |
| **EQUIPMENT** | What does the making or the measuring | TOOL · CHAMBER · TESTER · pipeline segment |
| **PROCESS** | The instructions and their versions | RECIPE · RECIPE\_VERSION · RECIPE\_STEP · TEST\_PROGRAM · TEST\_PROG\_VER |

The domain is not decoration — it decides exactly three things, and that is why it exists:

1. **Tree containment.** `entity_type.parent_type` stays inside one domain, so `parent_id` **never crosses a domain boundary**. A wafer's parent can never be a chamber. This is invariant I10, and the tree half of it is checked when you design the mapping (no trigger enforces it — the resolver builds the tree, so a wrong tree is a loader bug, not a rejected INSERT).
2. **Which relationships are legal.** Every rule in `link_type` is written in domains — next section. The *crossings* are trigger-enforced, fail-loud.
3. **Where specs can hang.** `param_spec.context_entity_id` must resolve to a PROCESS-domain entity — a trigger rejects anything else (D-023). This is why a limit attaches to a test-program version and never to a product.

**Choosing the domain for a new type**, in one question: is it *made or measured* (PRODUCT), does it *do the making or measuring* (EQUIPMENT), or is it *the instruction being followed* (PROCESS)?

A worked case: a **RECIPE\_STEP** is PROCESS (it is part of the instruction), a **CHAMBER** is EQUIPMENT (it executes), and the **RUN** where they meet is PRODUCT (it is the wafer's processing pass). Three domains, one event, joined by a link.

> **Real packs.** SEMES's tree is `LOT → WAFER → {SITE · RUN}` — SITE and RUN are *siblings* under WAFER (measurement sites and processing passes are different kinds of children). SCK carries **two** PRODUCT trees from one source: final test `LOT → RUN → UNIT`, and die origin `FAB_LOT → WAFER → DIE`, rebuilt from QFPROM fuse readback inside the very same STDF files. Two trees, one source, zero schema change.

### 4.3 Links cross the trees

So how does the database say "wafer W07 was etched in chamber C03 using recipe version V02"? With one row in `link`:

- `src_entity_id` = the wafer (PRODUCT)
- `dst_entity_id` = the chamber (EQUIPMENT)
- `link_kind` = `PROCESSED_IN`
- `context_entity_id` = the recipe version (PROCESS)

One row answers *which product, in which equipment, under which instructions*.

**Are link kinds free-form? No.** `link.link_kind` is a **foreign key to **`link_type` — a kind that is not registered there cannot be inserted at all. And each registered kind carries its own rules, which one generic guard enforces on every INSERT and UPDATE. These five are registered in the baseline DDL:

| link\_kind | src domain | dst domain | context domain | acyclic | What it means |
| --- | --- | --- | --- | --- | --- |
| `ASSEMBLY` | PRODUCT | PRODUCT | — | TRUE | many components → one assembled item |
| `MERGE` | PRODUCT | PRODUCT | — | TRUE | many source lots → one merged lot |
| `SPLIT` | PRODUCT | PRODUCT | — | TRUE | one source → many outputs, **only** when the output is not named under the source |
| `PROCESSED_IN` | PRODUCT | EQUIPMENT | PROCESS | FALSE | product processed in equipment; context = the RECIPE\_VERSION that ran |
| `TESTED_ON` | PRODUCT | EQUIPMENT | PROCESS | FALSE | product tested on equipment; context = the TEST\_PROG\_VER that ran |

What the guard checks, every time: the source's domain equals `src_domain` and the target's equals `dst_domain`; if a context is supplied, the kind must declare a `context_domain` (**NULL means context is forbidden for that kind**) and the context entity must be in it; and if `acyclic` is TRUE it walks the existing graph for that kind and rejects any edge that would close a cycle — so an assembly can never end up containing itself.

So the answer to "free or typed?" is: **typed, but the type system is rows.** Adding a new relationship kind is one INSERT into `link_type`, never a new bridge table — that is one of the universality tests on page 01. The rules are data, and the guard reads them at runtime.

Read the table as two families. *Routing* (PROCESSED\_IN, TESTED\_ON) crosses PRODUCT → EQUIPMENT and carries the PROCESS context that governed the event. *Lineage* (ASSEMBLY, MERGE, SPLIT) stays inside PRODUCT, takes no context, and is acyclic. One trap worth remembering: `SPLIT` is only for outputs *not* named under their source — ordinary dicing, where the child is named under the parent, is tree containment (`parent_id`), not a link.

`link` is the **only** legal way to relate things across domains, and the sole source of truth for all relationships (decision D-033). Multi-hop questions ("full ancestry of this module") are answered by a recursive query, or by a *derived* graph projection that is rebuilt from `link` and is always disposable.

> **Real packs.** *Why chamber can't be identity:* a SEMES FOUP's 25 wafers scatter across the etcher's six chambers — wafer W07 may run in C03 while W08 runs in C01, under the same recipe version. Chamber is a per-wafer PROCESSED\_IN link, never part of a name. *Genealogy at scale:* SCK's die-to-unit lineage is 210,890 ASSEMBLY links; 99.67% of units linked, and the unlinked remainder (dies whose fuses weren't burned yet) was **counted, not hidden** — fail-soft with numbers, never silent.

### 4.4 The dictionary: param\_def

Every kind of measurement is one row in `param_def` — the Dictionary. A parameter has:

- `source` + `native_id` — where it comes from and what the source calls it (`'SEMES'` + `'SVID12'`).
- `grain` — which entity level the value attaches to (WAFER-grain, LOT-grain, CHAMBER-grain…). Query at the wrong grain and you get nothing — not an error, just nothing. Check the grain first.
- `value_type` — FLOAT, STRING, BOOL, or CATEGORICAL.
- `storage_class` — the **lane**: where the value physically lives. This is the single most important column in the schema for you. Next section.

One SVID, one definition (invariant I9): a parameter's meaning lives *only* here. Synonyms (`param_alias`) and cross-source concepts (`param_concept`) *point at* definitions; they never duplicate them.

> **Real dictionary rows (SEMES pack).** `SVID12` RF Power (WAFER, TRACE+SCALAR) · `SVID14` Chamber Pressure (WAFER, TRACE+SCALAR) · `SVID31` Film Thickness (SITE, SCALAR) · `SVID88` Endpoint Signal (WAFER, TRACE) · `LOT-YLD` Yield (LOT, SCALAR). Five parameters, four different grain/lane combinations — this is why you check both before querying. And an I9 trap caught in the SCK pack: PTR test `t1680001` (FLOAT) and FTR test `f1680001` (BOOL) are **two different tests** that share a number — hence the `t`/`f` namespace prefix in `native_id`; a shared namespace would silently corrupt the Parquet column.

## 5. Where values live: the five lanes

A junior engineer's most common bug: `SELECT * FROM measurement WHERE param_id = …` returns zero rows, and they conclude data is missing. Usually the data is fine — it just lives in a different **lane**. `param_def.storage_class` tells you where:

| Lane | What it's for | Where the values are |
| --- | --- | --- |
| **SCALAR** | One value per (item, parameter): yields, bins, scores, states | `measurement` rows |
| **SCALAR\_BULK** | Huge homogeneous scalars: thousands of tests × 200k units (STDF) | Parquet file (one per parent, row per unit), pointed at by `file_ref`. **Zero** measurement rows, by design |
| **TRACE** | High-rate signals (faster than \~1 s period): 10 Hz chamber traces | Parquet file per traced entity, pointed at by `file_ref`. Raw samples **never** become rows |
| **TRACE+SCALAR** | Tool reports raw trace *and* scalar summaries of it | Summaries in `measurement`, raw in the file |
| **TIMESERIES** | Slow telemetry on standing assets (SCADA rate or slower): pipeline sensor, chamber health over months | `observation` rows, keyed by (entity, param, **ts**) |

Choosing a lane when onboarding: walk the decision tree in this order — *high-rate signal?* → TRACE. *Slow telemetry on a standing asset, needed in SQL?* → TIMESERIES. *High-volume homogeneous per-item scalars?* → SCALAR\_BULK. *Everything else* → SCALAR.

> **Real routing (SCK pack).** PTR/FTR mass → SCALAR\_BULK, one Parquet per RUN (datalog), row per unit, column per test — 2.4 GB of Parquet instead of hundreds of millions of rows. Bins and pass/fail → SCALAR: 634,779 measurement rows, exactly three per unit. STDF's own TSR summary statistics → deliberately **not stored** (I6: recompute from the PTRs, or they drift). On SEMES: the endpoint signal is raw-only TRACE; RF power and chamber pressure are TRACE+SCALAR — per-wafer summary as a row, 10 Hz raw in the file.

Why lanes exist: a 10 Hz × 200-channel trace unpivoted into rows is billions of useless index entries (invariant I5 forbids it). And hundreds of millions of STDF results as individual rows would drown Postgres for no benefit — a columnar Parquet per lot is both smaller and faster. Rows are for values you filter and join on; files are for bulk you scan with DuckDB/pandas.

The bridge between rows and files: `file_ref` (one row per Parquet file: which entity it's anchored to, which layout, what time window, which tier) and `file_layout` (which `param_id` is in which physical column — the file itself stores no names or units; the Dictionary does).

### The distinction that confuses everyone: measurement vs observation

- `measurement` — one scalar per (item, parameter). PK is `(entity_id, param_id)`. There is **no time in the key**: a wafer's thickness is one fact about that wafer. History on *products* is expressed by having more entities (each RUN, each wafer is its own entity).
- `observation` — telemetry on a **standing asset** (a chamber, a pipeline segment) where the *same* entity gets the *same* parameter measured forever. PK is `(entity_id, param_id, ts)` — time IS in the key, and that's the point.

If you find yourself wanting to create a fake entity per hour so you can store repeated measurements — stop. That's what `observation` exists for.

## 6. Speak factory, store model

The factory people you'll work with say PV, SV, SVID, travel, corner. Here's the translation table (keep it taped to your monitor):

| Factory says | The DB stores |
| --- | --- |
| **PV** (actual measured value) | `measurement` row (SCALAR), or Parquet behind `file_ref` (TRACE/SCALAR\_BULK) |
| **SV** (setpoint — the intent) | `param_spec` row, `spec_kind='SETPOINT'`, context = the RECIPE\_VERSION. Setpoints have exactly **one** home — never stored as measurements |
| **Test limit** | `param_spec` row, `spec_kind='LIMIT'`, context = the TEST\_PROG\_VER |
| **Recipe** | `RECIPE` → `RECIPE_VERSION` entities in the PROCESS domain |
| **SVID / test number** | `param_def.native_id` |
| **Test corner** (same test at 25 °C and 125 °C) | One `param_def` per corner + `param_condition` rows; a shared `param_concept` ties them together |
| **Lot travel / history** | `link` rows (PROCESSED\_IN / TESTED\_ON), context = the recipe/program version that ran |
| **BOM / made-from** | `link` rows (ASSEMBLY / MERGE / SPLIT) |
| **"Which file did this number come from?"** | `measurement.batch_id` → `ingest_batch.source_uri` — one join |

Note the deep pattern: **PV and SV never mix.** Actuals live in the VALUE box; intents live in the SPEC box, attached to the PROCESS entity that governs them. Pass/fail is always a *join* between the two, never a stored flag.

> **Real spec rows (SCK pack).** Limits attach to the TEST\_PROG\_VER entity: `t1234` (Iddq) LIMIT hi = 5.0e-6 A · `t1001` (Continuity VDD) LIMIT 0.2–0.8 V — 2,786 such rows in the load. When a second site runs the same test program with different limits, it onboards as its own TEST\_PROG\_VER entity with its own `param_spec` rows — different limits coexist as rows, no schema change.

## 7. Worked example: one wafer's day

Let's follow data end to end, using the **real SEMES pack's** entities and parameters. The story: lot **LOT2291**, wafer **7**, gets etched in chamber **C03** of tool **ETCH-03**, running recipe **ETCH-A** version **V03**. The tool logs a 10 Hz chamber-pressure trace plus a per-wafer pressure summary (`SVID14`, lane TRACE+SCALAR), and film thickness is measured afterwards at sites on the wafer (`SVID31`, grain SITE, lane SCALAR). (Ids and parameters are the pack's real rows; the specific values below are illustrative.)

**Registry rows (written once, when the SEMES source was onboarded — not per wafer):**

- `entity_type`: LOT (PRODUCT), WAFER (parent LOT), SITE and RUN (siblings under WAFER), TOOL (EQUIPMENT), CHAMBER (parent TOOL), RECIPE (PROCESS), RECIPE\_VERSION (parent RECIPE).
- `entity_key_def`: "a WAFER is named by field `wafer_no`, prefix `W`, rule `DIGITS_PAD2`" — so wafer 7 becomes `W07`, every time, from every file. Naming is *configuration*, not parser code.
- `param_def` (real pack rows): (`SEMES`, `SVID14`, "Chamber Pressure", grain WAFER, TRACE+SCALAR) and (`SEMES`, `SVID31`, "Film Thickness", grain SITE, SCALAR).
- `file_layout_set` + `file_layout`: a TRACE-lane layout; the `press` column in the Parquet is param SVID14.

**Rows written by the loader for this event:**

1. `ingest_batch`: one row — source SEMES, the file's URI, kind INGEST. Its `batch_id` gets stamped on everything below. (Provenance first, always.)
2. `entity`: resolve-or-create LOT2291 → its wafer (→ its SITE children when thickness arrives). `natural_key` becomes `LOT2291/W07` — built by the resolver from the naming recipe, never typed by the loader.
3. `link`: wafer → chamber `ETCH-03/C03`, kind PROCESSED\_IN, context = recipe-version entity `ETCH-A/V03`, with the occupancy window (`window_start`/`window_end`) if the source provides one.
4. `measurement`: two homes, because two grains — (wafer, SVID14) = the per-wafer pressure summary (the SCALAR half of TRACE+SCALAR), and (site, SVID31) = 512.3 nm thickness *on the SITE entity* `LOT2291/W07/S01`, not on the wafer. If this file is ever re-sent, the loader lands on the *same* rows with zero errors — and if a re-send carries a *different* value, the load **fails loudly** (only a CORRECTION batch may overwrite, and it archives the old value to `measurement_audit` first).
5. The trace: raw 10 Hz pressure samples go to a Parquet file; one `file_ref` row points at it (anchored to the traced entity, lane TRACE, `sampling_hz` = 10) — the TRACE half of TRACE+SCALAR.
6. The setpoint: `param_spec` (SVID14, context = `ETCH-A/V03`, SETPOINT) — written when the recipe version was registered, not per wafer.

**Now every question has one home:**

- "What was W07's mean pressure?" → `measurement` at wafer grain.
- "Show me the pressure curve" → find the Parquet via `file_ref`, read with DuckDB/pandas.
- "Was pressure on setpoint?" → compare against `param_spec` SETPOINT for V03.
- "Which chamber ran it, under which recipe?" → the `link` row.
- "Thickness uniformity across the wafer?" → the SITE-grain rows under W07 — grain in action.
- "Which file did 512.3 come from?" → `batch_id` → `ingest_batch.source_uri`.

And if wafer 7 is re-etched: RUN entities `LOT2291/W07/ETCH#1`, `…/ETCH#2` appear — the second pass may run in a *different* chamber, captured as structure (a new RUN with its own link), never as a timestamp key (I1).

Notice what did **not** happen: no `semes_wafer_etch` table was created. Another etcher customer tomorrow is more registry rows in the same schema.

### The same pattern at the test floor (SCK, 60 seconds)

Final test: LOT → RUN → UNIT. A **RUN is one STDF datalog** (`local_id` = `TEST_COD#<sessionTS>-<crc(filename)>`): `PART_ID` is a per-datalog counter, so without the RUN level, parts from two datalogs of one lot would collide — that's I1 solving a real bug, not theory.

Values: 211,593 units, each getting thousands of parametric tests — hundreds of millions of values. They do *not* become measurement rows: PTR/FTR results are **SCALAR\_BULK** — one Parquet per RUN (row per unit, column per test), 2.4 GB of Parquet total, zero measurement rows. Bins and pass/fail are SCALAR rows — 634,779 of them, three per unit. Test limits are 2,786 `param_spec` LIMIT rows at TEST\_PROG\_VER grain.

Genealogy: the same files carry each die's origin in QFPROM fuse readback, so a second PRODUCT tree `FAB_LOT → WAFER → DIE` is built from the same datalogs, plus 210,890 DIE→UNIT ASSEMBLY links. The payoff: die `STWGUX/W15/X085Y040` was tested **four times**, hard bin 5 every time — its whole retest history is one join through the links. Load: 26/26 files, \~40 GB, zero failures; a re-run produced identical counts (idempotency proven). Same six boxes, wildly different industry. *(Counts are as-built on the v1.1 *`sck` instance — field-test Test 1 re-verifies them on v2.2.)

## 8. The rules you must not break

The live 01 page lists 14 binding invariants (I1–I14) — the constitution. Count and read them there. Here is what they mean in practice, grouped the way you'll actually hit them:

**Identity (I1, I2, I3, I14) — the ones that hurt most when broken:**

- **Never build identity from time or equipment** (I1). A wafer measured Monday and re-measured Tuesday is *one* wafer, not two. Distinguish processing *passes* at the RUN level (`local_id = STEP + '#' + pass` — SEMES: `LOT2291/W07/ETCH#1`, `…/ETCH#2`), never by timestamp. Time and equipment go in `attrs`.
- **Never write **`natural_key` by hand (I2). Only the resolver writes it, from the naming recipes.
- **Never edit a naming recipe row** (I3). `entity_key_def` is append-only: naming changed? INSERT a new version and run an explicit re-key migration. Editing in place silently duplicates every entity on the next load.
- **One **`type_code`, one kind of thing (I14). If test lots and fab lots are both "LOT" split by an attrs marker, every generated view downstream silently mixes two different things. This actually happened: SCK's first build put OSAT test lots (keyed from `MIR.LOT_ID`, children RUNs) and fab lots (keyed from fuse text, children WAFERs) in one `type_code`, split by an attrs marker — every "lots" view blended the two. The fix was a separate `FAB_LOT` type; the 10 affected entities were retyped in place, no re-ingest needed. That's pack lesson L-SCK-01 — and why I14 is an invariant now.

**Reading & writing values (I4, I5, I6, I11):**

- **Join on **`entity_id`, always (I4). `natural_key` is for resolve/display/prefix only.
- **Raw traces never become rows** (I5). High-rate questions are answered from Parquet with DuckDB/pandas.
- **Never store a parent roll-up as a parent measurement** (I6). Lot mean of wafer values = compute at read, or a clearly-separate derived table. Stored roll-ups drift from their inputs.
- **One scalar per (item, parameter)** (I11). Repeated values on the same standing asset → `observation`. Repeated values on product → you're missing an entity level (RUN).

**Change discipline (I7, I8, I12, I13):**

- **Never DELETE registry rows** (I7) — deactivate with `is_active = FALSE`.
- **Everything derived is disposable** (I8) — wide tables, views, graph projections. Never a source of truth, never written back to.
- **Loads are idempotent; conflicting re-sends fail loudly** (I12). No silent overwrites, ever. Deliberate fixes go through a CORRECTION batch that archives the old value first.
- **Never guess** (I13). Missing naming recipe, unknown canonicalization rule, ambiguous timestamp → the load *fails with a clear error*. "Best-effort parsing" is a bug here, not a feature. All `attrs` timestamps are UTC ISO-8601 or the load fails.

**Structure (I9, I10):** one definition per parameter (I9); domains never cross via the tree, only via `link` (I10).

If a rule feels like it's blocking you, that is usually the rule *working* — see Section 11.

## 9. Before you write any SQL: the three checks

Tape this above the translation table.

**Check 1 — which schema version is THIS instance on?**

```sql
SELECT version FROM schema_migration ORDER BY applied_at DESC LIMIT 1;
-- table absent => instance is pre-v2.2
```

The spec describes the *target*, not necessarily the disk. Instances lag legally (the loaded `sck` instance is still on v1.1 until its migration — check the version board on page 08). Writing v2.2 SQL against a v1.1 instance wastes your afternoon.

**Check 2 — what lane is the parameter in?**

```sql
SELECT param_id, grain, value_type, storage_class
FROM param_def WHERE source = :src AND native_id = :nid;
```

A TRACE or SCALAR\_BULK parameter has zero `measurement` rows *by design*. Empty scalar query against a file-lane param = your bug, not missing data.

**Check 3 — what grain?** `param_def.grain` names the entity level the value attaches to. Wafer-grain values queried at lot level return nothing without a tree walk.

Then two standing habits: filter `is_active` on registries and links (soft-retired ≠ deleted), and when a user says "wafer 7" with no lot — **ask which lot**. Never guess.

## 10. Your first queries

Full cookbook with verified shapes is on the live 03 page; these four cover 80% of day one.

**Resolve a human name once, then use the id:**

```sql
SELECT entity_id FROM entity WHERE natural_key = 'LOT2291/W07';
```

**A scalar value with its provenance:**

```sql
SELECT m.value_num, b.source_uri
FROM measurement m
JOIN ingest_batch b ON b.batch_id = m.batch_id
WHERE m.entity_id = :wafer_id AND m.param_id = :param_id;
```

**Which chamber processed this wafer, under which recipe:**

```sql
SELECT eq.natural_key AS chamber, rv.natural_key AS recipe_version
FROM link l
JOIN entity eq ON eq.entity_id = l.dst_entity_id
LEFT JOIN entity rv ON rv.entity_id = l.context_entity_id
WHERE l.src_entity_id = :wafer_id AND l.link_kind = 'PROCESSED_IN' AND l.is_active;
```

**Lot-level roll-up, computed at read (never stored back):**

```sql
SELECT avg(m.value_num)
FROM entity child
JOIN measurement m ON m.entity_id = child.entity_id AND m.param_id = :param
WHERE child.parent_id = :lot_id;
```

**A real one from the SCK pack — per-die retest history across passes** (the pack's flagship query, translated from its v1.1 shape to v2.2's `link`; run it only against a migrated instance):

```sql
SELECT die.natural_key, count(DISTINCT run.entity_id) AS passes,
       array_agg(m.value_text ORDER BY run.local_id) AS hard_bins
FROM entity die
JOIN link e ON e.src_entity_id = die.entity_id AND e.link_kind = 'ASSEMBLY' AND e.is_active
JOIN entity u   ON u.entity_id = e.dst_entity_id
JOIN entity run ON run.entity_id = u.parent_id
JOIN measurement m ON m.entity_id = u.entity_id
JOIN param_def p ON p.param_id = m.param_id AND p.native_id = 'PRR.HARD_BIN'
WHERE die.type_code = 'DIE'
GROUP BY 1 HAVING count(DISTINCT run.entity_id) > 1;
-- walks DIE → ASSEMBLY link → UNIT → parent RUN; 4,249 dies have multi-pass history in the load
```

For file-lane data: find the file (`file_ref` by `anchor_entity_id` or time window), **check **`tier` first (non-HOT reads fail loudly on purpose), then read the Parquet with DuckDB or pandas, mapping columns through `file_layout`.

## 11. When new data arrives — and when it doesn't fit

### The runbook loop (page 02 — read it in full before your first source)

The golden rule: **design before code. Never start with the parser.**

1. **Survey the source** — what are the things, what are they named by, what gets measured at what grain/rate, are there limits/setpoints, corners, re-tests, what timezone.
2. **Draft the mapping** — entity tree, naming recipes, every parameter's lane, specs, relations.
3. **Get sign-off on the written mapping** (a pack spec, format on page 06). A mapping mistake caught at review costs one edit; caught after loading it costs a re-load and a correction cycle.
4. **Only then write the loader** — which must follow the loader contract on page 02: open an `ingest_batch` and stamp every row; resolve-or-create idempotently; identical re-sends land silently, differing re-sends fail loudly; validate timestamps; never commit unvalidated.
5. **Prove the gates**: full load green; re-run gives identical counts and zero errors; a deliberately mutated re-send fails and then succeeds as CORRECTION; counts reconcile against the source. Then register the pack on page 06.

### When something doesn't fit — the most important habit

One day you'll hit data the schema can't absorb as rows. This is *expected* — page 04 exists precisely for it. The discipline:

1. **Don't patch silently.** Don't add a column. Don't fork.
2. **Don't dump the shape into **`attrs`. Every "just put it in attrs" is itself a logged coverage defect.
3. **Log it on page 04**: which source, which shape, what you tried, which rule blocked you. Check the deferred-items table first — your case may already be there with a reopen trigger waiting for exactly your data.
4. The fix arrives as a ratified decision (D-###) on page 05, possibly superseding an older one.

This feels slow the first time. It is why after 48 decisions the schema is still one schema and not five forks — logging the defect *is* the fast path, long-term.

> **A real trip through this loop (SCK corners).** SCK's corner data — the same test run at different temperatures — didn't fit the schema of its day. It was **held out of the load** and logged as backlog item C-04, not squeezed into `attrs` and not hacked in with a local column. C-04 later became decision D-038 (the condition model: per-condition `param_def` + `param_condition` + shared concepts), and the held-out corner files are now field-test Test 2, waiting on the `sck` migration. Log → decide → load: the loop, working end to end.

### How change happens (so you don't fight the system)

- Pages 01–08 change only through decisions on page 05; the log is append-only, and wrong decisions get *superseded*, never edited away.
- Schema changes ship **only** as numbered migration scripts (dev → verify → promote, ending with a `schema_migration` row). **No hand-typed ALTERs on a live instance. Ever.** Not even small ones. Not even yours.
- Fail-loud errors reaching you (missing partition, non-HOT tier, guard rejection) are the system working as designed. The fix is the designed action (create the partition, restore-or-accept decision, fix the data), never a retry loop or a workaround.

## 12. Your first two weeks

**Day 1 — orientation.** Read this manual. Then skim live pages 01 (stop before the DDL) and 02. Learn the six boxes and the translation table by heart.

**Day 2 — the spec, for real.** Read all of page 01 including the DDL blocks and all 14 invariants. Run the three DDL blocks on a scratch Postgres 16 — watching the guards get created teaches more than reading. Keep [00a · Table and Column Reference](./00a-table-column-reference.md) open beside the DDL.

**Day 3 — hands on the tree.** On your scratch instance, hand-insert the Section 7 worked example: types, naming recipes, entities, one link, one measurement, one spec. Try to break it: a `/` in a `local_id`, two values in one measurement row, a PRODUCT context on a spec, an illegal link (wafer PROCESSED\_IN another wafer). Watch every guard and CHECK fire. (Insert entities through resolver-style SQL; remember you never hand-write `natural_key` in real life.)

**Day 4 — reading data.** Read page 03. Run the three checks against a real instance (ask which one you may read). Reproduce the cookbook queries. Deliberately query a SCALAR\_BULK param in `measurement` and observe the empty set — remember the feeling.

**Day 5 — lanes and files.** Open a real `file_ref`, join its `file_layout`, read the Parquet with DuckDB. Compute a roll-up at read.

**Week 2 — the working world.** Read pages 04 (know the deferred items — you *will* hit them), 05 (skim all 48 decisions; read D-026, D-030, D-033, D-034, D-038, D-042 closely), 06 (read the SCK pack row and lesson L-SCK-01), 07 and 08 (know the field-test gates and the migration discipline). Then take a small real task: a query for someone, or shadow whoever is running the current field test. Before proposing *anything* structural, learn the no-regression rule on the project instructions: decision-log scan, requirements walk, pack-lesson scan.

**The graduation test:** explain to another new hire why `measurement` has no timestamp in its key, and what to do instead in each case where you'd want one. If you can, you have the model.

## 13. Glossary

| Term | Meaning |
| --- | --- |
| **Entity** | Any tracked thing (lot, wafer, chamber, recipe version) — one row in `entity` |
| **Domain** | Which tree an entity lives in: PRODUCT, EQUIPMENT, or PROCESS |
| **Grain** | The entity level a parameter attaches to (WAFER-grain, LOT-grain) |
| **Lane / storage class** | Where a parameter's values physically live: SCALAR, SCALAR\_BULK, TRACE, TRACE+SCALAR, TIMESERIES |
| **Natural key** | Full derived path name (`LOT2291/W07`); resolve/display/prefix only |
| **Naming recipe** | `entity_key_def` rows saying how a type's `local_id` is built from source fields. (Unrelated to process recipes — two things share the word "recipe") |
| **Canonicalization** | Closed rule set normalizing source values into ids: AS\_IS, UPPER, DIGITS\_PAD2, DIGITS\_PAD3 |
| **Pack** | The written mapping spec + evidence for one onboarded source (page 06) |
| **Batch / provenance** | `ingest_batch` row every load opens; its id is stamped on every written row |
| **CORRECTION** | The only legal way to overwrite a value: a batch that archives the old value to `measurement_audit` first |
| **Coverage defect** | Data that won't fit as rows; logged on page 04, fixed by decision — never patched silently |
| **D-###** | A ratified decision on page 05 (append-only; superseded, never edited) |
| **I1–I14** | The binding invariants on page 01 — the constitution |
| **L-###** | A pack lesson on page 06 (L-SCK-01 became invariant I14) |
| **Concept** | Cross-source canonical quantity (`param_concept`), e.g. CHAMBER\_PRESSURE, with unit conversion via scale/offset on each `param_def` |
| **Corner / condition** | Same quantity measured under different conditions → one `param_def` per condition + `param_condition` rows, shared concept |
| **Guard** | A fail-loud database trigger enforcing an invariant (domain rules, lanes, cycles, spec contexts) |
| **Tier** | `file_ref` lifecycle: HOT / COLD / PURGED (purged = tombstone row survives) |
| **Instance** | One customer's physical database (`sck`, `semes`, …) — same schema, own migration schedule |

## 14. One-page cheat sheet

**The model:** things = `entity` rows in three domain trees (PRODUCT / EQUIPMENT / PROCESS); cross-tree facts = `link` rows whose kind is registered in `link_type`; measured values = `measurement` / `observation` / Parquet behind `file_ref`, per the lane in `param_def.storage_class`; intents = `param_spec`; every row stamped with `batch_id`.

**Before any SQL:** ① instance version (`schema_migration`) ② parameter lane ③ parameter grain.

**Iron rules:** join on `entity_id` · identity never from time/equipment · natural\_key is derived, never hand-written · traces never become rows · roll-ups computed, never stored · setpoints only in `param_spec` · no silent overwrites (CORRECTION only) · no DELETE on registries (`is_active=FALSE`) · derived = disposable · don't fit? → log on page 04, never into `attrs` · schema changes only via numbered migrations + D-### · when ambiguous: **fail loud, never guess**.

**Where to look:** [00a column reference](./00a-table-column-reference.md) · [01 spec & invariants](./01-schema-spec-v2.2.md) · [02 onboarding runbook & loader contract](./02-data-engineer-runbook.md) · [03 query cookbook](./03-backend-query-conventions.md) · [04 known limits (log hits here)](./04-known-limits-deferred.md) · [05 decision log](./05-decision-log.md) · [06 packs](./06-case-registry.md) · [07 field tests](./07-field-test-plan.md) · [08 operations](./08-operations.md).

**When stuck:** check the failure catalog (page 02), the deferred items (page 04), the decision log (page 05) — in that order. Then ask. Asking beats guessing, every time.
