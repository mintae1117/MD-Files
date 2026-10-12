# 00a · 테이블·컬럼 레퍼런스 — 모든 테이블, 모든 컬럼 (v2.2)
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/380239921/00a+v2.2
> pageId: 380239921 · lastModified: 8월 15, 2026 · saved: 2026-10-12

**모든 테이블, 모든 컬럼, 예시 포함 — 스키마 v2.2 (18개 테이블).**

> [00 · 시작하기](./00-KO-start-here.md)의 동반 페이지. 이 페이지는 [01 · Schema Spec](./01-schema-spec-v2.2.md)의 **표준 v2.2 DDL**(3개 블록, PostgreSQL 16.13에서 2026-08-11 검증)에서 만든 **읽기 보조 자료**입니다. 컬럼 이름·타입·키·CHECK는 그 DDL에서 왔고, 예시는 가능한 한 SCK·SEMES 팩에서 가져온 설명용입니다. **이 페이지와 01 페이지가 다르면 01 페이지가 우선**이며, 그건 이 페이지의 버그이니 알려주세요. 작성 2026-08-14.

## 컬럼 읽는 법

- **PK / FK** — PK 컬럼은 행의 식별자 일부이고, FK 컬럼은 다른 테이블의 PK를 가리킵니다. 조인은 id로, 이름으로는 절대 하지 않습니다 (I4).
- **CHECK / UNIQUE** — 데이터베이스가 스스로 강제하는 규칙. 걸리면 조용히 틀린 행이 아니라 큰 에러가 납니다. 그게 설계입니다.
- **트리거 강제** — CHECK로 표현하기엔 복잡한 규칙을, 크게 실패하는 가드 함수가 강제합니다 (도메인 규칙, 레인, 순환, 스펙 컨텍스트).
- **Nullable** — 비워둘 수 있음. "NULL이 의미를 갖는" 경우 해당 컬럼 행에 그 뜻을 적어두었습니다.

## 세 개의 도메인

모든 `entity_type` 행은 `domain`을 갖고, 모든 엔티티는 자기 유형에서 그것을 물려받습니다. 컬럼은 `VARCHAR(16) NOT NULL DEFAULT 'PRODUCT'` — 관례로 고정된 세 가지 값입니다.

| 도메인 | 여기 속하는 것 | 예시 유형 |
| --- | --- | --- |
| **PRODUCT** | 본품 — 만들어지고 측정되고 출하되는 것 | `LOT · WAFER · SITE · RUN · DIE · FAB_LOT · TRAY · MODULE · UNIT` |
| **EQUIPMENT** | 만들거나 측정하는 것 | `TOOL · CHAMBER · TESTER · 파이프라인 세그먼트` |
| **PROCESS** | 지시서와 그 버전 | `RECIPE · RECIPE_VERSION · RECIPE_STEP · TEST_PROGRAM · TEST_PROG_VER` |

**도메인은 세 가지를 결정합니다** — 도메인이 존재하는 이유 전부입니다:

1. **트리 포함관계.** `entity_type.parent_type`은 한 도메인 안에 머물러서, 웨이퍼의 부모가 챔버일 수 없습니다 (I10).
2. **어떤 링크가 합법인가.** 모든 `link_type` 규칙이 도메인으로 쓰여 있습니다 — 다음 절 참조.
3. **스펙이 어디 붙을 수 있는가.** `param_spec.context_entity_id`는 PROCESS 도메인 엔티티여야 하며, 트리거가 강제합니다 (D-023).

**새 유형의 도메인 고르기**, 질문 하나로: *만들어지거나 측정되는 것*인가 (PRODUCT), *만들거나 측정하는 것*인가 (EQUIPMENT), 아니면 *따르는 지시서*인가 (PROCESS)?

> **솔직한 참고.** 세 값은 CHECK 제약이 아니라 폐쇄된 관례입니다 — DDL이 컬럼 텍스트를 제한하지 않습니다. 오타 난 도메인은 `entity_type`에 들어가고, 유형 등록 시점이 아니라 그것을 건드리는 첫 링크에서 크게 실패하는 가드 에러로 드러납니다. 유형 등록은 리뷰를 거치는 단계라 실제로 문제가 된 적은 없습니다. 더 단단히 하고 싶다면 조용한 패치가 아니라 결정으로 올릴 사안입니다.

