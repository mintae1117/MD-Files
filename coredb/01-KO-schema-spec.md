# 01-KO · 스키마 명세 — v2.2 (확정 기준선)
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/376930426/01-KO+v2.2
> pageId: 376930426 · lastModified: 8월 14, 2026 · saved: 2026-10-12

**이 페이지는 번역본입니다. 정본(正本)이 아닙니다.** 정본은 영문 [01 · Schema Spec — v2.2 (Committed Baseline)](./01-schema-spec-v2.2.md) 페이지입니다. 두 페이지가 어긋나면 **영문 페이지가 이깁니다.** 번역 기준: 스키마 **v2.2**, 2026-08-12 시점. 비준된 변경(D-###)이 생기면 이 페이지는 **재동기화 전까지 낡은 것으로 취급**하십시오.

> **확정 기준선 — 스키마 v2.2, 테이블 18개, 불변규칙 I1–I14** (D-030 기반 + D-033–D-036 + 마감 패키지 D-037–D-048, 2026-08-11 비준). 이 페이지는 *비준된 것*만 기술합니다. 변경은 [05 · Decision Log](./05-decision-log.md)에 비준된 `D-###`을 통해서만 일어납니다. 아래 v2.2 DDL 전체 스택은 비준 전 2026-08-11에 **PostgreSQL 16.13에서 실행·검증**되었습니다. 과거 이력(v1.x/v2.0/v2.1 페이지, 제안 P-001–P-005, 외부 리뷰 R-001)은 동결된 Notion 아카이브에 있습니다.

이 문서는 엔지니어용 참조입니다: 목적, 불변규칙, DDL을 포함한 전체 데이터 모델, 그리고 컬럼 딕셔너리. **일상 작업 방법**은 [02 · Data Engineer Runbook](./02-data-engineer-runbook.md) (한국어: 02-KO)과 [03 · Backend Query Conventions](./03-backend-query-conventions.md)에 있습니다.

## 목적

**어떤** 산업 제조 도메인이든 흡수할 수 있는 하나의 스키마 — 식각 장비, 패키지 후 테스트, 카메라 모듈 테스트, 태양광 탭버, 가스 배관, 그리고 앞으로 올 무엇이든 — **도메인별·고객별 스키마 변경 없이**. 새 센서, 새 아이템 타입, 새 명명 체계는 **레지스트리 테이블의 행**으로 들어옵니다. 새 컬럼이나 포크된 테이블이 아닙니다. 여러 소스 시스템이 *같은* 데이터베이스에 공존합니다.

**배포 모델 (D-026): 논리적으로는 범용, 물리적으로는 프로젝트별.** 스키마는 하나이며 절대 포크되지 않지만, 고객/프로젝트마다 자기 물리 인스턴스를 운영합니다. "하나의 DB에 여러 소스"는 *모든 인스턴스가 갖는 능력*이며, 모든 고객이 하나를 공유한다는 뜻이 아닙니다.

## 현장 용어 → 모델 용어

스키마는 DB 언어를 쓰고, 사용자는 현장 언어를 씁니다. 이 표가 다리입니다.

| 현장에서 말하는 것 | DB가 저장하는 방식 |
| --- | --- |
| PV — 실제 측정값 | `measurement` 행 (SCALAR), 또는 `file_ref` 뒤의 Parquet (TRACE / SCALAR\_BULK). 파라미터의 `grain`에서. |
| SV — 설정값, 의도 | `param_spec` 행, `spec_kind='SETPOINT'`, 맥락 = RECIPE\_VERSION (단일 보관처 — D-031) |
| 테스트 한계값(limit) | `param_spec` 행, `spec_kind='LIMIT'`, 맥락 = TEST\_PROG\_VER |
| 프로파일 / 램프 설정값 (숫자 하나가 아닌 곡선) | RECIPE\_VERSION 아래의 RECIPE\_STEP 자식 엔티티, 또는 거기에 앵커된 TRACE 레인 파일 (D-031 패턴) |
| 레시피 — 장비가 돌리는 프로그램 | `RECIPE` → `RECIPE_VERSION` 엔티티 (PROCESS 도메인) |
| SVID / 테스트 번호 | `param_def.native_id` |
| 테스트 조건 / 코너 (25 °C vs 125 °C) | 조건마다 `param_def` 하나 + `param_condition` 행; 같은 `param_concept`가 묶어줌 (D-038) |
| 랏 이력 / 이동 경로 | `link` 행, 종류 PROCESSED\_IN / TESTED\_ON (맥락 = 실행된 레시피/프로그램 버전) |
| 구성 이력 / BOM | `link` 행, 종류 ASSEMBLY / MERGE / SPLIT |
| 챔버 텔레메트리 시계열 / 배관 모니터링 | `observation` 행 (TIMESERIES 레인, D-035) |
| "이 숫자는 어느 파일에서 왔나" | `measurement.batch_id` → `ingest_batch.source_uri` (조인 한 번, D-034) |

## "진정한 범용"의 뜻 — 판정 기준

이 DB가 범용인 조건: **새 도메인이나 새 고객을 온보딩할 때 행을 쓰면 되고(도메인 팩), 스키마를 고치거나 DB를 포크하지 않아도 된다.** 구체적으로:

- 새 측정항목 → 새 `param_def` 행. `ALTER TABLE` 절대 아님.
- 새 아이템 타입이나 명명 체계 → 새 `entity_type` / `entity_key_def` 행.
- 새 고속 신호 → `file_ref` (lane=TRACE) + Parquet을 가리키는 `file_layout` 행.
- 새 관계 종류(라우팅 또는 계보) → 자기 도메인 규칙을 지닌 `link_type` 행. 새 브리지 테이블 절대 아님.
- 새 테스트 조건 → 새 `param_def` + `param_condition` 행 (D-038). 키 변경 절대 아님.
- 새 *소스 시스템* → 같은 테이블 안의 새 `source` / `source_code` 네임스페이스. 배포 내 새 DB 없음. 새 *고객* → 포크되지 않은 같은 스키마의 새 물리 인스턴스 (D-026) — 여전히 스키마 변경 0.
- 저장 레인(**SCALAR** 행 / **SCALAR\_BULK** 컬럼형 Parquet / **TRACE** 원시 트레이스 Parquet / **TRACE+SCALAR** / **TIMESERIES** 시간 키 행)이 별도 저장장치 없이 데이터를 덮는다.

새 사례가 스키마 변경을 강제한다면 그것은 **범용성의 결함**입니다 — 스키마를 패치하기 전에 [04 · Known Limits & Deferred Items](./04-known-limits-deferred.md)에 기록하십시오.

## 헌법 — 불변규칙 I1–I14

확정 기준선에 구속력이 있습니다. 모든 항목은 **이식 가능한 요구사항**(D-029)이기도 합니다. 제안이 어느 하나에 도전하려면 요구사항 점검(그대로 충족 / 다르게 충족 / 실패 모드를 명시하고 의식적으로 포기)을 거쳐 `D-###`으로 비준되어야 합니다 — 조용한 수정은 절대 안 됩니다.

1. **I1 — 식별자는 절대 이벤트 메타데이터가 아니다.** `start_time`, `equipment`, 그 밖의 이벤트별 필드로 키를 만들지 마십시오 — 그것들은 `attrs`로 갑니다. 두 번의 처리 패스는 **RUN** 레벨에서 구분합니다(`local_id = STEP + '#' + pass`). 타임스탬프로 구분하지 않습니다. `link`의 시간창 컬럼(D-042)은 맥락이며 키가 아닙니다.
2. **I2 — **`natural_key`는 파생 전용입니다. 리졸버만 씁니다. 직접 INSERT/UPDATE 금지. `local_id` 안에 `/` 금지.
3. **I3 — **`entity_key_def`는 추가 전용(append-only)입니다. 레시피 변경 = 새 `version` INSERT + 명시적 re-key 마이그레이션. 레시피 행을 UPDATE/DELETE 하지 마십시오.
4. **I4 — 조인은 **`entity_id`로 합니다. `natural_key`는 해석·표시·접두어 필터(`LIKE 'LOT2291/%'`) 전용입니다.
5. **I5 — 원시 트레이스는 절대 관계형 행이 되지 않습니다.** 고속 질문은 TRACE 레인 Parquet을 DuckDB/pandas로 읽습니다. 샘플을 `measurement`나 `observation`으로 펼치지 마십시오 — TIMESERIES 레인은 케이던스 규칙(SCADA 속도 및 그보다 느림, 레인 가드로 강제)으로 한정됩니다(D-035).
6. **I6 — 롤업은 부모 그레인에서 계산하며, 부모 그레인 measurement로 저장하지 않습니다.** 명확히 분리된 파생 산출물로서의 롤업 *테이블*은 괜찮습니다.
7. **I7 — 레지스트리는 소프트 삭제만.** `is_active = FALSE`로 비활성화합니다. measurement가 딸린 행은 절대 DELETE 하지 마십시오. D-043으로 확장: 삭제(purge)된 파일의 카탈로그 행은 tombstone(`tier='PURGED'`)이 되며 삭제되지 않습니다.
8. **I8 — 와이드 테이블, 그래프 프로젝션, 모든 파생 데이터셋은 폐기 가능합니다.** 절대 진실의 원천이 아닙니다. 모든 파생 데이터셋은 인스턴스·스키마 버전·배치 워터마크를 고정한 manifest를 지닙니다(D-046).
9. **I9 — 하나의 SVID, 하나의 정의.** 파라미터의 의미는 `param_def`에만 존재합니다. `file_layout`은 `param_id` → 물리 컬럼 매핑만 하고 이름·단위를 저장하지 않습니다. 별칭과 개념은 그 하나의 정의를 *가리킵니다*(D-036). 서로 다른 조건에서 측정된 파라미터는 `param_concept`를 공유하는 별개의 `param_def`입니다(D-038).
10. **I10 — 차원은 트리를 통해 교차하지 않습니다.** `parent_id`는 하나의 `domain`(PRODUCT / EQUIPMENT / PROCESS) 안에 머무릅니다. 차원 간 관계는 `link_type` 규칙이 교차를 허용하는 `link` 행으로만 존재하며, `entity_id`로 조인합니다. 교차 규칙은 데이터이고, 하나의 범용 fail-loud 가드가 강제합니다(D-030).
11. **I11 — (아이템, 파라미터)당 스칼라 하나.** `measurement` PK는 `(entity_id, param_id)`입니다. *제품* 아이템의 이력은 엔티티들(RUN, wafer, …)에 걸쳐 표현됩니다. **고정 설비**의 이력은 TIMESERIES 레인(`observation`, `ts` 키)에 있습니다 — 반복된 `measurement` 행이 아닙니다.
12. **I12 — 적재는 멱등적입니다.** resolve-or-create는 재전송된 레코드를 *같은* 행에 떨어뜨립니다 — 중복은 절대 없습니다. **다른 값**을 담은 재전송(`link` 시간창이 다른 경우 포함 — D-042)은 배치가 CORRECTION으로 표시되지 않는 한 크게 실패합니다. CORRECTION은 이전 값을 먼저 보관합니다. 조용한 덮어쓰기는 없습니다.
13. **I13 — 정규화는 닫힌 집합이며 fail-loud입니다.** 규칙 집합(`AS_IS`, `UPPER`, `DIGITS_PAD2`, `DIGITS_PAD3`)은 불변입니다 — 새 동작은 새로 *명명된* 규칙입니다. 레시피가 없으면 크게 실패하며, 절대 추측하지 않습니다. 시간에도 적용됩니다: `attrs` 타임스탬프는 UTC ISO-8601이어야 하며, 아니면 적재가 실패합니다(D-044).
14. **I14 — 하나의 **`type_code`, 하나의 아이템 종류. 한 타입의 모든 엔티티는 같은 소스 필드로 키가 만들어지고 트리에서 같은 역할을 합니다. 그렇지 않다면 서로 다른 타입입니다. `attrs`로 아이템 종류를 구분하지 마십시오(D-028, 팩 교훈 L-SCK-01에서 도출).

## 데이터 모델 — 테이블 18개, 개념 6개

**ITEM** (`entity_type`, `entity_key_def`, `entity`) · **VALUE** (`param_def`, `param_condition`, `measurement`, `observation`, `param_concept`, `param_alias`) · **SPEC** (`param_spec`) · **FILE** (`file_layout_set`, `file_layout`, `file_ref`) · **LINK** (`link`, `link_type`) · **PROVENANCE** (`ingest_batch`, `measurement_audit`, `schema_migration`).

두 아이템을 관계짓는 방법 하나, 파일을 가리키는 방법 하나, 적재된 모든 행에 `batch_id` 하나, 인스턴스마다 조회 가능한 스키마 버전 하나. 새 측정·아이템·명명·관계·조건 타입은 모두 **행**이며 스키마 변경이 아닙니다.

| 테이블 | 한 행은… | 알아야 할 핵심 컬럼 |
| --- | --- | --- |
| `entity_type` | 아이템의 한 종류 | `type_code`, `parent_type`, `domain` |
| `entity_key_def` | 한 타입의 명명 레시피 중 하나의 코드 (소스별) | `type_code`, `source_code`, `version`, `seq`, `field_name`, `prefix`, `canon_rule` |
| `entity` | 추적되는 아이템 하나 | `entity_id` (조인용), `parent_id` (트리), `local_id`, `natural_key`, `attrs` |
| `param_def` | 측정의 한 종류 (딕셔너리) | `param_id`, `source`, `native_id`, `grain`, `value_type`, `storage_class`, `concept_id`, `to_canonical_scale`/`_offset` (D-040) |
| `param_condition` | 파라미터를 한정하는 조건 축 하나 (D-038) | PK (`param_id`, `condition_key`); `value_num`/`value_text` |
| `measurement` | 스칼라 값 하나 | PK (`entity_id`, `param_id`); `value_num`/`value_text`/`value_bool`; `captured_at`; `batch_id` |
| `observation` | 고정 설비의 시간 키 텔레메트리 포인트 하나 (TIMESERIES, D-035) | PK (`entity_id`, `param_id`, `ts`); `ts`로 파티셔닝 |
| `param_concept` | 소스 간 표준 개념 하나 | `concept_id`, `concept_code`, `canonical_unit` |
| `param_alias` | 파라미터로 해석되는 동의어 하나 | PK (`param_id`, `alias`); `kind` |
| `param_spec` | 어떤 맥락에서 한 파라미터의 스펙 밴드(한계/설정값) 하나 | PK (`param_id`, `context_entity_id`, `spec_kind`, `valid_from`); ACTIVE 밴드는 하나만 보장 |
| `file_layout_set` | 레인이 태깅된 파일 컬럼 레이아웃 하나 | `layout_id`, `lane`, `time_column` (D-041) |
| `file_layout` | 레이아웃의 컬럼(채널/테스트) 하나 | PK (`layout_id`, `param_id`); `column_name` |
| `file_ref` | Parquet 파일 하나를 가리키는 포인터 | `storage_uri` PK, `lane`, `anchor_entity_id`, `layout_id`, `row_grain`, `sampling_hz`, `window_start`/`window_end`, `tier` (D-043), `batch_id` |
| `link` | 두 아이템 사이의 관계 하나 — 라우팅 이벤트 또는 계보 엣지 | PK `link_id`; UNIQUE NULLS NOT DISTINCT (`src`, `dst`, `link_kind`, `context_entity_id`); `window_start`/`window_end` + `ts_window` (D-042); `batch_id` |
| `link_type` | 허용된 관계 종류 하나 + 그 도메인 규칙 | `link_kind`; `src_domain`, `dst_domain`, `context_domain`, `acyclic` |
| `ingest_batch` | 적재/보정 배치 하나 — 그것이 쓴 모든 행의 출처 | `batch_id`, `source_code`, `source_uri`, `kind` (INGEST / CORRECTION) |
| `measurement_audit` | CORRECTION 배치가 보관한 대체된 값 하나 (추가 전용) | (`entity_id`, `param_id`), 이전 값, `superseded_by_batch` |
| `schema_migration` | 적용된 DDL 패키지 하나 — 스키마 자체의 출처 (D-045) | `version` PK, `applied_at`, `script_sha256`, `decision_ids` |

중요한 세부: `value_type` ∈ {FLOAT, STRING, BOOL, CATEGORICAL}; `storage_class` ∈ {SCALAR, SCALAR\_BULK, TRACE, TRACE+SCALAR, TIMESERIES}; `entity`의 유일성은 `UNIQUE NULLS NOT DISTINCT (type_code, parent_id, local_id)`이므로 루트 아이템이 조용히 중복될 수 없습니다. `measurement` 행마다 값 컬럼은 정확히 하나만 채워집니다(CHECK).

## 정본 DDL — v2.2

**실행할 DDL은 여기가 아닙니다.** 정본 v2.2 DDL 3개 블록은 영문 [01 · Schema Spec](./01-schema-spec-v2.2.md) 페이지에만 존재합니다. 실제 인스턴스를 세울 때는 **그 페이지의 블록을 순서대로** 실행하십시오. 이유: 정본 DDL 사본이 둘이면 언젠가 어긋나고, 누군가 낡은 DDL을 실행합니다 — I9/D-031이 막으려는 바로 그 "두 개의 진실" 문제입니다. 아래는 각 블록이 *무엇을 하는지*에 대한 한국어 안내입니다.

### 블록 1 — v2.0 기반 (테이블 11개, D-030)

기반 스키마를 만듭니다: `entity_type` · `entity_key_def` · `entity` · `param_def` · `measurement` · `file_layout_set` · `file_layout` · `file_ref` · `param_spec` · `link_type` · `link` — CREATE TABLE 11개. 여기에 fail-loud 가드 함수 4개(`file_ref_lane_guard()`, `file_layout_set_lane_freeze()`, `param_spec_guard()`, `link_guard()`)와 그에 대응하는 트리거 4개, 인덱스 3개(`ix_file_ref_anchor`, `ix_link_dst`, `ix_link_kind`), 그리고 시드 INSERT 3개(`link_type` 5종, 범용 `entity_type` 4개, 그 명명 레시피)가 포함됩니다. `link_guard()`가 핵심입니다 — src/dst/context 도메인과 종류별 비순환성을 검사하는 **단 하나의 범용 가드**로, 관계 종류마다 브리지 테이블을 만들지 않아도 되게 합니다.

### 블록 2 — v2.0 → v2.1 델타 (D-034 · D-035 · D-036)

세 가지를 더합니다. (1) **정합성 강화**(D-034): `param_spec`에 `valid_from`을 넣어 이력을 갖게 하고 ACTIVE 밴드가 하나임을 부분 유니크 인덱스로 보장, `link`에 대리키 `link_id` + 자연키 제약 `uq_link_natural`, 그리고 출처 체계 — `ingest_batch` 테이블과 4개 테이블의 `batch_id` 컬럼, 그리고 추가 전용 `measurement_audit`. (2) **TIMESERIES 레인**(D-035): `ts`로 범위 파티셔닝된 `observation` 테이블(기본 월 단위, v1은 숫자형만). (3) **어휘 계층**(D-036): `param_concept`와 `param_alias`, 그리고 `param_def.concept_id`. 유의: `storage_class`에 'TIMESERIES'가 생기는 것은 *행 값*일 뿐 ALTER가 아닙니다.

### 블록 3 — v2.1 → v2.2 델타 (마감 패키지, D-038/D-040/D-041/D-042/D-043/D-045)

여섯 가지: `param_condition` 테이블로 조건 모델(D-038); `param_def`에 `to_canonical_scale`/`_offset` 쌍으로 단위 환산(D-040, 표준값 = 값 × scale + offset, 둘 다 있거나 둘 다 없어야 함); `file_layout_set.time_column`으로 불규칙·타임스탬프 트레이스(D-041); `link`에 시간창 컬럼 + 생성 컬럼 `ts_window` + 부분 GiST 인덱스(D-042 — **EXCLUDE 제약은 쓰지 않습니다**, 대량 삽입이 32배 느려지는 것이 측정되었기 때문); `file_ref.tier`로 보존 계층화(D-043, HOT/COLD/PURGED); `schema_migration` 레지스트리(D-045)와 v2.2 행 삽입.

**인스턴스는 명세보다 뒤처질 수 있습니다.** SQL·로더·팩 코드를 쓰기 전에 그 인스턴스의 `schema_migration` 테이블을 확인하십시오(없으면 v2.2 이전). 명세는 *목표*를 기술하며, 디스크에 있는 것을 기술하지 않습니다. 인스턴스 버전 보드는 [08 · Operations](./08-operations.md)에 있습니다.

## 세 개의 차원과 계보 — 규칙

**설비 (D-015/D-016/D-022/D-030).** 챔버와 장비는 두 번째 `domain` 트리(`TOOL` → `CHAMBER`)의 평범한 엔티티이며, `parent_id`로 제품 트리에 연결되지 않습니다. 다리는 제품 엔티티(WAFER, 재처리가 있으면 RUN)에서 설비 엔티티로 향하는 PROCESSED\_IN / TESTED\_ON 종류의 `link`입니다 — 제품↔설비 교차의 유일한 합법 경로입니다. 챔버는 절대 식별자가 아닙니다(I1). 설비의 현재값 지표(마지막 PM 날짜, 결함 상태)는 CHAMBER 그레인의 평범한 `measurement` 행이며 **스냅샷 전용**입니다(D-022). 고정 설비의 시간 이력은 `observation`에 있습니다(D-035).

**공정 레시피 (D-017/D-018/D-030/D-031).** 레시피는 *공정 명세*이며 세 번째 차원입니다: `RECIPE` → `RECIPE_VERSION`, 각 버전이 자기 엔티티입니다. 이것은 `link.context_entity_id`로 라우팅 이벤트에 실려 갑니다 — 한 행이 *어떤 제품이, 어떤 챔버에서, 어떤 레시피로*를 동시에 답합니다. 스칼라 설정값은 `param_spec`에 있습니다(`spec_kind='SETPOINT'`, 맥락 = RECIPE\_VERSION) — SV의 단일 보관처입니다(D-031). 설정값을 measurement로 저장하는 방식은 폐기되었습니다. 곡선/램프 설정값은 RECIPE\_STEP 자식 엔티티 또는 그 버전에 앵커된 TRACE 레인 파일입니다. 참고: `entity_key_def`의 *명명 레시피*는 전혀 다른 설정입니다 — 두 가지가 "레시피"라는 단어를 공유합니다.

**계보 (D-019/D-030).** 조립/병합/분할 계보는 트리 *옆에* 있는 그래프입니다: PRODUCT 전용 종류 ASSEMBLY / MERGE / SPLIT의 `link` 행(비순환, 가드로 강제). `parent_id`는 유일한 명명/포함 부모로 남고, 엣지는 절대 식별자나 `natural_key`에 관여하지 않습니다. 일반적인 다이싱(출력이 소스 아래에 명명되는 경우)은 트리에 머물고, SPLIT 링크는 출력이 소스 아래에 명명되지 *않는* 경우만을 위한 것입니다. 잘못된 엣지는 소프트 은퇴시키며 삭제하지 않습니다.

**관계에 대한 판정 (D-033).** `link`가 모든 관계의 유일한 진실입니다. 다중 홉 순회는 **파생 그래프 프로젝션**이 담당합니다 — 워터마크 기준으로 `link WHERE is_active`에서 재생성되며, 폐기 가능하고, 직접 쓰이지 않습니다(I8). 그래프 저장소를 *기록 시스템의 대체물*로 쓰는 방안은 명시적으로 비준되지 않았습니다.

## 스키마 다이어그램 (ERD)

ERD의 Mermaid 소스는 영문 [01 · Schema Spec](./01-schema-spec-v2.2.md) 페이지에 있습니다 (DDL과 같은 이유로 사본을 두지 않습니다). 읽는 방법: crow's foot 표기로 `||`는 1, `o{`는 다(多)입니다. `entity`가 허브입니다 — 트리를 위해 자기 자신을 참조하고, `link`에서 세 가지 역할(소스 / 대상 / 맥락)을 합니다.

## 컬럼 딕셔너리 — v2.2 추가 및 변경

v2.1 전체 컬럼 딕셔너리는 동결된 Notion Guideline에 그대로 보존되어 있습니다([05 · Decision Log](./05-decision-log.md)에서 링크). 거기 있는 내용은 모두 유효합니다. v2.2가 추가한 것:

| 테이블 | 컬럼 | 설명 | 참조 |
| --- | --- | --- | --- |
| `param_def` | `to_canonical_scale` / `to_canonical_offset` | 개념의 `canonical_unit`으로 가는 선형 변환 (표준값 = 값 × scale + offset). 둘 다 또는 둘 다 없음 CHECK. `unit` ≠ `canonical_unit`이면 매핑 리뷰에서 필수 | — (D-040) |
| `param_condition` | `param_id` | 이 조건이 한정하는 파라미터 | PK; FK → `param_def.param_id` (D-038) |
| `param_condition` | `condition_key` | 조건 축: 'TEMP\_C', 'VDD\_V', 'FREQ\_HZ', … | PK |
| `param_condition` | `value_num` / `value_text` | 조건의 값 (정확히 하나만 채움, CHECK) | — |
| `file_layout_set` | `time_column` | 불규칙/타임스탬프 트레이스의 파일 내 타임스탬프 컬럼. NULL = 균일 샘플링(`file_ref.sampling_hz`) | — (D-041) |
| `link` | `window_start` / `window_end` | 이벤트의 점유 시간창 (맥락이며 절대 식별자 아님). NULL 허용. 순서 CHECK | — (D-042) |
| `link` | `ts_window` | STORED 생성 컬럼 `tstzrange(window_start, window_end, '[)')`. 부분 복합 GiST 인덱스 (dst, kind, ts\_window) | — (D-042) |
| `file_ref` | `tier` | HOT / COLD / PURGED (CHECK). PURGED = tombstone, 행은 절대 삭제되지 않음. 독자는 HOT이 아니면 크게 실패 | — (D-043) |
| `file_ref` | `tier_changed_at` | tier가 마지막으로 바뀐 시점 | — (D-043) |
| `schema_migration` | `version` | 적용된 스키마 버전('v2.2'). 인스턴스 버전 = `applied_at` 기준 최신 행 | PK (D-045) |
| `schema_migration` | `applied_at`, `script_sha256`, `decision_ids`, `note` | 언제, 정확히 무엇이 실행되었고, 어떤 결정을 구현하는지 | — (D-045) |

## 상태 스냅샷 (2026-08-11)

| 항목 | 상태 |
| --- | --- |
| 스키마 v2.2 비준 (D-038–D-048 마감) | 완료 — DDL을 PostgreSQL 16.13에서 검증 |
| 마일스톤 1 (커버리지 백로그 확정) | **종료** — 모든 항목이 해결되었거나 재개 트리거와 함께 공식 보류 |
| 마일스톤 2 (아키텍처 판정) | 판정 비준 (D-033). 벤치마크는 현장 테스트 모드에서 수행 |
| `sck` 인스턴스 | 여전히 **v1.1** — v1.1 → v2.2를 한 번에 마이그레이션 (게이트: P-001 + P-002 §6 + D-038 코너 적재). 인스턴스에 SQL을 쓰기 전에 항상 `schema_migration`을 확인하십시오. |
| SCK / SEMES 팩 | 낡음 — 현장 테스트에서 재검증 |
| Innotek / KOGAS | 계획됨 — 현장 테스트 사례 |

*이 페이지는 비준된 결정으로만 바뀝니다(그리고 영문 정본이 바뀐 뒤에 재동기화됩니다). 문제를 발견했다면 *[*04 · Known Limits & Deferred Items*](./04-known-limits-deferred.md)*에 기록하십시오 — 조용히 패치하지 마십시오.*
