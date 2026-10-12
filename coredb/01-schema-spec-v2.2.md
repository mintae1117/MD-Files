# 01 · Schema Spec — v2.2 (Committed Baseline)
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/376995857/01+Schema+Spec+v2.2+Committed+Baseline
> pageId: 376995857 · lastModified: 9월 20, 2026 · saved: 2026-10-12

> **Committed baseline — schema v2.2, 18 tables, invariants I1–I14** (D-030 base + D-033–D-036 + close-out package D-037–D-048, ratified 2026-08-11). This page describes what is ratified; it changes only via a ratified `D-###` in the [Decision Log](./05-decision-log.md). The full v2.2 DDL stack below was executed and verified on PostgreSQL 16.13 on 2026-08-11 before ratification. History (v1.x/v2.0/v2.1 pages, proposals P-001–P-005, outside review R-001) lives in the frozen Notion archive — see the [Decision Log](./05-decision-log.md) page for links.

This is the engineer's reference: the objective, the invariants, the full data model with DDL, and the column dictionary. **How to work with it day-to-day** lives in [02 · Data Engineer Runbook](./02-data-engineer-runbook.md) and [03 · Backend Query Conventions](./03-backend-query-conventions.md).

## Objective

One schema that can absorb **any** industrial manufacturing domain — etcher, post-package test, camera-module test, solar tabber, gas pipeline, and whatever comes next — **without a schema change per domain or per customer**. New sensors, item types, and naming schemes arrive as **rows in registry tables**, never as new columns or forked tables. Many source systems coexist in the *same* database.

**Deployment model (D-026): logically universal, physically per-project.** The schema is one and never forks, but each customer/project runs its own physical instance. "Many sources in one database" is a capability of every instance — not a claim that all customers share one.

## Factory language → model language

The schema speaks database; its users speak factory. This table is the bridge:

| You say | The DB stores it as |
| --- | --- |
| PV — the actual, measured value | `measurement` row (SCALAR), or Parquet behind `file_ref` (TRACE / SCALAR\_BULK), at the param's `grain` |
| SV — the setpoint, the intent | `param_spec` row, `spec_kind='SETPOINT'`, context = the RECIPE\_VERSION (single home — D-031) |
| Test limit | `param_spec` row, `spec_kind='LIMIT'`, context = the TEST\_PROG\_VER |
| Profile / ramp setpoint (a curve, not one number) | RECIPE\_STEP child entities under the RECIPE\_VERSION, or a TRACE-lane file anchored to it (D-031 pattern) |
| Recipe — the program the tool runs | `RECIPE` → `RECIPE_VERSION` entities (PROCESS domain) |
| SVID / test number | `param_def.native_id` |
| Test condition / corner (25 °C vs 125 °C) | one `param_def` per condition + `param_condition` rows; same `param_concept` ties them (D-038) |
| Lot history / travel | `link` rows, kinds PROCESSED\_IN / TESTED\_ON (context = the recipe / program version that ran) |
| Made-from / BOM | `link` rows, kinds ASSEMBLY / MERGE / SPLIT |
| Chamber telemetry over time / pipeline monitoring | `observation` rows (TIMESERIES lane, D-035) |
| "Which file did this number come from" | `measurement.batch_id` → `ingest_batch.source_uri` (one join, D-034) |

## What "truly universal" means (the test)

The DB is universal when **onboarding a new domain or customer requires writing rows (a domain pack), not editing the schema or forking the database.** Concretely:

- A new measurement → a new `param_def` row. Never `ALTER TABLE`.
- A new item type or naming scheme → new `entity_type` / `entity_key_def` rows.
- A new high-rate signal → a `file_ref` (lane=TRACE) + `file_layout` rows pointing at Parquet.
- A new relationship kind (routing or lineage) → a `link_type` row carrying its own domain rules. Never a new bridge table.
- A new test condition → new `param_def` + `param_condition` rows (D-038). Never a key change.
- A new *source system* → a new `source` / `source_code` namespace in the same tables; no new database within a deployment. A new *customer* → a new physical instance of the same unforked schema (D-026) — still zero schema changes.
- The storage lanes (**SCALAR** rows / **SCALAR\_BULK** columnar Parquet / **TRACE** raw-trace Parquet / **TRACE+SCALAR** / **TIMESERIES** time-keyed rows) cover the data without bespoke storage.

If a new case ever forces a schema change, that is a **defect in universality** — log it (see [04 · Known Limits & Deferred Items](./04-known-limits-deferred.md)) before patching the schema.

## The constitution — invariants I1–I14

Binding on the committed baseline; every one is also a **portable requirement** (D-029): a proposal may challenge one only through a requirements walk (satisfied as-is / satisfied differently / consciously dropped with the failure mode named), ratified by a `D-###` — never a silent edit.