## 링크 종류는 자유롭지 않습니다

`link.link_kind`는 `link_type`을 참조하는 외래 키입니다. 거기 등록되지 않은 종류는 삽입 자체가 불가능합니다 — FK가 거부합니다. 그리고 등록된 각 종류는 자기 규칙을 갖고, 범용 가드 하나가 모든 INSERT·UPDATE에서 그것을 강제합니다:

| link\_kind | src 도메인 | dst 도메인 | context 도메인 | 비순환 | 의미 |
| --- | --- | --- | --- | --- | --- |
| `ASSEMBLY` | PRODUCT | PRODUCT | — | TRUE | 여러 부품 → 조립품 하나 |
| `MERGE` | PRODUCT | PRODUCT | — | TRUE | 여러 소스 랏 → 병합된 랏 하나 |
| `SPLIT` | PRODUCT | PRODUCT | — | TRUE | 하나 → 여럿, 단 출력이 소스 아래 이름을 받지 않을 때만 |
| `PROCESSED_IN` | PRODUCT | EQUIPMENT | PROCESS | FALSE | 제품이 장비에서 처리됨; 컨텍스트 = 실행된 RECIPE\_VERSION |
| `TESTED_ON` | PRODUCT | EQUIPMENT | PROCESS | FALSE | 제품이 장비에서 테스트됨; 컨텍스트 = 실행된 TEST\_PROG\_VER |

**가드가 매번 검사하는 것:**

- 출발 엔티티의 도메인이 `src_domain`과, 도착 엔티티가 `dst_domain`과 같은지 — 아니면 둘 다 적시한 큰 에러;
- 컨텍스트가 주어졌다면, 그 종류가 `context_domain`을 선언했는지(**NULL이면 그 종류는 컨텍스트 금지**), 그리고 컨텍스트 엔티티가 그 도메인인지;
- `acyclic`이 TRUE면, 그 종류의 기존 그래프를 걸어서 순환을 닫는 엣지를 거부합니다 — 조립품이 자기 자신을 포함할 수 없습니다.

**새 관계 종류를 추가하는 것은 **`link_type`에 INSERT 하나입니다 — 새 브리지 테이블이 아닙니다. 01 페이지의 보편성 테스트 중 하나입니다: 새 관계 유형은 스키마가 아니라 *행*이어야 합니다. 규칙이 데이터이므로, 규칙을 바꾸면 이후의 모든 링크가 할 수 있는 일이 바뀝니다.

**한 테이블, 두 계열.** 라우팅(`PROCESSED_IN`, `TESTED_ON`)은 PRODUCT → EQUIPMENT를 가로지르며 그 사건을 지배한 PROCESS 컨텍스트를 함께 나릅니다. 계보(`ASSEMBLY`, `MERGE`, `SPLIT`)는 PRODUCT 안에 머물고, 컨텍스트를 갖지 않으며, 비순환입니다. `SPLIT`은 출력이 소스 아래 이름을 받지 *않을* 때만 씁니다 — 자식이 부모 아래 이름을 받는 보통의 다이싱은 링크가 아니라 트리 포함관계(`parent_id`)입니다.

## 18개 테이블 한눈에

