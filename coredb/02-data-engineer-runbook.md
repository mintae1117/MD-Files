# 02 · Data Engineer Runbook — New Data Arrives, Do This
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/376930324/02+Data+Engineer+Runbook+New+Data+Arrives+Do+This
> pageId: 376930324 · lastModified: 8월 13, 2026 · saved: 2026-10-12

> **Audience: data engineers onboarding a new source or maintaining a loader.** The schema reference is [01 · Schema Spec](./01-schema-spec-v2.2.md). The one-sentence version of this page: **agree the mapping on paper first, write rows in the right order, make every load idempotent and provenance-stamped, and fail loudly instead of guessing.**

## The golden rule — design before code

Never start with the parser. Every source is onboarded design-first:

1. **Survey the source** (step 0 below) and draft the mapping: entity tree, parameters, lanes, specs, relations.
2. **Review the mapping** against the checklists here — this is where I14 violations, lane mistakes, and identity-from-timestamp bugs get caught cheaply.
3. Get **sign-off on the mapping** (a written pack spec — see [06 · Case Registry](./06-case-registry.md) for the format).
4. Only then write the loader.

A mapping mistake found after loading costs a re-load and possibly a correction cycle. Found at review, it costs one edit.

## Step 0 — survey the source

Answer these before mapping anything:

| Question | Why it matters |
| --- | --- |
| What are the *things* in this data (lot, unit, chamber, pipeline segment)? What are they named by, and are those names stable? | Entity tree + naming recipes. Identity must come from stable business codes — never time or equipment (I1). |
| Do any two different kinds of thing share one name field? | If keyed from different source fields or having different children → **different **`entity_type`s (I14 / L-SCK-01). |
| What gets measured, at what grain, at what rate and volume? | Picks the storage lane (decision tree below) — the single most consequential mapping choice. |
| Does the source report re-processing / re-testing? How does it distinguish passes? | RUN-level modeling; any source discriminator folds into the RUN `local_id` (D-039). |
| Are there limits or setpoints, and what governs them (test program, recipe)? | `param_spec` rows at the governing PROCESS entity. |
| Same quantity measured under different conditions (temperature/voltage corners)? | One `param_def` per condition + `param_condition` rows + shared concept (D-038). |
| What units? Same quantity known elsewhere under another name/unit? | Concept mapping + conversion pair (D-036/D-040). |
| Does the source name a product at all? | If not (SCADA tags, alarms, meters): land on the EQUIPMENT entity in `observation`; product attribution comes later via time windows (D-042). |
| What timestamps exist, in what timezone and format? | D-044: `attrs` times must be UTC ISO-8601; declare the source timezone in the pack spec. |
| How will the source re-send data (replays, corrections, replicates)? | Idempotency plan + the CORRECTION policy (I12, D-034). |

## Storage lane decision tree

For every parameter, in this order:

1. **Is it high-rate signal data (faster than SCADA cadence, roughly \< 1 s period)?** → **TRACE**. Raw samples go to Parquet, referenced by `file_ref`; they NEVER become rows (I5). If the tool also reports scalar summaries of the same quantity → **TRACE+SCALAR** (summaries as `measurement` rows, raw in the file).
2. **Is it slow telemetry on a standing asset (chamber over time, pipeline sensor), SCADA cadence or slower, needed in SQL?** → **TIMESERIES**. Rows in `observation`, keyed by `ts`. Numeric only (v1).
3. **Is it high-volume homogeneous scalar per item (STDF PTR/FTR: thousands of tests × many units)?** → **SCALAR\_BULK**. One Parquet per parent (row per unit/die, column per test) behind `file_ref`; zero `measurement` rows.
4. **Everything else (yields, bins, scores, states, snapshots)** → **SCALAR**. One `measurement` row per (item, param).