1. **I1 — Identity is never event metadata.** Never build keys from `start_time`, `equipment`, or any per-event field — those go in `attrs`. Distinguish two processing passes at the **RUN** level (`local_id = STEP + '#' + pass`), never by timestamp. Time-window columns on `link` (D-042) are context, never key.
2. **I2 —** `natural_key` is derived only. Only the resolver writes it; never INSERT/UPDATE it directly; no `/` inside a `local_id`.
3. **I3 —** `entity_key_def` is append-only. A recipe change = INSERT a new `version` + an explicit re-key migration. Never UPDATE/DELETE a recipe row.
4. **I4 — Joins use** `entity_id`. `natural_key` is only for resolution, display, and prefix filters (`LIKE 'LOT2291/%'`).
5. **I5 — Raw traces never become relational rows.** High-rate questions → read the TRACE-lane Parquet in DuckDB/pandas; never unpivot samples into `measurement` or `observation` — the TIMESERIES lane is bounded by its cadence rule (SCADA-rate and slower, lane-guarded; D-035).
6. **I6 — Roll-ups are computed at parent grain, never stored as parent-grain measurements.** A materialized roll-up *table* is fine as a clearly-separate derived artifact.
7. **I7 — Soft-delete only in registries.** Deactivate with `is_active = FALSE`; never DELETE rows that have measurements. Extended by D-043: a purged file's catalog row becomes a tombstone (`tier='PURGED'`), never deleted.
8. **I8 — Wide tables, the graph projection, and every derived dataset are disposable.** Never a source of truth. Every derived dataset carries a manifest pinning instance, schema version and batch watermark (D-046).
9. **I9 — One SVID, one definition.** A parameter's meaning lives ONLY in `param_def`. `file_layout` maps `param_id` → physical column and stores NO names/units. Aliases and concepts point *at* the one definition (D-036). A parameter measured under a distinct condition is a distinct `param_def` sharing a `param_concept` (D-038).
10. **I10 — Dimensions never cross via the tree.** `parent_id` stays inside one `domain` (PRODUCT / EQUIPMENT / PROCESS). Cross-dimension relationships exist **only** as `link` rows whose `link_type` rules allow the crossing, joined on `entity_id`. Crossing rules are data, enforced by one generic fail-loud guard (D-030).
11. **I11 — One scalar per (item, parameter).** `measurement` PK is `(entity_id, param_id)`. History on *product* items is expressed across entities (RUN, wafer, …); history on a **standing asset** lives in the TIMESERIES lane (`observation`, keyed by `ts`) — never repeated `measurement` rows.
12. **I12 — Ingestion is idempotent.** Resolve-or-create lands a re-sent record on the *same* row — no duplicates, ever. A re-send carrying a **different value** (including a different `link` window — D-042) fails loudly unless its batch is flagged CORRECTION, which archives the old value first. No silent overwrites.
13. **I13 — Canonicalization is closed and fail-loud.** The rule set (`AS_IS`, `UPPER`, `DIGITS_PAD2`, `DIGITS_PAD3`) is immutable — new behaviour = a new *named* rule; a missing recipe fails loudly, never guesses. Applied to time too: `attrs` timestamps are UTC ISO-8601 or the load fails (D-044).
14. **I14 — One** `type_code`, one item kind. All entities of a type are keyed from the same source field(s) and play the same role in the tree; otherwise they are different types. Never discriminate item kinds via `attrs` (D-028, from pack lesson L-SCK-01).

## The data model — 18 tables, six concepts

**ITEM** (`entity_type`, `entity_key_def`, `entity`) · **VALUE** (`param_def`, `param_condition`, `measurement`, `observation`, `param_concept`, `param_alias`) · **SPEC** (`param_spec`) · **FILE** (`file_layout_set`, `file_layout`, `file_ref`) · **LINK** (`link`, `link_type`) · **PROVENANCE** (`ingest_batch`, `measurement_audit`, `schema_migration`).

One way to relate two items, one way to point at a file, one `batch_id` on every ingested row, one queryable schema version per instance. New measurement / item / naming / relationship / condition types are **rows**, never schema changes.

| Table | One row is… | Key columns to know |
| --- | --- | --- |
| `entity_type` | a kind of item | `type_code`, `parent_type`, `domain` |
| `entity_key_def` | one code in one type's naming recipe (per source) | `type_code`, `source_code`, `version`, `seq`, `field_name`, `prefix`, `canon_rule` |
| `entity` | one tracked item | `entity_id` (joins), `parent_id` (tree), `local_id`, `natural_key`, `attrs` |
| `param_def` | a kind of measurement (the Dictionary) | `param_id`, `source`, `native_id`, `grain`, `value_type`, `storage_class`, `concept_id`, `to_canonical_scale`/`_offset` (D-040) |
| `param_condition` | one condition axis qualifying a parameter (D-038) | PK (`param_id`, `condition_key`); `value_num`/`value_text` |
| `measurement` | one scalar value | PK (`entity_id`, `param_id`); `value_num`/`value_text`/`value_bool`; `captured_at`; `batch_id` |
| `observation` | one time-keyed telemetry point on a standing asset (TIMESERIES, D-035) | PK (`entity_id`, `param_id`, `ts`); partitioned by `ts` |
| `param_concept` | one canonical cross-source concept | `concept_id`, `concept_code`, `canonical_unit` |
| `param_alias` | one synonym resolving to a parameter | PK (`param_id`, `alias`); `kind` |
| `param_spec` | one spec band (limit/setpoint) for a param in a context | PK (`param_id`, `context_entity_id`, `spec_kind`, `valid_from`); one ACTIVE band guaranteed |
| `file_layout_set` | one file-column layout, lane-tagged | `layout_id`, `lane`, `time_column` (D-041) |
| `file_layout` | one column (channel / test) of a layout | PK (`layout_id`, `param_id`); `column_name` |
| `file_ref` | a pointer to one Parquet file | `storage_uri` PK, `lane`, `anchor_entity_id`, `layout_id`, `row_grain`, `sampling_hz`, `window_start`/`window_end`, `tier` (D-043), `batch_id` |
| `link` | one relationship between two items — routing event or lineage edge | PK `link_id`; UNIQUE NULLS NOT DISTINCT (`src`, `dst`, `link_kind`, `context_entity_id`); `window_start`/`window_end` + `ts_window` (D-042); `batch_id` |
| `link_type` | one allowed relationship kind + its domain rules | `link_kind`; `src_domain`, `dst_domain`, `context_domain`, `acyclic` |
| `ingest_batch` | one ingestion / correction batch — provenance of every row it wrote | `batch_id`, `source_code`, `source_uri`, `kind` (INGEST / CORRECTION) |
| `measurement_audit` | one superseded value, archived by a CORRECTION batch (append-only) | (`entity_id`, `param_id`), old value, `superseded_by_batch` |
| `schema_migration` | one applied DDL package — provenance of the schema itself (D-045) | `version` PK, `applied_at`, `script_sha256`, `decision_ids` |

