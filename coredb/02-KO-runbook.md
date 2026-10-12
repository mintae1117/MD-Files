# 02-KO · 데이터 엔지니어 런북 — 새 데이터가 들어오면 이렇게
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/376930459/02-KO
> pageId: 376930459 · lastModified: 8월 13, 2026 · saved: 2026-10-12

**이 페이지는 번역본입니다. 정본(正本)이 아닙니다.** 정본은 영문 [02 · Data Engineer Runbook](./02-data-engineer-runbook.md) 페이지입니다. 두 페이지가 어긋나면 **영문 페이지가 이깁니다.** 번역 기준: 스키마 **v2.2**, 2026-08-12 시점. 비준된 변경(D-###)이 생기면 이 페이지는 **재동기화 전까지 낡은 것으로 취급**하십시오.

> **대상: 새 소스를 온보딩하거나 로더를 유지보수하는 데이터 엔지니어.** 스키마 참조는 [01-KO · 스키마 명세](./01-KO-schema-spec.md) (정본: [01 · Schema Spec](./01-schema-spec-v2.2.md)). 이 페이지를 한 문장으로: **매핑을 먼저 문서로 합의하고, 정해진 순서로 행을 쓰고, 모든 적재를 멱등적이며 출처가 찍히게 만들고, 추측하지 말고 크게 실패하라.**

## 황금률 — 코드보다 설계가 먼저

파서부터 시작하지 마십시오. 모든 소스는 설계 우선으로 온보딩합니다:

1. **소스를 조사**하고(아래 0단계) 매핑을 초안으로 작성합니다: 엔티티 트리, 파라미터, 레인, 스펙, 관계.
2. 여기 있는 체크리스트로 **매핑을 리뷰**합니다 — I14 위반, 레인 선택 실수, 타임스탬프를 식별자로 쓰는 버그가 *싸게* 잡히는 지점입니다.
3. **매핑에 대한 승인**을 받습니다 (문서화된 팩 명세 — 형식은 [06 · Case Registry](./06-case-registry.md) 참조).
4. 그 다음에야 로더를 작성합니다.

적재 후에 발견된 매핑 실수는 재적재와 (경우에 따라) 보정 사이클을 요구합니다. 리뷰에서 발견되면 수정 한 번으로 끝납니다.

## 0단계 — 소스 조사

무엇이든 매핑하기 전에 이 질문들에 답하십시오:

| 질문 | 왜 중요한가 |
| --- | --- |
| 이 데이터에서 *사물*은 무엇인가 (랏, 유닛, 챔버, 배관 구간)? 무엇으로 이름 붙는가, 그 이름은 안정적인가? | 엔티티 트리 + 명명 레시피. 식별자는 안정적인 업무 코드에서 나와야 합니다 — 시간이나 설비에서 나오면 안 됩니다 (I1). |
| 서로 다른 두 종류의 사물이 하나의 이름 필드를 공유하는가? | 서로 다른 소스 필드로 키가 만들어지거나 자식이 다르면 → **서로 다른 **`entity_type` (I14 / L-SCK-01). |
| 무엇이 측정되는가, 어떤 그레인에서, 어떤 주기와 양으로? | 저장 레인을 결정합니다(아래 결정 트리) — 매핑에서 가장 파급력이 큰 선택입니다. |
| 소스가 재처리/재검사를 보고하는가? 패스를 어떻게 구분하는가? | RUN 레벨 모델링. 소스의 구분자는 RUN `local_id`에 접힙니다 (D-039). |
| 한계값이나 설정값이 있는가, 무엇이 그것을 지배하는가(테스트 프로그램, 레시피)? | 지배하는 PROCESS 엔티티에 붙는 `param_spec` 행. |
| 같은 양을 서로 다른 조건(온도/전압 코너)에서 측정하는가? | 조건마다 `param_def` 하나 + `param_condition` 행 + 공유 개념 (D-038). |
| 단위는? 같은 양이 다른 곳에서 다른 이름/단위로 알려져 있는가? | 개념 매핑 + 환산 쌍 (D-036/D-040). |
| 소스가 제품을 아예 언급하지 않는가? | 그렇다면(SCADA 태그, 알람, 미터): EQUIPMENT 엔티티에 `observation`으로 떨어뜨립니다. 제품 귀속은 나중에 시간창으로 (D-042). |
| 어떤 타임스탬프가 있는가, 어떤 타임존과 형식으로? | D-044: `attrs`의 시간은 UTC ISO-8601이어야 합니다. 소스 타임존을 팩 명세에 선언하십시오. |
| 소스가 데이터를 어떻게 재전송하는가(리플레이, 보정, 반복 측정)? | 멱등성 계획 + CORRECTION 정책 (I12, D-034). |

## 저장 레인 결정 트리

모든 파라미터에 대해, 이 순서로:

1. **고속 신호 데이터인가 (SCADA 주기보다 빠름, 대략 1초 미만 주기)?** → **TRACE**. 원시 샘플은 Parquet으로 가고 `file_ref`로 참조됩니다. 절대 행이 되지 않습니다 (I5). 장비가 같은 양의 스칼라 요약값도 보고한다면 → **TRACE+SCALAR** (요약값은 `measurement` 행, 원시는 파일).
2. **고정 설비의 저속 텔레메트리(시간에 따른 챔버, 배관 센서)이고, SCADA 주기 또는 그보다 느리며, SQL에서 필요한가?** → **TIMESERIES**. `observation`의 행, `ts`로 키. v1은 숫자형만.
3. **아이템당 대량 동종 스칼라인가 (STDF PTR/FTR: 수천 개 테스트 × 다수 유닛)?** → **SCALAR\_BULK**. 부모당 Parquet 하나(유닛/다이가 행, 테스트가 컬럼)를 `file_ref` 뒤에 둡니다. `measurement` 행은 0개.
4. **그 밖의 모든 것(수율, 빈, 점수, 상태, 스냅샷)** → **SCALAR**. (아이템, 파라미터)당 `measurement` 행 하나.

레인을 틀렸을 때의 증상: `measurement` 행이 수십억 개(SCALAR\_BULK여야 함); 트레이스 샘플이 `observation`에 있음(케이던스 규칙 위반); TIMESERIES 파라미터가 시간창마다 가짜 엔티티를 만들어냄(I1 위반 — `observation`이 존재하는 이유가 바로 이것을 막기 위함).

## 온보딩 단계 — 이 순서로 행을 쓴다

새 인스턴스를 세울 때는 [01 · Schema Spec](./01-schema-spec-v2.2.md)(영문 정본)의 DDL 3개 블록을 순서대로 실행하고, `schema_migration`이 `v2.2`를 보이는지 확인하십시오.

1. **타입** — `entity_type` 행 INSERT. `domain`(PRODUCT / EQUIPMENT / PROCESS)을 설정합니다. `parent_type`은 하나의 도메인 안에 머무릅니다(I10). `type_code`는 16자 이하(D-027: `TEST_PROG_VER`이며 `TEST_PROGRAM_VERSION`이 아님). 아이템 종류마다 `type_code` 하나(I14).
2. **명명** — 이 `source_code`에 대해 타입별 `entity_key_def` 행 INSERT (추가 전용, I3. 정규화 규칙은 닫힌 집합 `AS_IS` / `UPPER` / `DIGITS_PAD2` / `DIGITS_PAD3`에서만, I13).
3. **딕셔너리** — 모든 측정에 대해 `param_def` 행 INSERT. `grain`과 `storage_class`는 *의도적으로* 고르십시오 — 앞으로의 모든 질의를 결정합니다. 조건별 파라미터: `native_id`에 접미사(`t1001@125C`), `param_condition` 행 추가, 하나의 `concept_id` 공유(D-038).
4. **개념과 단위** — 소스 간 공통 양이 있으면 파라미터를 `param_concept`에 매핑합니다. `unit` ≠ `canonical_unit`이면 `to_canonical_scale`/`_offset` 쌍을 설정합니다(D-040 — 매핑 리뷰에서 필수). 알려진 동의어는 `param_alias` 행으로 추가합니다.
5. **레이아웃** — 파일 형태마다 올바른 `lane`을 지닌 `file_layout_set` 하나. 채널/테스트 컬럼마다 `file_layout` 행 하나. 불규칙/타임스탬프 트레이스: `time_column` 설정(D-041). 균일 트레이스: 참조가 `sampling_hz`를 지님.
6. **스펙** — 한계값/설정값에 대해 `param_spec` 행 INSERT. 지배하는 PROCESS 도메인 엔티티에 키를 둡니다(가드로 강제, D-023). 설정값: `spec_kind='SETPOINT'`, 맥락 = RECIPE\_VERSION(D-031). 개정 = 이전 것 은퇴 + 새것 삽입을 **하나의 트랜잭션**으로(D-034).
7. **적재** — 아래 로더 계약 참조.
8. **등록** — 팩을 [06 · Case Registry](./06-case-registry.md)에 기록합니다. 행으로 담기지 않는 것은 무엇이든 [04 · Known Limits](./04-known-limits-deferred.md)에 기록된 결함으로 갑니다 — **절대 조용히 **`attrs`**로 넣지 마십시오.**

## 로더 계약

모든 로더는 **반드시**:

1. **ingest\_batch를 연다** (`source_code`, `source_uri`, `kind='INGEST'`) 그리고 자신이 쓰는 모든 행에 `batch_id`를 찍는다(`measurement`, `link`, `file_ref`, `param_spec`, `observation`). 커밋 시 배치를 닫는다(`finished_at`).
2. **엔티티를 멱등적으로 resolve-or-create** — `ON CONFLICT (natural_key) DO UPDATE SET attrs = EXCLUDED.attrs`. `natural_key`를 직접 INSERT하지 말 것(I2) — `entity_key_def` 레시피를 통해 리졸버로 만들어야 한다. 들어온 타입에 레시피가 없으면 적재를 실패시킨다(I13).
3. **충돌 정책을 지킨다 (I12/D-034):** 동일한 값의 재전송은 같은 행에 떨어지고 오류 0. **다른** 값의 재전송(`link` 시간창이 다른 경우 포함 — D-042)은 **크게 실패**해야 한다. 의도적으로 덮어쓰려면 `kind='CORRECTION'` 배치로 실행한다 — 먼저 이전 행을 `measurement_audit`에 보관하고, 그 다음에 덮어쓴다. 조용한 덮어쓰기는 절대 없다.
4. **올바른 충돌 대상을 쓴다:** `entity` → `(natural_key)`; `measurement` → `(entity_id, param_id)`; `link` → `uq_link_natural (src, dst, kind, context)`; `file_ref` → `(storage_uri)`; `observation` → `(entity_id, param_id, ts)`.
5. **시간을 검증한다 (D-044):** `attrs`에 쓰이는 모든 타임스탬프는 `Z`가 붙은 UTC ISO-8601. 키는 `_at`으로 끝난다. 소스 로컬 시간은 팩 명세에 선언된 타임존으로 변환한다. 파싱 불가하거나 모호한 시간은 적재를 실패시킨다.
6. **시간창 (D-042):** `window_start`/`window_end`는 소스가 제공한 것에서만 쓴다 — 끝 시각을 절대 합성하지 않는다. 대량 적재: EXCLUDE 방식의 행 단위 검사를 건너뛰고, COMMIT 전에 집합 기반 겹침 검증을 실행한다(아래).
7. **대량 적재 가드 패턴 (D-034):** 적재 트랜잭션 안에서 `ALTER TABLE link DISABLE TRIGGER trg_link_guard`를 해도 되지만, COMMIT 전에 도메인 + 순환 + 겹침 검증을 **집합 기반으로 한 번** 실행해야 한다. 검증하지 않고 커밋하지 말 것.
8. **크게 실패하는 철학:** 이 페이지에서 "실패한다"고 쓴 모든 곳은 *해당 행과 규칙을 명시한 분명한 오류로 적재가 멈춘다*는 뜻이다. 추측, 조용한 건너뛰기, "최선을 다한" 파싱은 모두 버그다.

### 참조 SQL

시간창을 지닌 링크의 겹침 검증입니다. **running-max 형태**를 쓰십시오 — `lag()` 형태는 **틀렸습니다**. 중첩된 겹침을 놓칩니다(2026-08-11 검증). 결과가 COMMIT 전에 0이어야 합니다.

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

보정 정책을 지키는 measurement upsert(로더 측 검사 형태)입니다. 영향 행이 0인데 값이 다르면 — 값 충돌이므로 CORRECTION 배치가 필요하다는 예외를 올리십시오.

```sql
INSERT INTO measurement (entity_id, param_id, value_num, captured_at, batch_id)
VALUES (:eid, :pid, :val, :ts, :batch)
ON CONFLICT (entity_id, param_id) DO UPDATE
SET value_num = EXCLUDED.value_num, batch_id = EXCLUDED.batch_id
WHERE measurement.value_num IS NOT DISTINCT FROM EXCLUDED.value_num;
-- 0 rows affected + value differs => raise: value conflict, CORRECTION batch required
```

## 하지 말아야 할 것 — 실패 카탈로그

| 하지 말 것 | 이유 | 대신 |
| --- | --- | --- |
| 타임스탬프나 설비로 무언가를 키잉하기 | I1 — 웨이퍼 하나가 유령 아이템들로 부서진다 | RUN 레벨. 시간은 `attrs`로 |
| 모델링되지 않은 형태를 `attrs`에 쏟아붓기 | "attrs에 넣자"는 모두 기록되는 커버리지 결함이다 | [04 · Known Limits](./04-known-limits-deferred.md)에 기록하고, 그 다음 모델링 |
| 하나의 `type_code` 안에서 `attrs`로 두 아이템 종류를 구분하기 | I14 — 생성되는 모든 뷰를 오염시킨다 (L-SCK-01) | 두 개의 타입, 두 개의 정직한 명명 레시피 |
| 트레이스를 행으로 펼치기 | I5 — 저장공간과 인덱스 폭발 | TRACE 레인, Parquet + `file_ref` |
| 부모 롤업을 `measurement`로 저장하기 | I6 — 집계값이 표류한다 | 읽을 때 계산, 또는 명확히 파생된 테이블 |
| 같은 설정값을 measurement와 spec 양쪽에 저장하기 | D-031 — 진실이 두 개 | `param_spec` SETPOINT만 |
| `natural_key`를 손으로 쓰기 | I2 — 리졸버만 | `entity_key_def`를 통해 해석 |
| `entity_key_def` 행을 수정하기 | I3 — 다음 적재에서 조용한 중복 | 새 버전 + 명시적 re-key |
| 레지스트리 행을 DELETE하기 | I7 | `is_active = FALSE` |
| 값이 다른 재전송을 조용히 덮어쓰기 | I12 — 추적 불가능한 데이터 변형 | 크게 실패시키고, CORRECTION 배치 |
| 없는 명명 레시피나 타임존을 추측하기 | I13/D-044 | 적재를 실패시킨다 |
| 시간창을 어떤 키에든 넣기 | I1/D-034 거부 | 시간창은 맥락 컬럼일 뿐 |
| 단위가 섞인 값에 소스별 SQL을 쓰기 | D-040이 이것을 위해 존재한다 | 읽을 때 scale/offset으로 환산 |

## 승인 게이트 — 팩이 완료되는 조건

1. 매핑 명세가 리뷰되고 승인되었다 (로더 코드보다 **먼저**).
2. 전체 적재가 통과한다. 재실행이 **동일한 카운트와 오류 0**을 낸다 (멱등성 증명).
3. 고의로 변형시킨 재전송이 크게 실패한다. 그 다음 CORRECTION 배치가 성공하며, 이전 값이 `measurement_audit`에 있다.
4. 모든 `attrs` 타임스탬프가 UTC ISO-8601로 검증된다.
5. 행 카운트가 소스와 대조된다 (파일 수 × 예상 행 수 — 산식을 명시할 것).
6. 팩이 [06 · Case Registry](./06-case-registry.md)에 레인·트리·명명 레시피, 그리고 기록된 한계와 함께 등록되었다.