| 상자 | 테이블 | 한 행은… |
| --- | --- | --- |
| ITEM (사물) | `entity_type` | 품목 종류 하나 |
| ITEM (사물) | `entity_key_def` | 한 유형의 명명 규칙 중 '한 조각' (소스별) |
| ITEM (사물) | `entity` | 추적되는 사물 하나 |
| VALUE (값) | `param_def` | 측정 종류 하나 — 사전(Dictionary) |
| VALUE (값) | `param_condition` | 파라미터를 한정하는 조건 축 하나 (D-038) |
| VALUE (값) | `measurement` | 스칼라 값 하나 |
| VALUE (값) | `observation` | 고정 자산의 시간 키 텔레메트리 한 점 (D-035) |
| VALUE (값) | `param_concept` | 소스를 가로지르는 표준 물리량 하나 (D-036) |
| VALUE (값) | `param_alias` | 파라미터로 해석되는 동의어 하나 (D-036) |
| SPEC (스펙) | `param_spec` | 어떤 컨텍스트에서 파라미터 하나의 스펙 밴드 |
| FILE (파일) | `file_layout_set` | 파일 컬럼 레이아웃 하나, 레인 태그 포함 |
| FILE (파일) | `file_layout` | 레이아웃의 컬럼(채널 또는 테스트) 하나 |
| FILE (파일) | `file_ref` | Parquet 파일 하나를 가리키는 포인터 — 행과 벌크를 잇는 다리 |
| LINK (관계) | `link` | 두 사물 사이의 관계 하나 — 라우팅 이벤트 또는 계보 엣지 |
| LINK (관계) | `link_type` | 허용된 관계 종류 하나와 그것을 단속하는 규칙 |
| PROVENANCE (이력) | `ingest_batch` | 적재 또는 수정 배치 하나 — 그 배치가 쓴 모든 행의 이력 |
| PROVENANCE (이력) | `measurement_audit` | CORRECTION 배치가 보관한, 대체된 값 하나 (append-only) |
| PROVENANCE (이력) | `schema_migration` | 적용된 DDL 패키지 하나 — 스키마 자신의 이력 (D-045) |

---

## 테이블별, 컬럼별

### ITEM (사물)

#### `entity_type` — 한 행은 품목 종류 하나

*PK: type\_code*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `type_code` | 종류 코드. 16자 이하 (D-027). 생성 뷰가 이걸로 필터 | 'WAFER' |
| `parent_type` | FK → entity\_type: 트리상 부모 종류. NULL = 루트 | 'LOT' (WAFER의 경우) |
| `domain` | PRODUCT \| EQUIPMENT \| PROCESS. NOT NULL, 기본값 PRODUCT | CHAMBER면 'EQUIPMENT' |
| `description` | 사람이 읽는 설명 | '팹 랏 안의 웨이퍼' |

- `parent_type`는 한 도메인 안에 머물러야 합니다 — I10의 트리 쪽이며, 매핑 설계 시점에 확인합니다 (강제 트리거 없음).
- 하나의 `type_code` = 하나의 품목 종류 (I14). 서로 다른 소스 필드로 키를 만들면 서로 다른 유형입니다.

#### `entity_key_def` — 한 행은 한 유형의 명명 규칙 중 '한 조각' (소스별)

*PK: (type\_code, source\_code, version, seq) · APPEND-ONLY (I3)*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `type_code` | 이 규칙이 명명하는 종류 | 'DIE' |
| `source_code` | 적용 소스. '\*' = 기본값; 구체적 코드가 우선 | 'SCK' |
| `version` | 규칙 버전. 명명 변경 = 새 버전 + 명시적 re-key 마이그레이션 | 1 |
| `seq` | 복합 키 안에서 이 조각의 순서 | 1, 그다음 2 |
| `field_name` | 읽어올 소스 필드 | 'DIE\_X' (퓨즈 PTR t850021) |
| `prefix` | 정규화된 값 앞에 붙는 리터럴 | 'X' |
| `canon_rule` | 폐쇄 규칙 집합 (I13): AS\_IS \| UPPER \| DIGITS\_PAD2 \| DIGITS\_PAD3 | 'DIGITS\_PAD3' → 57이 '057'로 |

- SCK의 DIE는 두 행을 씁니다 — seq 1 ('X', PAD3), seq 2 ('Y', PAD3) — 합쳐서 `local_id` `X057Y028`.
- 들어온 유형의 규칙이 없으면 로드를 실패시킵니다. 절대 추측하지 않습니다 (I13).

#### `entity` — 한 행은 추적되는 사물 하나