Load-bearing details: `value_type` ∈ {FLOAT, STRING, BOOL, CATEGORICAL}; `storage_class` ∈ {SCALAR, SCALAR\_BULK, TRACE, TRACE+SCALAR, TIMESERIES}; `entity` uniqueness is `UNIQUE NULLS NOT DISTINCT (type_code, parent_id, local_id)` so root items can't silently duplicate; exactly one value column per `measurement` row (CHECK).

## Canonical DDL — v2.2

Run the three blocks **in order** on a fresh instance. Together they are the canonical v2.2 DDL (verified end-to-end on PostgreSQL 16.13, 2026-08-11 → 18 tables, all guards firing).

### Block 1 — v2.0 base (11 tables, D-030)

```sql
CREATE TABLE entity_type (
    type_code   VARCHAR(16) PRIMARY KEY,            -- LOT|WAFER|SITE|RUN|TRAY|MODULE|TOOL|CHAMBER|RECIPE|RECIPE_VERSION|...
    parent_type VARCHAR(16) REFERENCES entity_type(type_code),  -- must stay within one domain (I10)
    domain      VARCHAR(16) NOT NULL DEFAULT 'PRODUCT',         -- PRODUCT | EQUIPMENT | PROCESS
    description TEXT
);

CREATE TABLE entity_key_def (                       -- APPEND-ONLY (I3)
    type_code   VARCHAR(16) NOT NULL REFERENCES entity_type(type_code),
    source_code VARCHAR(32) NOT NULL DEFAULT '*',   -- specific source beats '*'
    version     SMALLINT    NOT NULL DEFAULT 1,
    seq         SMALLINT    NOT NULL,
    field_name  VARCHAR(64) NOT NULL,
    prefix      VARCHAR(8)  NOT NULL DEFAULT '',
    canon_rule  VARCHAR(16) NOT NULL,
    PRIMARY KEY (type_code, source_code, version, seq)
);

CREATE TABLE entity (
    entity_id   BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    type_code   VARCHAR(16)  NOT NULL REFERENCES entity_type(type_code),
    parent_id   BIGINT REFERENCES entity(entity_id),
    local_id    VARCHAR(64)  NOT NULL CHECK (local_id !~ '/'),
    natural_key VARCHAR(256) NOT NULL,              -- derived: parent.natural_key || '/' || local_id
    attrs       JSONB        NOT NULL DEFAULT '{}', -- start_time, equipment... NOT identity
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT now(),
    CONSTRAINT uq_entity_local  UNIQUE NULLS NOT DISTINCT (type_code, parent_id, local_id),
    CONSTRAINT uq_entity_natkey UNIQUE (natural_key)
);

CREATE TABLE param_def (
    param_id      BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    source        VARCHAR(32)  NOT NULL,            -- 'SEMES' | 'LG' | ...
    native_id     VARCHAR(128) NOT NULL,            -- 'SVID12', 'PID3', 'LOT-YLD'
    name          VARCHAR(256) NOT NULL,
    unit          VARCHAR(32),
    grain         VARCHAR(16)  NOT NULL REFERENCES entity_type(type_code),
    value_type    VARCHAR(16)  NOT NULL,            -- FLOAT|STRING|BOOL|CATEGORICAL
    storage_class VARCHAR(16)  NOT NULL DEFAULT 'SCALAR',  -- SCALAR | SCALAR_BULK | TRACE | TRACE+SCALAR
    is_active     BOOLEAN      NOT NULL DEFAULT TRUE,
    UNIQUE (source, native_id)
);

CREATE TABLE measurement (
    entity_id   BIGINT NOT NULL REFERENCES entity(entity_id),
    param_id    BIGINT NOT NULL REFERENCES param_def(param_id),
    value_num   DOUBLE PRECISION,
    value_text  VARCHAR(256),
    value_bool  BOOLEAN,
    captured_at TIMESTAMPTZ,
    PRIMARY KEY (entity_id, param_id),
    -- I11: exactly ONE value column is populated per row.
    CONSTRAINT one_value_only CHECK (num_nonnulls(value_num, value_text, value_bool) = 1)
);

-- Unified file-layout catalog (D-025): ONE param_id -> Parquet-column map for BOTH file lanes
CREATE TABLE file_layout_set (
    layout_id   INT PRIMARY KEY,
    lane        VARCHAR(16) NOT NULL CHECK (lane IN ('TRACE','SCALAR_BULK')),
    description TEXT
);
CREATE TABLE file_layout (
    layout_id   INT    NOT NULL REFERENCES file_layout_set(layout_id),
    param_id    BIGINT NOT NULL REFERENCES param_def(param_id),
    column_name VARCHAR(64) NOT NULL,
    PRIMARY KEY (layout_id, param_id),
    UNIQUE (layout_id, column_name)   -- two params never share one physical column
);

-- Unified file pointer (D-030): ONE table for both Parquet lanes.
-- lane=TRACE: one file per traced entity (anchor = that entity)
-- lane=SCALAR_BULK: one file per parent, row per unit/die (anchor = the parent)
CREATE TABLE file_ref (
    storage_uri      TEXT        PRIMARY KEY,
    lane             VARCHAR(16) NOT NULL CHECK (lane IN ('TRACE','SCALAR_BULK')),
    anchor_entity_id BIGINT      NOT NULL REFERENCES entity(entity_id),
    layout_id        INT         NOT NULL REFERENCES file_layout_set(layout_id),
    row_grain        VARCHAR(16) REFERENCES entity_type(type_code),  -- SCALAR_BULK only: UNIT | DIE
    row_key_column   VARCHAR(64),                                    -- SCALAR_BULK only
    row_count        BIGINT,                                         -- SCALAR_BULK only
    sampling_hz      DOUBLE PRECISION,                                -- TRACE only
    sample_count     BIGINT,                                          -- TRACE only
    captured_at      TIMESTAMPTZ,
    CONSTRAINT file_ref_lane_shape CHECK (
        (lane = 'SCALAR_BULK' AND row_grain IS NOT NULL AND row_key_column IS NOT NULL
             AND sampling_hz IS NULL AND sample_count IS NULL)
     OR (lane = 'TRACE' AND row_grain IS NULL AND row_key_column IS NULL AND row_count IS NULL)
    )
);
CREATE INDEX ix_file_ref_anchor ON file_ref (anchor_entity_id);

-- Fail-loud lane guards (D-025/D-030)
CREATE FUNCTION file_ref_lane_guard() RETURNS trigger AS $$
DECLARE
    actual VARCHAR(16);
BEGIN
    SELECT lane INTO actual FROM file_layout_set WHERE layout_id = NEW.layout_id;
    IF actual IS DISTINCT FROM NEW.lane THEN
        RAISE EXCEPTION 'file_ref.lane=% must match its layout lane=% (layout_id=%)',
            NEW.lane, actual, NEW.layout_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_file_ref_lane BEFORE INSERT OR UPDATE ON file_ref
    FOR EACH ROW EXECUTE FUNCTION file_ref_lane_guard();

CREATE FUNCTION file_layout_set_lane_freeze() RETURNS trigger AS $$
BEGIN
    IF NEW.lane <> OLD.lane THEN
        RAISE EXCEPTION 'file_layout_set.lane is immutable (layout_id=%)', OLD.layout_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_file_layout_set_lane_freeze
    BEFORE UPDATE ON file_layout_set
    FOR EACH ROW EXECUTE FUNCTION file_layout_set_lane_freeze();

-- Spec/limit dimension (D-021)
CREATE TABLE param_spec (
    param_id          BIGINT      NOT NULL REFERENCES param_def(param_id),
    context_entity_id BIGINT      NOT NULL REFERENCES entity(entity_id),  -- governing node: TEST_PROG_VER / RECIPE_VERSION
    spec_kind         VARCHAR(16) NOT NULL DEFAULT 'LIMIT',               -- LIMIT | SETPOINT
    lo_num            DOUBLE PRECISION,
    hi_num            DOUBLE PRECISION,
    target_num        DOUBLE PRECISION,
    src_unit          VARCHAR(32),
    src_scale         SMALLINT,
    is_active         BOOLEAN     NOT NULL DEFAULT TRUE,
    captured_at       TIMESTAMPTZ,
    PRIMARY KEY (param_id, context_entity_id, spec_kind),
    CHECK (lo_num IS NULL OR hi_num IS NULL OR lo_num <= hi_num)
);

-- Fail-loud guard: param_spec context must be a PROCESS-domain entity (D-023)
CREATE FUNCTION param_spec_guard() RETURNS trigger AS $$
BEGIN
    IF (SELECT et.domain
          FROM entity e
          JOIN entity_type et ON et.type_code = e.type_code
         WHERE e.entity_id = NEW.context_entity_id) IS DISTINCT FROM 'PROCESS' THEN
        RAISE EXCEPTION
            'param_spec.context_entity_id must be a PROCESS-domain entity (got entity_id=%)',
            NEW.context_entity_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_param_spec_guard
    BEFORE INSERT OR UPDATE ON param_spec
    FOR EACH ROW EXECUTE FUNCTION param_spec_guard();

-- Unified relationships (D-030)
CREATE TABLE link_type (
    link_kind      VARCHAR(16) PRIMARY KEY,   -- ASSEMBLY | MERGE | SPLIT | PROCESSED_IN | TESTED_ON | ...
    src_domain     VARCHAR(16) NOT NULL,
    dst_domain     VARCHAR(16) NOT NULL,
    context_domain VARCHAR(16),               -- NULL = no context allowed
    acyclic        BOOLEAN NOT NULL DEFAULT FALSE,
    description    TEXT
);
INSERT INTO link_type (link_kind, src_domain, dst_domain, context_domain, acyclic, description) VALUES
  ('ASSEMBLY',     'PRODUCT', 'PRODUCT',   NULL,      TRUE,  'many components -> one assembled item'),
  ('MERGE',        'PRODUCT', 'PRODUCT',   NULL,      TRUE,  'many source lots -> one merged lot'),
  ('SPLIT',        'PRODUCT', 'PRODUCT',   NULL,      TRUE,  'one source -> many outputs, ONLY when the output is not named under the source'),
  ('PROCESSED_IN', 'PRODUCT', 'EQUIPMENT', 'PROCESS', FALSE, 'product processed in equipment; context = the RECIPE_VERSION that ran'),
  ('TESTED_ON',    'PRODUCT', 'EQUIPMENT', 'PROCESS', FALSE, 'product tested on equipment; context = the TEST_PROG_VER that ran');

CREATE TABLE link (
    src_entity_id     BIGINT NOT NULL REFERENCES entity(entity_id),
    dst_entity_id     BIGINT NOT NULL REFERENCES entity(entity_id),
    link_kind         VARCHAR(16) NOT NULL REFERENCES link_type(link_kind),
    context_entity_id BIGINT REFERENCES entity(entity_id),
    attrs             JSONB NOT NULL DEFAULT '{}',    -- qty, slot, start_time... NOT identity (I1)
    captured_at       TIMESTAMPTZ,
    is_active         BOOLEAN NOT NULL DEFAULT TRUE,
    PRIMARY KEY (src_entity_id, dst_entity_id, link_kind),
    CHECK (src_entity_id <> dst_entity_id)
);
CREATE INDEX ix_link_dst  ON link (dst_entity_id);
CREATE INDEX ix_link_kind ON link (link_kind);

-- ONE generic fail-loud guard: validates src/dst/context domains and per-kind acyclicity
CREATE FUNCTION link_guard() RETURNS trigger AS $$
DECLARE
    rule  link_type%ROWTYPE;
    d_src VARCHAR(16);
    d_dst VARCHAR(16);
    d_ctx VARCHAR(16);
BEGIN
    SELECT * INTO rule FROM link_type WHERE link_kind = NEW.link_kind;
    SELECT et.domain INTO d_src FROM entity e JOIN entity_type et ON et.type_code = e.type_code
        WHERE e.entity_id = NEW.src_entity_id;
    SELECT et.domain INTO d_dst FROM entity e JOIN entity_type et ON et.type_code = e.type_code
        WHERE e.entity_id = NEW.dst_entity_id;
    IF d_src IS DISTINCT FROM rule.src_domain OR d_dst IS DISTINCT FROM rule.dst_domain THEN
        RAISE EXCEPTION 'link %: src must be % and dst must be % (got src=% dst=%)',
            NEW.link_kind, rule.src_domain, rule.dst_domain, d_src, d_dst;
    END IF;
    IF NEW.context_entity_id IS NOT NULL THEN
        IF rule.context_domain IS NULL THEN
            RAISE EXCEPTION 'link %: context_entity_id not allowed for this kind', NEW.link_kind;
        END IF;
        SELECT et.domain INTO d_ctx FROM entity e JOIN entity_type et ON et.type_code = e.type_code
            WHERE e.entity_id = NEW.context_entity_id;
        IF d_ctx IS DISTINCT FROM rule.context_domain THEN
            RAISE EXCEPTION 'link %: context must be %-domain (got %)',
                NEW.link_kind, rule.context_domain, d_ctx;
        END IF;
    END IF;
    IF rule.acyclic AND EXISTS (
        WITH RECURSIVE down AS (
            SELECT dst_entity_id AS node FROM link
              WHERE src_entity_id = NEW.dst_entity_id AND link_kind = NEW.link_kind AND is_active
            UNION
            SELECT l.dst_entity_id FROM link l
              JOIN down d ON l.src_entity_id = d.node
             WHERE l.link_kind = NEW.link_kind AND l.is_active
        )
        SELECT 1 FROM down WHERE node = NEW.src_entity_id
    ) THEN
        RAISE EXCEPTION 'link % would create a cycle (src=% dst=%)',
            NEW.link_kind, NEW.src_entity_id, NEW.dst_entity_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_link_guard
    BEFORE INSERT OR UPDATE ON link
    FOR EACH ROW EXECUTE FUNCTION link_guard();

-- Generic dimension seeds (rows, not schema -- adjust per deployment)
INSERT INTO entity_type (type_code, parent_type, domain) VALUES
  ('TOOL',           NULL,     'EQUIPMENT'),
  ('CHAMBER',        'TOOL',   'EQUIPMENT'),
  ('RECIPE',         NULL,     'PROCESS'),
  ('RECIPE_VERSION', 'RECIPE', 'PROCESS');

INSERT INTO entity_key_def (type_code, source_code, seq, field_name, prefix, canon_rule) VALUES
  ('TOOL',           '*', 1, 'tool_id',     '',  'UPPER'),
  ('CHAMBER',        '*', 1, 'chamber_no',  'C', 'DIGITS_PAD2'),
  ('RECIPE',         '*', 1, 'recipe_name', '',  'UPPER'),
  ('RECIPE_VERSION', '*', 1, 'version',     'V', 'DIGITS_PAD2');
```