Wrong-lane symptoms: billions of `measurement` rows (should be SCALAR\_BULK); trace samples in `observation` (cadence rule violated); a TIMESERIES param minting fake entities per time window (I1 violated — that's what `observation` exists to prevent).

## Onboarding steps — write rows, in this order

To stand up a fresh instance: run the three DDL blocks from [01 · Schema Spec](./01-schema-spec-v2.2.md) in order, confirm `schema_migration` shows `v2.2`.

1. **Types** — INSERT `entity_type` rows. Set `domain` (PRODUCT / EQUIPMENT / PROCESS); `parent_type` stays inside one domain (I10). `type_code` ≤ 16 chars (D-027: `TEST_PROG_VER`, never `TEST_PROGRAM_VERSION`). One `type_code` per item kind (I14).
2. **Naming** — INSERT `entity_key_def` rows per type for this `source_code` (append-only, I3; canon rules only from the closed set `AS_IS` / `UPPER` / `DIGITS_PAD2` / `DIGITS_PAD3`, I13).
3. **Dictionary** — INSERT `param_def` rows for every measurement. Choose `grain` and `storage_class` deliberately — they decide every future query. Per-condition params: suffix the `native_id` (`t1001@125C`), add `param_condition` rows, share one `concept_id` (D-038).
4. **Concepts & units** — map params to `param_concept` where a cross-source quantity exists; when `unit` ≠ `canonical_unit`, set the `to_canonical_scale`/`_offset` pair (D-040 — mandatory at mapping review). Add `param_alias` rows for known synonyms.
5. **Layouts** — one `file_layout_set` per file shape with the right `lane`; one `file_layout` row per channel/test column. Irregular/timestamped traces: set `time_column` (D-041). Uniform traces: refs carry `sampling_hz`.
6. **Specs** — INSERT `param_spec` rows for limits/setpoints, keyed to the governing PROCESS-domain entity (guard-enforced, D-023). Setpoints: `spec_kind='SETPOINT'`, context = RECIPE\_VERSION (D-031). A revision = retire old + insert new in ONE transaction (D-034).
7. **Ingest** — see the loader contract below.
8. **Register** — record the pack in [06 · Case Registry](./06-case-registry.md). Anything that will not fit as rows goes to [04 · Known Limits](./04-known-limits-deferred.md) as a logged defect — **never quietly into **`attrs`.

## The loader contract

Every loader MUST:

1. **Open an **`ingest_batch` (`source_code`, `source_uri`, `kind='INGEST'`) and stamp `batch_id` on every row it writes (`measurement`, `link`, `file_ref`, `param_spec`, `observation`). Close the batch (`finished_at`) on commit.
2. **Resolve-or-create entities idempotently** — `ON CONFLICT (natural_key) DO UPDATE SET attrs = EXCLUDED.attrs`. Never INSERT a `natural_key` directly (I2) — build it through the resolver from `entity_key_def` recipes. A missing recipe for an incoming type FAILS the load (I13).
3. **Honor the conflict policy (I12/D-034):** a re-send with identical values lands on the same row, zero errors. A re-send with a **different** value (including a different `link` window — D-042) must FAIL LOUDLY. To overwrite deliberately: run a batch with `kind='CORRECTION'` — archive the old row to `measurement_audit` first, then overwrite. No silent overwrites, ever.
4. **Use the right conflict targets:** `entity` → `(natural_key)`; `measurement` → `(entity_id, param_id)`; `link` → `uq_link_natural (src, dst, kind, context)`; `file_ref` → `(storage_uri)`; `observation` → `(entity_id, param_id, ts)`.
5. **Validate times (D-044):** every timestamp written into `attrs` is UTC ISO-8601 with `Z`; keys end `_at`. Source-local times are converted using the timezone declared in the pack spec; unparseable/ambiguous time fails the load.
6. **Windows (D-042):** write `window_start`/`window_end` only from what the source provides — never synthesize an end. Bulk loads: skip the EXCLUDE-style row-by-row checking; run the set-based overlap validation before COMMIT (below).
7. **Bulk-load guard pattern (D-034):** inside the load transaction you may `ALTER TABLE link DISABLE TRIGGER trg_link_guard`, then run ONE set-based domain + cycle + overlap validation before COMMIT. Never commit unvalidated.
8. **Fail-loud philosophy:** wherever this page says "fails" — that means the load stops with a clear error naming the row and rule. Guessing, skipping silently, or "best effort" parsing are all bugs.

### Reference SQL

Overlap validation for windowed links (running-max form — the `lag()` form is WRONG, it misses nested overlaps; verified 2026-08-11):

```sql
SELECT count(*) AS overlapping
FROM (
  SELECT dst_entity_id, link_kind, window_start,
         max(window_end) OVER (PARTITION BY dst_entity_id, link_kind
                               ORDER BY window_start
                               ROWS BETWEEN UNBOUNDED PRECEDING AND 1 PRECEDING) AS prior_max_end
  FROM link
  WHERE is_active AND window_start IS NOT NULL
) s
WHERE prior_max_end IS NOT NULL AND window_start < prior_max_end;
-- must return 0 before COMMIT
```

Measurement upsert honoring the correction policy (loader-side check shape):

```sql
INSERT INTO measurement (entity_id, param_id, value_num, captured_at, batch_id)
VALUES (:eid, :pid, :val, :ts, :batch)
ON CONFLICT (entity_id, param_id) DO UPDATE
SET value_num = EXCLUDED.value_num, batch_id = EXCLUDED.batch_id
WHERE measurement.value_num IS NOT DISTINCT FROM EXCLUDED.value_num;
-- 0 rows affected + value differs => raise: value conflict, CORRECTION batch required
```

## What NOT to do — the failure catalog

| Don't | Because | Instead |
| --- | --- | --- |
| Key anything by timestamp or equipment | I1 — one wafer shatters into phantom items | RUN level; time in `attrs` |
| Dump an unmodeled shape into `attrs` | Every "put it in attrs" is a logged coverage defect | Log it in [04 · Known Limits](./04-known-limits-deferred.md), then model it |
| Discriminate two item kinds inside one `type_code` via `attrs` | I14 — poisons every generated view (L-SCK-01) | Two types, two truthful naming recipes |
| Unpivot traces into rows | I5 — storage and index explosion | TRACE lane, Parquet + `file_ref` |
| Store a parent roll-up as a `measurement` | I6 — drifting aggregates | Compute on read, or a clearly-derived table |
| Store the same setpoint as both measurement and spec | D-031 — two truths | `param_spec` SETPOINT only |
| Hand-write `natural_key` | I2 — resolver only | Resolve through `entity_key_def` |
| Edit an `entity_key_def` row | I3 — silent duplicates on next ingest | New version + explicit re-key |
| DELETE registry rows | I7 | `is_active = FALSE` |
| Silent-overwrite a differing re-send | I12 — untraceable data mutations | Fail loudly; CORRECTION batch |
| Guess a missing naming recipe or timezone | I13/D-044 | Fail the load |
| Put the window in any key | I1/D-034 rejection | Windows are context columns only |
| Write per-source SQL against unit-mixed values | D-040 exists for this | Convert through scale/offset at read |

## Sign-off gates — a pack is done when

1. Mapping spec reviewed and signed off (before any loader code).
2. Full load runs green; re-run produces **identical counts and zero errors** (idempotency proof).
3. A deliberately mutated re-send fails loudly; a CORRECTION batch then succeeds with the old value in `measurement_audit`.
4. Every `attrs` timestamp validates as UTC ISO-8601.
5. Row counts reconcile against the source (files × expected rows — state the arithmetic).
6. Pack registered in [06 · Case Registry](./06-case-registry.md) with lanes, tree, naming recipes, and any logged limits.