*PK: entity\_id · UNIQUE: natural\_key · UNIQUE NULLS NOT DISTINCT (type\_code, parent\_id, local\_id)*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `entity_id` | 의미 없는 BIGINT identity. 조인은 항상 이걸로 (I4) | 40217 |
| `type_code` | FK → entity\_type: 어떤 종류의 사물인가 | 'WAFER' |
| `parent_id` | FK → entity (자기참조): 유일한 포함/명명 부모. NULL = 루트 | 그 LOT의 entity\_id |
| `local_id` | 부모 안에서의 이름. CHECK: '/' 포함 불가 | 'W07' |
| `natural_key` | 파생: parent.natural\_key \|\| '/' \|\| local\_id. 리졸버 전용 (I2) | 'LOT2291/W07' |
| `attrs` | JSONB 서술 정보 — 절대 식별자 아님. 시간은 UTC ISO-8601, 키는 \_at로 끝남 (D-044) | {"started\_at":"2026-08-11T02:14:00Z"} |
| `created_at` | 행이 생성된 시각 (DB 시계) — 사건 발생 시각이 아님 | 2026-08-11 05:00Z |

- (type\_code, parent\_id, local\_id)의 `UNIQUE NULLS NOT DISTINCT` 덕분에, 같은 이름의 루트 두 개는 조용히 중복되지 않고 충돌합니다.
- 자식의 부모가 `entity_type.parent_type`과 맞는지는 DDL이 강제하지 않습니다 — 트리를 올바르게 세우는 건 리졸버의 책임입니다.

### VALUE (값)

#### `param_def` — 한 행은 측정 종류 하나 (사전)

*PK: param\_id · UNIQUE (source, native\_id)*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `param_id` | identity PK — measurement·file\_layout이 가리키는 값 | 102 |
| `source` | 소스 시스템 네임스페이스 | 'SEMES' |
| `native_id` | 소스가 부르는 이름. source와 함께 UNIQUE | 'SVID14', 't1001', 'PRR.HARD\_BIN' |
| `name` | 사람이 읽는 이름 | 'Chamber Pressure' |
| `unit` | 소스에서 받은 그대로의 단위 | 'mTorr' |
| `grain` | FK → entity\_type: 값이 붙는 엔티티 레벨. 가장 먼저 확인 | 'WAFER' / 'SITE' / 'UNIT' |
| `value_type` | FLOAT \| STRING \| BOOL \| CATEGORICAL | 'FLOAT' |
| `storage_class` | 레인: SCALAR \| SCALAR\_BULK \| TRACE \| TRACE+SCALAR \| TIMESERIES | 'TRACE+SCALAR' |
| `is_active` | 여기서 소프트 폐기; 측정값 있는 파라미터는 DELETE 금지 (I7) | TRUE |
| `concept_id` | FK → param\_concept: 소스 간 공통 물리량 (D-036) | 7 = CHAMBER\_PRESSURE |
| `to_canonical_scale / _offset` | 개념의 표준 단위로 가는 선형 변환. 둘 다 또는 둘 다 없음 (CHECK, D-040) | 0.133322 / 0 (mTorr → Pa) |

- 하나의 SVID, 하나의 정의 (I9). 별칭과 개념은 이 행을 *가리킬* 뿐, 의미를 복사하지 않습니다.
- `grain`과 `storage_class`가 앞으로의 모든 쿼리를 결정합니다. 매핑 리뷰에서 신중히 정하세요.

#### `param_condition` — 한 행은 파라미터를 한정하는 조건 축 하나 (D-038)

*PK: (param\_id, condition\_key)*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `param_id` | FK → param\_def: 이 조건이 한정하는 파라미터 | 't1001@125C' 파라미터 |
| `condition_key` | 조건 축 | 'TEMP\_C', 'VDD\_V', 'FREQ\_HZ' |
| `value_num / value_text` | 조건 값 — 정확히 하나만 채움 (CHECK) | 125 |

- 축이 둘이면 같은 `param_id`에 행이 둘 (TEMP\_C=125, VDD\_V=1.8).
- 한 물리량의 코너들은 하나의 `param_concept`을 공유하는 별개의 `param_def` 행입니다 — 가변 조건 하나짜리 파라미터가 아닙니다.

#### `measurement` — 한 행은 스칼라 값 하나

*PK: (entity\_id, param\_id) — 키에 시간 없음 (I11)*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `entity_id` | FK → entity: 파라미터의 grain에서 이 값이 속한 사물 | SITE 'LOT2291/W07/S01' |
| `param_id` | FK → param\_def: 어떤 측정인가 | SVID31 (Film Thickness) |
| `value_num / value_text / value_bool` | 값 — 정확히 한 컬럼만 채움 (CHECK num\_nonnulls = 1) | 512.3 |
| `captured_at` | 소스가 측정한 시각. NULL 허용; 키의 일부 아님 | 2026-08-11T02:15:03Z |
| `batch_id` | FK → ingest\_batch: 어느 로드가 썼는가. 파일까지 조인 한 번 | 884 |