### Block 2 — v2.0 → v2.1 delta (D-034 · D-035 · D-036)

```sql
-- ===== D-034 (P-002): integrity hardening =====
ALTER TABLE param_spec ADD COLUMN valid_from TIMESTAMPTZ NOT NULL DEFAULT now();
ALTER TABLE param_spec DROP CONSTRAINT param_spec_pkey;
ALTER TABLE param_spec ADD PRIMARY KEY (param_id, context_entity_id, spec_kind, valid_from);
CREATE UNIQUE INDEX uq_param_spec_active
    ON param_spec (param_id, context_entity_id, spec_kind) WHERE is_active;
-- a spec revision = one transaction: UPDATE old SET is_active = FALSE; INSERT new.

ALTER TABLE link ADD COLUMN link_id BIGINT GENERATED ALWAYS AS IDENTITY;
ALTER TABLE link DROP CONSTRAINT link_pkey;
ALTER TABLE link ADD PRIMARY KEY (link_id);
ALTER TABLE link ADD CONSTRAINT uq_link_natural
    UNIQUE NULLS NOT DISTINCT (src_entity_id, dst_entity_id, link_kind, context_entity_id);
-- loader ON CONFLICT target is now uq_link_natural (I12)

CREATE TABLE ingest_batch (
    batch_id    BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    source_code VARCHAR(32) NOT NULL,
    source_uri  TEXT,                                   -- file / feed of origin
    kind        VARCHAR(16) NOT NULL DEFAULT 'INGEST',  -- INGEST | CORRECTION
    started_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    finished_at TIMESTAMPTZ,
    note        TEXT
);
ALTER TABLE measurement ADD COLUMN batch_id BIGINT REFERENCES ingest_batch(batch_id);
ALTER TABLE link        ADD COLUMN batch_id BIGINT REFERENCES ingest_batch(batch_id);
ALTER TABLE file_ref    ADD COLUMN batch_id BIGINT REFERENCES ingest_batch(batch_id);
ALTER TABLE param_spec  ADD COLUMN batch_id BIGINT REFERENCES ingest_batch(batch_id);

-- POLICY: a re-send whose value differs FAILS LOUDLY unless batch.kind='CORRECTION';
-- a correction first archives the old row to measurement_audit, then overwrites.
CREATE TABLE measurement_audit (        -- append-only
    entity_id           BIGINT NOT NULL,
    param_id            BIGINT NOT NULL,
    value_num           DOUBLE PRECISION,
    value_text          TEXT,
    value_bool          BOOLEAN,
    captured_at         TIMESTAMPTZ,
    batch_id            BIGINT,
    superseded_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    superseded_by_batch BIGINT NOT NULL REFERENCES ingest_batch(batch_id)
);

ALTER TABLE file_ref ADD COLUMN window_start TIMESTAMPTZ;
ALTER TABLE file_ref ADD COLUMN window_end   TIMESTAMPTZ;
ALTER TABLE file_ref ADD CONSTRAINT file_ref_window
    CHECK (window_start IS NULL OR window_end IS NULL OR window_start <= window_end);

ALTER TABLE measurement ALTER COLUMN value_text  TYPE TEXT;   -- metadata-only ALTER
ALTER TABLE entity      ALTER COLUMN natural_key TYPE TEXT;   -- metadata-only ALTER
-- type_code stays VARCHAR(16) (D-027).
-- Guard ops: link_guard() takes pg_advisory_xact_lock(hashtext(NEW.link_kind)) before
-- the cycle walk on acyclic kinds; bulk loads disable trg_link_guard in-tx + one
-- set-based validation before COMMIT; nightly verification query as backstop.

-- ===== D-035 (P-003): TIMESERIES lane =====
-- param_def.storage_class gains the VALUE 'TIMESERIES' (a row value, no ALTER).
CREATE TABLE observation (
    entity_id BIGINT NOT NULL REFERENCES entity(entity_id),   -- the standing asset
    param_id  BIGINT NOT NULL REFERENCES param_def(param_id),
    ts        TIMESTAMPTZ NOT NULL,
    value_num DOUBLE PRECISION NOT NULL,                      -- v1: numeric only
    batch_id  BIGINT REFERENCES ingest_batch(batch_id),
    PRIMARY KEY (entity_id, param_id, ts)
) PARTITION BY RANGE (ts);                                    -- monthly by default
-- cadence boundary: SCADA-rate and slower; faster stays TRACE (I5).

-- ===== D-036 (P-004): vocabulary layer =====
CREATE TABLE param_concept (
    concept_id     BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    concept_code   VARCHAR(64) UNIQUE NOT NULL,   -- 'CHAMBER_PRESSURE'
    name           TEXT NOT NULL,
    canonical_unit VARCHAR(32),
    description    TEXT
);
ALTER TABLE param_def ADD COLUMN concept_id BIGINT REFERENCES param_concept(concept_id);
CREATE TABLE param_alias (
    param_id BIGINT NOT NULL REFERENCES param_def(param_id),
    alias    TEXT   NOT NULL,                          -- '압력', 'PRESS', 'SVID12'
    kind     VARCHAR(16) NOT NULL DEFAULT 'SYNONYM',   -- SYNONYM | LOCALE | LEGACY
    PRIMARY KEY (param_id, alias)
);
-- optional param_def.role (ALARM|COUNTER|YIELD|PV); deliberately NO 'SV' role (I9, D-031).
-- D-033 (P-005): the graph projection is DERIVED under I8 — no baseline DDL.
```

