# Universal DB — CORE_SOLUTION_DB (start here)
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/overview
> pageId: 375259277 · lastModified: 9월 20, 2026 · saved: 2026-10-12

**This space is the single source of truth for the AIBIZ Universal DB** (since 2026-08-11, decision D-037). One PostgreSQL schema that absorbs any industrial manufacturing data — without a schema change per domain or customer. Committed schema: **v2.2 — 18 tables, invariants I1–I14**, verified on PostgreSQL 16.13 before ratification. Phase: **verification & field test**. The former Notion pages are a **frozen, read-only archive** — never write there; links on [05 · Decision Log — D-001…D-050 (append-only)](./05-decision-log.md).

## Where to start

| If you are… | Read in this order | First thing to check |
| --- | --- | --- |
| **A data engineer** building a loader / onboarding a source | 1. [02 · Data Engineer Runbook — New Data Arrives, Do This](./02-data-engineer-runbook.md) — the how 2. [01 · Schema Spec — v2.2 (Committed Baseline)](./01-schema-spec-v2.2.md) — the what 3. [04 · Known Limits & Deferred Items — Read Before Field Testing](./04-known-limits-deferred.md) — the honest edges 4. [06 · Case Registry — Domain Packs](./06-case-registry.md) — packs + pack template | The storage lane for every parameter. It is the most consequential mapping choice you will make, and the hardest to undo. |
| **A backend engineer** SQL, services, views | 1. [03 · Backend Query Conventions — Reading the Universal DB](./03-backend-query-conventions.md) 2. [01 · Schema Spec — v2.2 (Committed Baseline)](./01-schema-spec-v2.2.md) 3. [04 · Known Limits & Deferred Items — Read Before Field Testing](./04-known-limits-deferred.md) | The instance's `schema_migration` table before you write one line of SQL. The spec describes the target, not the disk. |
| **A field tester or operator** | 1. [07 · Field-Test Plan — Verification Mode](./07-field-test-plan.md) — the six tests, gates fixed up front 2. [08 · Operations — Instances, Migrations, Retention, Guards](./08-operations.md) — migrations, retention, guards, version board 3. [04 · Known Limits & Deferred Items — Read Before Field Testing](./04-known-limits-deferred.md) | The gates were fixed *before* the runs. A run cannot move its own goalposts. |
| **New to the project** or asking "why is it like this?" | 1. This page, top to bottom 2. [05 · Decision Log — D-001…D-050 (append-only)](./05-decision-log.md) — D-001…D-050 | Nothing here changes silently. If it is not in the Decision Log, it did not happen. |
| **한국어 독자** | [01-KO · 스키마 명세 — v2.2 (확정 기준선)](./01-KO-schema-spec.md) [02-KO · 데이터 엔지니어 런북 — 새 데이터가 들어오면 이렇게](./02-KO-runbook.md) | 번역본입니다. 어긋나면 **영문 페이지가 정본**입니다. 실행할 DDL은 영문 01 페이지에만 있습니다. |

## Status at a glance — as of 2026-09-19

Live truth is on the pages linked in each row — this block is a summary, and a summary can go stale. It did once already: it claimed `sck` was on v1.1 for a month after D-050 measured it at v2.0.