- 같은 값의 재전송은 바로 이 행에 안착합니다. *다른* 값은 배치가 CORRECTION이 아니면 크게 실패합니다 (I12).
- 같은 값을 시간에 따라 또 넣고 싶다면? 고정 자산이면 `observation`, 제품이면 엔티티 레벨(RUN)이 빠진 것입니다.

#### `observation` — 한 행은 고정 자산의 시간 키 텔레메트리 한 점 (D-035)

*PK: (entity\_id, param\_id, ts) · PARTITION BY RANGE (ts), 월별*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `entity_id` | FK → entity: 모니터링 대상 고정 자산 | CHAMBER 'ETCH-03/C03' |
| `param_id` | FK → param\_def, storage\_class가 TIMESERIES | 파이프라인 압력 |
| `ts` | 샘플 시각 — 키에 포함. 이 테이블의 존재 이유 | 2026-08-11T02:15:00Z |
| `value_num` | NOT NULL. v1은 숫자만 — 상태/불리언 채널은 이연 | 4.21 |
| `batch_id` | FK → ingest\_batch: 다른 곳과 동일한 이력 | 902 |

- 주기 경계: SCADA 속도 이하만 여기; 더 빠르면 Parquet의 TRACE로 (I5).
- 파티션 누락 = 크게 실패하는 에러. 파티션 생성이 설계된 조치이지 우회가 아닙니다.

#### `param_concept` — 한 행은 소스를 가로지르는 표준 물리량 하나 (D-036)

*PK: concept\_id · UNIQUE: concept\_code*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `concept_id` | identity PK — param\_def.concept\_id가 가리키는 값 | 7 |
| `concept_code` | 표준 코드, UNIQUE | 'CHAMBER\_PRESSURE' |
| `name` | 사람이 읽는 이름 | '챔버 압력' |
| `canonical_unit` | 매핑된 모든 파라미터가 변환되어 들어갈 단위 | 'Pa' |
| `description` | 자유 텍스트 | '공정 챔버의 절대 압력' |

- 덕분에 쿼리 하나가 SEMES SVID14와 KOGAS PT-201을 단위 일관되게, 소스별 SQL 없이 함께 반환합니다.
- 개념은 값도 설정값도 담지 않습니다 — 물리량의 이름일 뿐입니다 (I9 / D-031).

#### `param_alias` — 한 행은 파라미터로 해석되는 동의어 하나 (D-036)

*PK: (param\_id, alias)*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `param_id` | FK → param\_def: 이 동의어가 가리키는 대상 | 102 |
| `alias` | 사용자가 실제로 입력하거나 소스가 실제로 보내는 이름 | '압력', 'PRESS', 'SVID12' |
| `kind` | SYNONYM \| LOCALE \| LEGACY | '압력'이면 'LOCALE' |

- 별칭은 해석할 뿐, 정의하지 않습니다. 의미는 `param_def`에 남습니다 (I9).
- 필드 테스트 게이트: '압력' / 'PRESS' / 네이티브 id가 모두 같은 `param_id`로 해석되어야 합니다.

### SPEC (스펙)

#### `param_spec` — 한 행은 어떤 컨텍스트에서 파라미터 하나의 스펙 밴드

