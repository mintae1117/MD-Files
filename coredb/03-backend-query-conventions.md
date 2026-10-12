# 03 · Backend Query Conventions — Reading the Universal DB
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/377028632/03+Backend+Query+Conventions+Reading+the+Universal+DB
> pageId: 377028632 · lastModified: 8월 13, 2026 · saved: 2026-10-12

> **Audience: backend engineers writing SQL, services, or view generators against a Universal DB instance.** Schema reference: \[01 · Schema Spec\]. The one-sentence version: **check the instance version and the parameter's lane before writing a line of SQL, join on **`entity_id`, and treat everything derived as disposable.

## Before any SQL — three checks

1. **Which schema version is THIS instance on?** The spec describes the target, not necessarily the disk (the loaded `sck` instance is still on v1.1 until its migration).

```sql
SELECT version FROM schema_migration ORDER BY applied_at DESC LIMIT 1;
-- table absent => instance is pre-v2.2; check the migration plan in 08 · Operations
```

1. **What lane is the parameter in?** A file-lane parameter has **zero** `measurement` rows by design — an empty scalar query against a TRACE/SCALAR\_BULK param is your bug, not missing data.

```sql
SELECT param_id, grain, value_type, storage_class
FROM param_def WHERE source = :src AND native_id = :nid;
```

Where the values live per lane: SCALAR → `measurement` · SCALAR\_BULK / TRACE → Parquet behind `file_ref` (query with DuckDB/pandas; find files via `anchor_entity_id` or `window_start`/`window_end`; check `tier` first) · TIMESERIES → `observation` · TRACE+SCALAR → summaries in `measurement`, raw in the file.

1. **What grain?** `param_def.grain` names the entity level the value attaches to. Querying wafer-grain params at lot level without a tree walk returns nothing.

## Identity and join rules

- **Join on **`entity_id`. Always. `natural_key` is for resolution, display, and prefix filters only (`WHERE natural_key LIKE 'LOT2291/%'`). Never join tables on name strings (I4).
- Resolve a human path to an id once: `SELECT entity_id FROM entity WHERE natural_key = 'LOT2291/W07'`.
- "Wafer 7" with no lot is ambiguous — services must require the full path or ask, never guess.
- The tree: `entity.parent_id` (one domain only). Cross-domain hops (product↔equipment↔recipe) go through `link` — never through the tree (I10).
- Soft-retired rows: filter `is_active` on registries and `link`; retired ≠ deleted.

## Query cookbook (all shapes verified on the v2.2 schema)

**Which chamber processed this wafer, under which recipe:**

```sql
SELECT eq.natural_key AS chamber, rv.natural_key AS recipe_version, l.captured_at
FROM link l
JOIN entity eq ON eq.entity_id = l.dst_entity_id
LEFT JOIN entity rv ON rv.entity_id = l.context_entity_id
WHERE l.src_entity_id = :wafer_id AND l.link_kind = 'PROCESSED_IN' AND l.is_active;
```

**Corner curve — one quantity across test conditions (D-038):**

```sql
SELECT pc.value_num AS temp_c, m.value_num AS reading
FROM measurement m
JOIN param_def pd USING (param_id)
JOIN param_condition pc ON pc.param_id = pd.param_id AND pc.condition_key = 'TEMP_C'
WHERE pd.concept_id = :concept AND m.entity_id = :unit
ORDER BY 1;
```

**Cross-source, unit-consistent values through the concept layer (D-036/D-040):**

```sql
SELECT e.natural_key, pd.source,
       m.value_num * COALESCE(pd.to_canonical_scale, 1)
         + COALESCE(pd.to_canonical_offset, 0) AS value_canonical
FROM measurement m
JOIN param_def pd USING (param_id)
JOIN entity e ON e.entity_id = m.entity_id
WHERE pd.concept_id = (SELECT concept_id FROM param_concept WHERE concept_code = :code);
```

**Current spec band + pass/fail for a value (versioned specs, D-034):**

```sql
SELECT s.lo_num, s.hi_num,
       (m.value_num BETWEEN COALESCE(s.lo_num,'-infinity') AND COALESCE(s.hi_num,'infinity')) AS in_spec
FROM measurement m
JOIN link t ON t.src_entity_id = m.entity_id AND t.link_kind = 'TESTED_ON' AND t.is_active
JOIN param_spec s ON s.param_id = m.param_id
                 AND s.context_entity_id = t.context_entity_id
                 AND s.spec_kind = 'LIMIT' AND s.is_active
WHERE m.entity_id = :unit_id AND m.param_id = :param_id;
-- historical band instead: drop is_active, filter valid_from <= :as_of, take latest
```