### Block 3 — v2.1 → v2.2 delta (close-out package, D-038/D-040/D-041/D-042/D-043/D-045)

```sql
-- ===== D-038: condition model (resolves C-04) =====
CREATE TABLE param_condition (
    param_id      BIGINT NOT NULL REFERENCES param_def(param_id),
    condition_key VARCHAR(32) NOT NULL,          -- 'TEMP_C', 'VDD_V', 'FREQ_HZ', ...
    value_num     DOUBLE PRECISION,
    value_text    TEXT,
    PRIMARY KEY (param_id, condition_key),
    CONSTRAINT one_condition_value CHECK (num_nonnulls(value_num, value_text) = 1)
);

-- ===== D-040: unit-conversion mechanics =====
-- canonical_value = value_num * to_canonical_scale + to_canonical_offset.
ALTER TABLE param_def ADD COLUMN to_canonical_scale  DOUBLE PRECISION;
ALTER TABLE param_def ADD COLUMN to_canonical_offset DOUBLE PRECISION;
ALTER TABLE param_def ADD CONSTRAINT param_def_conv_pair
    CHECK (num_nonnulls(to_canonical_scale, to_canonical_offset) IN (0, 2));

-- ===== D-041: timestamped / irregular traces =====
-- NULL = uniformly sampled (file_ref.sampling_hz describes timing);
-- set  = the physical Parquet column carrying per-row timestamps.
ALTER TABLE file_layout_set ADD COLUMN time_column VARCHAR(64);

-- ===== D-042: time-window bridge on link =====
-- Windows are context, NEVER identity: uq_link_natural is unchanged (I1, D-034).
ALTER TABLE link ADD COLUMN window_start TIMESTAMPTZ;
ALTER TABLE link ADD COLUMN window_end   TIMESTAMPTZ;
ALTER TABLE link ADD CONSTRAINT link_window_ck
    CHECK (window_start IS NULL OR window_end IS NULL OR window_start <= window_end);
ALTER TABLE link ADD COLUMN ts_window tstzrange
    GENERATED ALWAYS AS (tstzrange(window_start, window_end, '[)')) STORED;
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE INDEX ix_link_occupancy ON link
    USING gist (dst_entity_id, link_kind, ts_window) WHERE is_active;
-- Guard: set-based overlap validation before COMMIT on bulk loads (running-max
-- form, NOT lag()). NO EXCLUDE constraint (measured 32x bulk-insert slowdown).

-- ===== D-043: file retention & tiering (resolves Q-002) =====
ALTER TABLE file_ref ADD COLUMN tier VARCHAR(16) NOT NULL DEFAULT 'HOT'
    CHECK (tier IN ('HOT', 'COLD', 'PURGED'));
ALTER TABLE file_ref ADD COLUMN tier_changed_at TIMESTAMPTZ;
-- PURGED keeps the row as a tombstone; readers fail loudly on non-HOT tiers.

-- ===== D-045: schema-version registry (resolves Q-006) =====
CREATE TABLE schema_migration (
    version       VARCHAR(16) PRIMARY KEY,       -- 'v2.2'
    applied_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    script_sha256 CHAR(64),
    decision_ids  TEXT,
    note          TEXT
);
INSERT INTO schema_migration (version, decision_ids, note)
VALUES ('v2.2', 'D-038, D-040, D-041, D-042, D-043, D-045',
        'close-out package: condition model, unit conversion, trace timing, link windows, tiering, migration registry');
```