*PK: (param\_id, context\_entity\_id, spec\_kind, valid\_from) · (param, context, kind)당 ACTIVE 밴드 하나*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `param_id` | FK → param\_def: 어떤 파라미터를 제한하나 | t1001 (Continuity VDD) |
| `context_entity_id` | FK → entity: 지배 노드. 반드시 PROCESS 도메인 — 트리거 강제 (D-023) | TEST\_PROG\_VER 엔티티 |
| `spec_kind` | LIMIT (테스트 리밋) \| SETPOINT (SV, D-031) | 'LIMIT' |
| `lo_num / hi_num` | 밴드 경계. CHECK: 둘 다 있으면 lo ≤ hi | 0.2 / 0.8 V |
| `target_num` | 목표 값 — 설정값의 숫자가 사는 곳 | 45 (mTorr 설정값) |
| `src_unit / src_scale` | 받은 그대로의 단위·스케일, 이력용 | 'V' / 0 |
| `valid_from` | 이 밴드의 발효 시각. PK의 일부라 이력이 남음 (D-034) | 2026-07-01T00:00Z |
| `is_active` | (param, context, kind)당 활성 밴드 정확히 하나 — 부분 UNIQUE 인덱스 | TRUE |
| `captured_at / batch_id` | 소스 시각 / 어느 로드가 썼는가 | 884 |

- 개정은 한 트랜잭션입니다: 옛 행을 `is_active = FALSE`로 UPDATE한 뒤 새 밴드를 INSERT (D-034). 제자리 수정 금지.
- 한 테스트 프로그램에 사이트별 리밋이 다르면 = TEST\_PROG\_VER 컨텍스트 둘, 행 둘. 스키마 변경 없음.

### FILE (파일)

#### `file_layout_set` — 한 행은 파일 컬럼 레이아웃 하나 (레인 태그 포함)

*PK: layout\_id*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `layout_id` | 레이아웃 id (INT 지정), file\_ref·file\_layout이 참조 | 3 |
| `lane` | TRACE \| SCALAR\_BULK (CHECK). 삽입 후 불변 — 트리거로 고정 | 'SCALAR\_BULK' |
| `time_column` | 불규칙 트레이스의 파일 내 타임스탬프 컬럼. NULL = 균일 샘플링 (D-041) | NULL 또는 'ts' |
| `description` | 자유 텍스트 | 'SCK STTV7X 프로그램 컬럼형 레이아웃' |

- 파일 *모양*당 하나이지, 파일당 하나가 아닙니다. SCK는 TEST\_PROG\_VER당 레이아웃 하나를 쓰고, 컬럼은 그 프로그램 데이터로그들의 합집합으로 누적됩니다.
- `time_column`이 NULL이면 타이밍은 `file_ref.sampling_hz`에서 옵니다.

#### `file_layout` — 한 행은 레이아웃의 컬럼(채널·테스트) 하나

*PK: (layout\_id, param\_id) · UNIQUE (layout\_id, column\_name)*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `layout_id` | FK → file\_layout\_set: 어느 레이아웃의 컬럼인가 | 3 |
| `param_id` | FK → param\_def: 이 물리 컬럼이 담는 파라미터 | t1001의 param\_id |
| `column_name` | Parquet 파일 안의 실제 컬럼 이름 | 't1001' |

- UNIQUE (layout\_id, column\_name): 두 파라미터가 한 물리 컬럼을 공유할 수 없습니다.
- 이 테이블은 이름도 단위도 저장하지 않습니다 — 그 의미는 오직 `param_def`에 (I9). 파일은 단순하고, 사전이 똑똑합니다.

#### `file_ref` — 한 행은 Parquet 파일 하나를 가리키는 포인터

*PK: storage\_uri · CHECK file\_ref\_lane\_shape (레인 전용 컬럼이 레인과 맞아야 함)*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `storage_uri` | 파일 위치. 이것이 PK — 파일당 한 행 | 'file://…/sck/\<LOT\>/\<RUN\>.parquet' |
| `lane` | TRACE \| SCALAR\_BULK. 레이아웃의 레인과 일치해야 함 — 트리거 강제 | 'SCALAR\_BULK' |
| `anchor_entity_id` | 파일이 매달린 대상: TRACE = 대상 엔티티; SCALAR\_BULK = 부모 | 그 RUN 엔티티 |
| `layout_id` | FK → file\_layout\_set: 컬럼을 읽는 방법 | 3 |
| `row_grain / row_key_column / row_count` | SCALAR\_BULK 전용: 한 행이 무엇인지, id 컬럼, 행 수 | 'UNIT' / 'entity\_id' / 8141 |
| `sampling_hz / sample_count` | TRACE 전용: 샘플링 레이트와 샘플 수 | 10 / 18000 |
| `window_start / window_end` | 파일이 커버하는 시간 범위. CHECK: 순서 | 02:14Z → 02:44Z |
| `tier` | HOT \| COLD \| PURGED (CHECK). HOT이 아니면 리더가 크게 실패 (D-043) | 'HOT' |
| `tier_changed_at` | 티어가 마지막으로 바뀐 시각 | 2026-08-01T00:00Z |
| `captured_at / batch_id` | 소스 시각 / 어느 로드가 썼는가 | 884 |