| What | Where | State | Detail |
| --- | --- | --- | --- |
| Committed schema | [01 · Schema Spec — v2.2 (Committed Baseline)](./01-schema-spec-v2.2.md) | V2.2 RATIFIED | 18 tables, I1–I14. Full DDL stack executed and verified end-to-end on PostgreSQL 16.13 before ratification (D-038–D-048). Two places where the page's prose is not yet executable DDL are logged on 04. |
| Field tests | [07 · Field-Test Plan — Verification Mode](./07-field-test-plan.md) | TEST 1 · 7 OF 8 GATES | Test 1 ran 2026-09-19: G1–G5, G7, G8 pass; G6 blocked pending D-051. That unblocks Tests 2, 5 and 6. Tests 3 and 4 wait on **data delivery**, not on us. |
| Domain packs | [06 · Case Registry — Domain Packs](./06-case-registry.md) | 0 OF 4 LIVE | SEMES and SCK **Outdated** (re-verify against v2.2); Innotek and KOGAS **Planned**. SCK's instance is migrated, but its pack page is still incomplete. |
| Loaded instances | [08 · Operations — Instances, Migrations, Retention, Guards](./08-operations.md) | ALL CURRENT | `sck` migrated to **v2.2** on 2026-09-19 — the first migration run through the D-045 discipline. `innotek` stands empty on v2.2. `semes` / `kogas` stand up on v2.2 directly. `sck_raw` is retained as the v2.0 rollback, not a live instance. |
| Milestone 1 — coverage backlog | [04 · Known Limits & Deferred Items — Read Before Field Testing](./04-known-limits-deferred.md) | CLOSED | Every item resolved or formally deferred with a reopen trigger. Test 1 added a new category to that page: **specification gaps**, where a ratified page asserts behaviour no ratified DDL creates. |
| Milestone 2 — architecture verdict | [05 · Decision Log — D-001…D-050 (append-only)](./05-decision-log.md) | RATIFIED | D-033: relational truth in `link` + a derived, disposable graph projection. Benchmark verdicts still owed — they land in the field tests. |
| Decisions | [05 · Decision Log — D-001…D-050 (append-only)](./05-decision-log.md) | D-001…D-050 | Append-only. Next id continues at **D-051** — which is already spoken for: the `link_guard()` advisory lock drafted during Test 1. |

## What the thing actually is

One schema. Semiconductor etch, post-package test, camera-module test, solar, gas pipeline — and whatever comes next. New sensors, item types, naming schemes, relationship kinds and test conditions arrive as **rows in registry tables**, never as new columns or forked tables. One unforked schema; **one physical instance per customer** (D-026) — logically universal, physically per-project.

**Six concepts.** **ITEM** — what things exist and how they are named · **VALUE** — what gets measured, including time-series and cross-source vocabulary · **SPEC** — limits and setpoints · **FILE** — Parquet behind pointers, never unpivoted · **LINK** — every relationship between two things · **PROVENANCE** — which file wrote every number, and which schema version an instance runs.

**The test of universality:** onboarding a new domain or customer requires writing rows (a domain pack), not editing the schema. If a new case ever forces a schema change, that is a *defect in universality* — log it on [04 · Known Limits & Deferred Items — Read Before Field Testing](./04-known-limits-deferred.md) before patching anything.

## The three rules everyone must know

1. **Identity never comes from time or equipment.** Stable business codes only. A second processing pass is a new RUN, never a timestamped duplicate (I1).
2. **Check the parameter's storage lane before you query it.** File-lane parameters have zero relational value rows *by design* — an empty result is not a bug (I5 / I9).
3. **Ingestion is idempotent and provenance-stamped.** A re-send lands on the same row. A *changed* value fails loudly unless its batch is a flagged CORRECTION (I12).

## Two things that will bite you

**1 · An instance can lag the committed schema.** The spec describes the target, not the disk. Before writing SQL, a loader, or pack code against any instance: read its `schema_migration` table (absent = pre-v2.2) and check the version board on [08 · Operations — Instances, Migrations, Retention, Guards](./08-operations.md). Every instance happens to be current as of 2026-09-19 — read the table anyway, because this sentence is exactly the kind that goes stale.

**2 · "Put it in **`attrs`**" is never the answer.** Data that will not fit as rows is a **coverage defect**, and every one gets logged on [04 · Known Limits & Deferred Items — Read Before Field Testing](./04-known-limits-deferred.md) — never dumped silently into `attrs`, never patched locally into the schema.