## The three dimensions and genealogy — the rules

**Equipment (D-015/D-016/D-022/D-030).** Chambers and tools are ordinary entities in a second `domain` tree (`TOOL` → `CHAMBER`), never wired into the product tree by `parent_id`. The bridge is a `link` of kind PROCESSED\_IN / TESTED\_ON from the product entity (WAFER, or RUN when reprocessing exists) to the equipment entity — the only legal product↔equipment crossing. Chamber is never identity (I1). Current-value equipment metrics (last PM date, fault state) are ordinary `measurement` rows at CHAMBER grain — **snapshots only** (D-022); time-history on a standing asset lives in `observation` (D-035).

**Process recipe (D-017/D-018/D-030/D-031).** A recipe is a *process spec*, third dimension: `RECIPE` → `RECIPE_VERSION`, each version its own entity. It rides on the routing event as `link.context_entity_id` — the event answers *which product, in which chamber, under which recipe* in one row. Scalar setpoints live in `param_spec` (`spec_kind='SETPOINT'`, context = RECIPE\_VERSION) — the single home of SVs (D-031); setpoints-as-measurements are deprecated. Curve/ramp setpoints: RECIPE\_STEP child entities or a TRACE-lane file anchored to the version. Note: the `entity_key_def` *naming recipe* is unrelated config — two things share the word "recipe."