- 읽기 전에 `tier`를 확인하세요. PURGED 파일도 행은 묘비로 영구 보존됩니다 — 바이트는 사라져도 이력은 남습니다 (D-043).
- 레인 모양 CHECK 때문에 SCALAR\_BULK 행에 `sampling_hz`를, TRACE 행에 `row_grain`을 넣는 것이 물리적으로 불가능합니다.

### LINK (관계)

#### `link` — 한 행은 두 사물 사이의 관계 하나

*PK: link\_id · uq\_link\_natural UNIQUE NULLS NOT DISTINCT (src, dst, link\_kind, context) ← 로더의 ON CONFLICT 타깃*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `link_id` | 대리 identity PK (D-034에서 추가) | 55021 |
| `src_entity_id` | FK → entity: '출발' 사물. 도메인이 link\_type.src\_domain과 일치해야 함 | 그 DIE / 그 WAFER |
| `dst_entity_id` | FK → entity: '도착' 사물. CHECK: src ≠ dst | 그 UNIT / 그 CHAMBER |
| `link_kind` | FK → link\_type: 반드시 등록된 종류. 미등록이면 INSERT 실패 | 'ASSEMBLY', 'PROCESSED\_IN' |
| `context_entity_id` | FK → entity: 지배하는 PROCESS 노드. 종류가 context\_domain을 선언한 경우에만 허용 | 'ETCH-A/V03' |
| `attrs` | JSONB: 수량, 슬롯 등. 절대 식별자 아님 (I1) | {"slot": 7} |
| `is_active` | 잘못된 엣지는 소프트 폐기; DELETE 금지 (I7). 쿼리에서 반드시 필터 | TRUE |
| `window_start / window_end` | 점유 시간 창 — 컨텍스트일 뿐 식별자 아님 (D-042). CHECK: 순서 | 02:14Z → 02:44Z |
| `ts_window` | STORED 생성 컬럼 tstzrange(start, end, '\[)'). (dst, kind)와 함께 GiST 인덱스 | \['02:14Z','02:44Z') |
| `captured_at / batch_id` | 소스 시각 / 어느 로드가 썼는가 | 884 |

- 시간 창은 유니크 키에 없습니다 — 그래서 다른 창으로 재전송하면 값 충돌로 크게 실패하고 CORRECTION 배치가 필요합니다 (D-042).
- 범용 트리거 하나가 모든 INSERT·UPDATE에서 src/dst/context 도메인과 종류별 비순환성을 검증합니다.

#### `link_type` — 한 행은 허용된 관계 종류 하나와 그것을 단속하는 규칙

*PK: link\_kind — 링크 종류가 자유롭지 않은 이유가 이 테이블*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `link_kind` | 종류 코드, 16자 이하. `link.link_kind`가 이걸 FK로 참조 | 'PROCESSED\_IN' |
| `src_domain` | 출발 엔티티가 반드시 속해야 할 도메인 | 'PRODUCT' |
| `dst_domain` | 도착 엔티티가 반드시 속해야 할 도메인 | 'EQUIPMENT' |
| `context_domain` | 컨텍스트가 속해야 할 도메인. NULL = 이 종류는 컨텍스트 금지 | 'PROCESS' (ASSEMBLY는 NULL) |
| `acyclic` | TRUE = 가드가 그래프를 걸어 순환을 만드는 엣지를 거부 | ASSEMBLY는 TRUE |
| `description` | 이 종류의 의미 | '제품이 장비에서 처리됨; 컨텍스트 = RECIPE\_VERSION' |

- 새 관계 종류는 여기 **행 하나**입니다 — 새 브리지 테이블이 아닙니다. 01 페이지의 보편성 테스트 중 하나입니다.
- 규칙이 **데이터**입니다: 가드가 이 행을 읽어 강제합니다. 규칙을 바꾸면 이후의 모든 링크가 새 규칙을 따릅니다.