**Standing-asset telemetry over a window (TIMESERIES, D-035):**

```sql
SELECT ts, value_num FROM observation
WHERE entity_id = :chamber AND param_id = :param
  AND ts >= :from AND ts < :to
ORDER BY ts;
```

**Attribute telemetry to whichever product occupied the equipment (D-042) — guard band mandatory:**

```sql
SELECT p.natural_key AS product, o.ts, o.value_num,
       CASE WHEN o.ts <  lower(l.ts_window) + :guard_band::interval
              OR o.ts >= upper(l.ts_window) - :guard_band::interval
            THEN 'AMBIGUOUS' ELSE 'ASSIGNED' END AS attribution
FROM observation o
JOIN link l ON l.dst_entity_id = o.entity_id
           AND l.link_kind = 'PROCESSED_IN' AND l.is_active
           AND l.ts_window @> o.ts
JOIN entity p ON p.entity_id = l.src_entity_id
WHERE o.entity_id = :chamber AND o.param_id = :param
  AND o.ts >= :from AND o.ts < :to;
-- guard band per source (see the pack spec). In-band samples are AMBIGUOUS, not assigned.
-- Open windows (window_end IS NULL) are excluded by @> semantics only at the start side —
-- refuse open windows unless the pack declares a next-start rule.
```

**Which file did this number come from (provenance, D-034):**

```sql
SELECT b.source_uri, b.kind, b.started_at
FROM measurement m JOIN ingest_batch b ON b.batch_id = m.batch_id
WHERE m.entity_id = :eid AND m.param_id = :pid;
```

**Roll-ups — compute at read (I6):**

```sql
SELECT parent.entity_id, avg(m.value_num) AS mean_child_value
FROM entity child
JOIN entity parent ON parent.entity_id = child.parent_id
JOIN measurement m ON m.entity_id = child.entity_id AND m.param_id = :param
WHERE parent.entity_id = :lot_id
GROUP BY 1;
-- NEVER write this result back as a measurement row on the parent
```

**Multi-hop lineage (ASSEMBLY chains) — recursive CTE; the graph projection serves the same query when built:**

```sql
WITH RECURSIVE ancestry AS (
  SELECT src_entity_id, dst_entity_id, 1 AS depth
  FROM link WHERE dst_entity_id = :module_id AND link_kind = 'ASSEMBLY' AND is_active
  UNION ALL
  SELECT l.src_entity_id, l.dst_entity_id, a.depth + 1
  FROM link l JOIN ancestry a ON l.dst_entity_id = a.src_entity_id
  WHERE l.link_kind = 'ASSEMBLY' AND l.is_active
)
SELECT e.natural_key, a.depth FROM ancestry a JOIN entity e ON e.entity_id = a.src_entity_id;
```

## The derived layer — services, views, datasets

- **Everything derived is disposable (I8):** wide tables, generated service views, the graph projection, training matrices. Regenerate freely; never write back to them as truth; never let a service treat one as the system of record.
- Generated service views filter on `type_code` — which is why I14 (one type\_code, one kind) is enforced at mapping time. If a view "mixes two kinds of thing," the pack has an I14 defect; report it, don't patch the view.
- Wide/materialized artifacts rebuild on an `entity_id` / `batch_id` watermark.
- **Every extracted dataset ships a manifest (D-046):** `instance`, `schema_version`, `watermark_batch_id` (= max `ingest_batch.batch_id` included), `built_at`, `generator`. Reproduce = same instance + version + `WHERE batch_id <= watermark`. A model citing a dataset cites its manifest. The DB stores nothing about derived datasets.
- Human-friendly naming belongs in the derived layer: use `param_alias` / `param_concept` to generate readable view columns — the base tables keep native codes (D-032: renames rejected; the vocabulary layer is the ergonomics answer).

## Service-layer must-nots

- No writes outside the loader path. Services read; loaders write through the batch/conflict discipline (\[02 · Runbook\]).
- Don't cache `natural_key` → `entity_id` mappings across re-key migrations (I3 events); subscribe to pack-version changes.
- Don't expose `link_id` or any surrogate as a public identifier — surrogates are internal; public identity is the `natural_key` path.
- Don't read a `file_ref` without checking `tier` — non-HOT reads must surface the tier error to the caller, not retry silently (D-043).