**Genealogy (D-019/D-030).** Assembly/merge/split lineage is a graph *beside* the tree: `link` rows of PRODUCT-only kinds ASSEMBLY / MERGE / SPLIT (acyclic, guard-enforced). `parent_id` stays the single naming/containment parent; an edge never feeds identity or `natural_key`. Ordinary dicing (outputs named under the source) stays in the tree; SPLIT links are only for outputs *not* named under their source. Wrong edges are soft-retired, never deleted.

**Relations verdict (D-033).** `link` is the sole source of truth for all relations. Multi-hop traversal is served by a **derived graph projection** — rebuilt from `link WHERE is_active` on a watermark, disposable, never written directly (I8). A graph store as a *replacement* system of record is explicitly not ratified.

## Schema diagram (ERD)

Mermaid source — renders in any Mermaid viewer (crow's foot: `||` one, `o{` many). `entity` is the hub: it self-loops for the tree and plays three roles in `link` (source / target / context).

```
erDiagram
  %% ---- ITEM ----
  ENTITY_TYPE ||--o{ ENTITY_TYPE : "parent of"
  ENTITY_TYPE ||--o{ ENTITY_KEY_DEF : "named by"
  ENTITY_TYPE ||--o{ PARAM_DEF : "grain of"
  ENTITY_TYPE ||--o{ ENTITY : "typed as"
  ENTITY ||--o{ ENTITY : "parent of"
  %% ---- VALUE ----
  ENTITY ||--o{ MEASUREMENT : "has value"
  PARAM_DEF ||--o{ MEASUREMENT : "defines"
  PARAM_DEF ||--o{ PARAM_CONDITION : "qualified by"
  ENTITY ||--o{ OBSERVATION : "telemetry on"
  PARAM_DEF ||--o{ OBSERVATION : "defines"
  PARAM_CONCEPT ||--o{ PARAM_DEF : "canonicalizes"
  PARAM_DEF ||--o{ PARAM_ALIAS : "known as"
  %% ---- FILE ----
  PARAM_DEF ||--o{ FILE_LAYOUT : "mapped as"
  FILE_LAYOUT_SET ||--o{ FILE_LAYOUT : "contains"
  FILE_LAYOUT_SET ||--o{ FILE_REF : "describes"
  ENTITY ||--o{ FILE_REF : "anchor of"
  %% ---- SPEC ----
  PARAM_DEF ||--o{ PARAM_SPEC : "bounded by"
  ENTITY ||--o{ PARAM_SPEC : "context for"
  %% ---- LINK ----
  ENTITY ||--o{ LINK : "source of"
  ENTITY ||--o{ LINK : "target of"
  ENTITY ||--o{ LINK : "context of"
  LINK_TYPE ||--o{ LINK : "kind of"
  %% ---- PROVENANCE ----
  INGEST_BATCH ||--o{ MEASUREMENT : "wrote"
  INGEST_BATCH ||--o{ LINK : "wrote"
  INGEST_BATCH ||--o{ FILE_REF : "wrote"
  INGEST_BATCH ||--o{ PARAM_SPEC : "wrote"
  INGEST_BATCH ||--o{ OBSERVATION : "wrote"
  INGEST_BATCH ||--o{ MEASUREMENT_AUDIT : "superseded via"
  ENTITY_TYPE { string type_code PK }
  ENTITY_KEY_DEF { string type_code FK }
  PARAM_DEF { bigint param_id PK }
  PARAM_CONDITION { bigint param_id PK }
  ENTITY { bigint entity_id PK }
  MEASUREMENT { bigint entity_id PK }
  OBSERVATION { timestamptz ts PK }
  PARAM_CONCEPT { bigint concept_id PK }
  PARAM_ALIAS { string alias PK }
  FILE_LAYOUT_SET { int layout_id PK }
  FILE_LAYOUT { string column_name UK }
  FILE_REF { string storage_uri PK }
  PARAM_SPEC { timestamptz valid_from PK }
  LINK { bigint link_id PK }
  LINK_TYPE { string link_kind PK }
  INGEST_BATCH { bigint batch_id PK }
  MEASUREMENT_AUDIT { bigint entity_id }
  SCHEMA_MIGRATION { string version PK }
```

## Column dictionary — v2.2 additions and changes

The full v2.1 column dictionary is preserved verbatim in the frozen Notion Guideline (linked from the [Decision Log](./05-decision-log.md) page); everything there remains correct. v2.2 adds:

| Table | Column | Detail | Links |
| --- | --- | --- | --- |
| `param_def` | `to_canonical_scale` / `to_canonical_offset` | Linear transform to the concept's `canonical_unit` (canonical = value × scale + offset); pair-or-neither CHECK; mandatory at mapping review when unit ≠ canonical\_unit | — (D-040) |
| `param_condition` | `param_id` | The parameter this condition qualifies | PK; FK → `param_def.param_id` (D-038) |
| `param_condition` | `condition_key` | Condition axis: 'TEMP\_C', 'VDD\_V', 'FREQ\_HZ', … | PK |
| `param_condition` | `value_num` / `value_text` | The condition's value (exactly one populated, CHECK) | — |
| `file_layout_set` | `time_column` | In-file timestamp column for irregular/timestamped traces; NULL = uniformly sampled (`file_ref.sampling_hz`) | — (D-041) |
| `link` | `window_start` / `window_end` | Occupancy window of the event (context, never identity); NULL legal; CHECK ordered | — (D-042) |
| `link` | `ts_window` | STORED generated `tstzrange(window_start, window_end, '[)')`; indexed by partial composite GiST (dst, kind, ts\_window) | — (D-042) |
| `file_ref` | `tier` | HOT / COLD / PURGED (CHECK); PURGED = tombstone, row never deleted; readers fail loudly on non-HOT | — (D-043) |
| `file_ref` | `tier_changed_at` | When the tier last changed | — (D-043) |
| `schema_migration` | `version` | Applied schema version ('v2.2'); instance version = latest row by `applied_at` | PK (D-045) |
| `schema_migration` | `applied_at`, `script_sha256`, `decision_ids`, `note` | When, what exactly ran, and which decisions it implements | — (D-045) |

## Status snapshot (2026-08-11; `sck` and gap rows updated 2026-09-19)

| Piece | Status |
| --- | --- |
| Schema v2.2 ratified (D-038–D-048 close-out) | Done — DDL verified on PostgreSQL 16.13 |
| Milestone 1 (coverage backlog locked) | **Closed** — every item resolved or formally deferred with trigger |
| Milestone 2 (architecture verdict) | Verdict ratified (D-033); benchmarks run in field-test mode |
| `sck` instance | On **v2.2** — migrated from v2.0 on 2026-09-19 (Block 2 then Block 3; **Block 1 was not re-run**), path and scope per [D-050](./05-D050-full-record.md), which corrected the earlier "still on v1.1, one hop" reading. [07 · Field-Test Plan](./07-field-test-plan.md) Test 1: **7 of 8 gates green**; G6 alone is held open (see the row below). `script_sha256` `6af0eccb…21dae2`. The database is `sck`; `sck_raw` is retained as the v2.0 rollback. Always check `schema_migration` before writing SQL against an instance. |
| Two gaps between this page and the shipped DDL | Found by measurement during Test 1 and logged on [04 · Known Limits](./04-known-limits-deferred.md). (1) The `link_guard()` **advisory lock** that 08 · Operations asserts and Block 2 mentions in a comment exists in *no* DDL block — measured, the ratified guard lets two concurrent sessions commit a cycle on an acyclic kind. Fix drafted, pending **D-051**; blocks G6. (2) The **I12 correction policy** in Block 2 is likewise a comment with no trigger or constraint behind it — currently honoured only by loader code. Neither is a change to this page's ratified model; both are places where prose here is not yet executable. |
| SCK / SEMES packs | Outdated — re-verification in field tests |
| Innotek / KOGAS | Planned — field-test cases. An empty `innotek` instance stands on v2.2. |

*This page changes only by ratified decision. Found a problem? Log it per* [*04 · Known Limits & Deferred Items*](./04-known-limits-deferred.md) *— never patch silently.*