**3 · A ratified page can describe behaviour that no ratified DDL creates.** Test 1 found two: the `link_guard()` advisory lock and the I12 correction policy both live in comments, not in executable SQL. When a page says "the guard does X", verify against `pg_get_functiondef` before relying on it. Both are logged on 04.

## How changes happen

Nothing changes silently. Spot it → fetch the live page → draft the exact edit plus its `D-###` entry → get approval → apply, log on [05 · Decision Log — D-001…D-050 (append-only)](./05-decision-log.md), then sweep the sibling pages for consistency. Ratified schema changes ship **only** as numbered migration scripts through the [08 · Operations — Instances, Migrations, Retention, Guards](./08-operations.md) discipline (dev → verify → promote, with a `schema_migration` row) — never hand-typed ALTERs.

**Solved problems stay solved.** Before any schema or architecture proposal: scan the decision log for a rejected alternative and address its original rationale by ID; walk *every* invariant on the live 01 page and mark it satisfied as-is / satisfied differently / consciously dropped; scan the pack lessons on [06 · Case Registry — Domain Packs](./06-case-registry.md). A proposal missing those scans is incomplete.

Ratify-then-validate is legal (the D-024 pattern): validation lives in the field tests, and a flaw found there is fixed by a *superseding* decision — never a silent edit.

## Finish line for this phase

Every test on [07 · Field-Test Plan — Verification Mode](./07-field-test-plan.md) green — or closed by a superseding decision — with the SCK and SEMES packs back to **Live**, every loaded instance on the committed schema version, and the pending D-033 / D-035 benchmark verdicts recorded on [05 · Decision Log — D-001…D-050 (append-only)](./05-decision-log.md). "Still testing" with no gate movement is not an end state.

## Every page in this space

| Page | What it is | Go here when |
| --- | --- | --- |
| [01 · Schema Spec — v2.2 (Committed Baseline)](./01-schema-spec-v2.2.md) 한국어: 01-KO | The committed baseline: objective, invariants I1–I14, full data model, canonical DDL, column dictionary, ERD, status snapshot. | You need the truth about a table or column, or you are standing up a fresh instance. |
| [02 · Data Engineer Runbook — New Data Arrives, Do This](./02-data-engineer-runbook.md) 한국어: 02-KO | New data arrives → do this. Source survey, storage-lane decision tree, write-order, loader contract, failure catalog, sign-off gates. | You are onboarding a source or writing/fixing a loader. |
| [03 · Backend Query Conventions — Reading the Universal DB](./03-backend-query-conventions.md) | Version and lane checks, query cookbook, derived-layer rules. | You are writing SQL, a service, or a view over an instance. |
| [04 · Known Limits & Deferred Items — Read Before Field Testing](./04-known-limits-deferred.md) | The model's honest edges: specification gaps, deferred items with reopen triggers, edges closed by convention, unreconciled figures, and where coverage hits get logged. | Something will not fit as rows — or you want to know what we already know is missing. |
| [05 · Decision Log — D-001…D-050 (append-only)](./05-decision-log.md) | D-001…D-050, append-only, next id D-051. The D-037–D-048 close-out package and D-050 each have their own full-record child page. | You are asking "why is it like this?", or you need the next decision id. |
| [06 · Case Registry — Domain Packs](./06-case-registry.md) | One entry per onboarded or planned source, plus pack lessons (e.g. L-SCK-01) and the 10-section pack template new packs must follow. | You are authoring a new pack page, or checking what an existing pack did. |
| [07 · Field-Test Plan — Verification Mode](./07-field-test-plan.md) | The six tests, their gates fixed up front, and the status board. | You are running or reporting a field test. This is the board that closes the phase. |
| [08 · Operations — Instances, Migrations, Retention, Guards](./08-operations.md) | Migration discipline, retention and tiering, guard operations, backups, instance version board. | You are migrating, promoting, purging, or need to know what version an instance runs. |

*Found something on any page that contradicts reality? The live page wins over any summary — including this one. Say so out loud, then fix it through the loop above.*