### PROVENANCE (이력)

#### `ingest_batch` — 한 행은 적재 또는 수정 배치 하나

*PK: batch\_id · measurement·link·file\_ref·param\_spec·observation에 스탬프*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `batch_id` | identity PK. 로더가 쓰는 모든 행이 이 값을 지님 | 884 |
| `source_code` | 이 배치가 온 소스 시스템 | 'SCK' |
| `source_uri` | 원본 파일·피드 — '이 숫자 어느 파일에서 왔지'에 답하는 값 | 'file://…/STTV7X\_1\_20260711.stdf' |
| `kind` | INGEST (일반) \| CORRECTION (값을 덮어쓰는 유일한 합법 경로) | 'INGEST' |
| `started_at / finished_at` | 로드 시작 시 열림; finished\_at은 커밋 때 기록 | 05:00Z / 05:42Z |
| `note` | 운영자용 자유 텍스트 | '코너 로드, 2차 시도' |

- 다른 걸 쓰기 전에 배치를 **먼저** 여세요. 이력은 나중 일이 아니라 로더 계약의 1단계입니다.
- 끝나지 않은 배치(finished\_at IS NULL)로 중간에 죽은 로드를 찾아냅니다.

#### `measurement_audit` — 한 행은 CORRECTION 배치가 보관한, 대체된 값 하나 (append-only)

*PK 없음 — append-only 아카이브 로그*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `entity_id / param_id` | 어떤 측정이 덮어써졌나 | (SITE id, SVID31) |
| `value_num / value_text / value_bool` | *옛* 값 — 수정 전에 행이 담고 있던 값 | 512.3 |
| `captured_at` | 옛 행의 측정 시각 | 2026-08-11T02:15:03Z |
| `batch_id` | 옛 값을 처음 쓴 배치 | 884 |
| `superseded_at` | 보관된 시각. 기본값 now() | 2026-08-13T09:11Z |
| `superseded_by_batch` | FK → ingest\_batch: 덮어쓴 CORRECTION 배치 | 991 |

- 아카이브 행은 덮어쓰기 **전에** 기록됩니다. 이 테이블이 비어 있다면 조용히 바뀐 값이 하나도 없다는 뜻입니다.
- 필드 테스트 게이트가 읽는 테이블입니다: 값을 변조해 로드 실패를 확인하고, CORRECTION을 돌린 뒤, 여기서 옛 값을 찾습니다.

#### `schema_migration` — 한 행은 적용된 DDL 패키지 하나 (D-045)

*PK: version · 인스턴스 버전 = applied\_at 기준 최신 행*

| 컬럼 | 무엇인가 | 예시 |
| --- | --- | --- |
| `version` | 이 패키지가 인스턴스를 올려놓은 스키마 버전 | 'v2.2' |
| `applied_at` | 적용 시각. **최신** 행이 인스턴스의 버전 | 2026-08-11T00:00Z |
| `script_sha256` | 실제로 실행된 스크립트의 해시 — 어떤 스크립트였는지 모호함 없음 | '9f2c…' |
| `decision_ids` | 이 패키지가 구현하는 비준 결정들 | 'D-038, D-040, D-041, D-042, D-043, D-045' |
| `note` | 패키지가 한 일 설명 | 'close-out 패키지' |

- **이 테이블이 없다는 것**은 인스턴스가 v2.2 이전이라는 뜻입니다. SQL 쓰기 전 확인 1번입니다.
- 행은 오직 번호 붙은 마이그레이션 스크립트로만 생깁니다. 손으로 친 ALTER는 여기 행을 남기지 않습니다 — 그래서 금지입니다.

---

## 이 페이지가 틀렸을 때

읽기 보조 자료라서 어긋날 수 있습니다. 진실은 01 페이지의 라이브 DDL입니다. 불일치를 발견하면 알려주세요 — 여기서는 문구 수정으로 고치고, 틀린 쪽이 *스펙*이라면 그건 [05 · Decision Log](./05-decision-log.md)의 결정 사안이지 조용한 패치가 아닙니다.
