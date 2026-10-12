# 00 · 시작하기 — 신입 엔지니어 온보딩 매뉴얼 (한국어)
> Source: <CONFLUENCE_HOST>/wiki/spaces/COREDB/pages/379617347/00
> pageId: 379617347 · lastModified: 8월 15, 2026 · saved: 2026-10-12

**아무것도 모르는 상태에서, 첫 안전한 쿼리와 첫 데이터 소스 온보딩까지.**

> **먼저 읽어주세요.** 이 매뉴얼은 *학습 보조 자료*이며, 진실의 원본이 아닙니다. 진실은 COREDB 페이지 [01 · Schema Spec](./01-schema-spec-v2.2.md) · [02 · Runbook](./02-data-engineer-runbook.md) · [03 · Query Conventions](./03-backend-query-conventions.md) · [04 · Known Limits](./04-known-limits-deferred.md) · [05 · Decision Log](./05-decision-log.md) · [06 · Case Registry](./06-case-registry.md) · [07 · Field-Test Plan](./07-field-test-plan.md) · [08 · Operations](./08-operations.md) 에 있습니다. 이 매뉴얼과 라이브 페이지가 다르면 **항상 라이브 페이지가 우선**입니다. 2026-08-14 기준 라이브 페이지(v2.2)와 대조 확인. 18개 테이블의 컬럼 단위 상세는 하위 페이지 [00a · 테이블·컬럼 레퍼런스](./00a-KO-table-column-reference.md)에 있습니다. **실제 팩** 예제는 SCK·SEMES 팩(06 페이지 + frozen Notion)에서 왔고, 두 팩은 v2.1 이전이라 구조는 v2.2 이름으로 번역했으며 로드 카운트는 v1.1 `sck` 인스턴스의 as-built 수치로 필드 테스트 Test 1에서 재검증됩니다. English version: parent page.

## 0. 누구를 위한 문서인가, 무엇을 할 수 있게 되는가

당신은 방금 입사한 데이터 엔지니어입니다. SQL과 파이썬은 알지만 이 데이터베이스는 처음 봅니다. 이 매뉴얼을 마치면 다음을 할 수 있게 됩니다.

1. Universal DB가 왜 이런 모양인지 1분 안에 설명하기.
2. DB 안(또는 옆의 Parquet 파일)에서 어떤 값이든 추측 없이 찾아내기.
3. 하우스 룰을 하나도 깨지 않고 첫 쿼리 작성하기.
4. 새 데이터 파일이 도착했을 때 런북(runbook) 절차 따라가기.
5. 데이터가 스키마에 *안 맞을 때* 무엇을 해야 하는지 알기 — 여기서 가장 중요한 기술입니다.

예상 소요: 반나절 동안 읽고, 첫 2주는 12장의 실습 계획을 따라가세요.

## 1. 이 데이터베이스가 푸는 문제

AIBIZ는 전혀 다른 세계의 제조 데이터를 다룹니다. 반도체 에처(SEMES), STDF 파일 기반 패키지 후공정 테스트(SCK), 카메라 모듈 테스터(LG Innotek), 가스 파이프라인(KOGAS) — 그리고 다음에 올 무언가.

전통적 방식은 고객마다 스키마 하나씩입니다. 여기는 `wafer` 테이블, 저기는 `test_result` 테이블, 또 어디는 `pipeline_sensor` 테이블. 프로젝트 하나에는 잘 맞습니다. 그런데 다음 고객이 오면 전부 포크(fork)하게 됩니다: 새 테이블, 새 로더, 새 쿼리, 새 버그. 고객 다섯이면 미묘하게 다른 DB 다섯 개를 유지보수하게 되고, 재사용 가능한 건 아무것도 없습니다.

Universal DB는 정반대에 베팅합니다: **고객이 바뀌어도 절대 변하지 않는 하나의 스키마**. 새 센서, 새 품목 유형, 새 명명 규칙, 새 관계 종류 — 전부 **레지스트리 테이블의 행(row)** 으로 들어오지, 새 컬럼이나 새 테이블로 들어오지 않습니다. 새 도메인 온보딩에 행 입력만 필요하면 스키마가 제 역할을 하는 것입니다. 새 케이스가 스키마 변경을 *강제*하면 그것은 **커버리지 결함(coverage defect)** 이며, 정식 의사결정을 통해 기록되고 수정됩니다 — 조용히 패치되는 일은 없습니다.

한 가지 분명히 할 것(결정 D-026): 스키마는 *논리적으로* 보편이지만 *물리적으로는* 프로젝트별입니다. 고객마다 자기 인스턴스(`sck`, `semes`, `innotek`, `kogas`, …)를 운영하되, 모든 인스턴스는 **포크되지 않은 동일한 스키마**를 돌립니다.

## 2. 핵심 아이디어 — 한 문단

공장 사물의 이름을 딴 테이블(`wafer`, `lot`, `chamber`)을 만드는 대신, 이 DB는 공장 사물을 소수의 범용 테이블에 **행으로** 저장합니다. *어떤 종류의 사물이 존재하는가*도 데이터입니다(`entity_type`). *각 사물이 어떻게 명명되는가*도 데이터입니다(`entity_key_def`). *무엇이 측정되는가*도 데이터입니다(`param_def`). *사물이 어떻게 연결되는가*도 데이터입니다(`link_type`). 스키마 자체는 여섯 개념만 압니다: **품목(item), 값(value), 스펙(spec), 파일(file), 링크(link), 이력(provenance)**. 그래서 18개 테이블이 에처도, 테스트 라인도, 가스 파이프라인도 모양 변경 없이 흡수할 수 있습니다.

## 3. 여섯 상자, 열여덟 테이블

테이블 18개를 외우려 하지 마세요. 상자 여섯 개를 외우면 테이블은 저절로 정리됩니다.

| 상자 | 답하는 질문 | 테이블 |
| --- | --- | --- |
| **ITEM** | 어떤 사물이 존재하고, 어떻게 명명되는가? | `entity_type`, `entity_key_def`, `entity` |
| **VALUE** | 무엇이 측정되었고, 값은 무엇인가? | `param_def`, `param_condition`, `measurement`, `observation`, `param_concept`, `param_alias` |
| **SPEC** | 값이 *어때야 하는가* (리밋, 설정값)? | `param_spec` |
| **FILE** | 대용량 원시 파일은 어디 있고 어떻게 읽는가? | `file_layout_set`, `file_layout`, `file_ref` |
| **LINK** | 트리를 가로질러 사물이 어떻게 연결되는가 (라우팅, 계보)? | `link`, `link_type` |
| **PROVENANCE** | 모든 행은 어디서 왔는가? 디스크의 스키마 버전은? | `ingest_batch`, `measurement_audit`, `schema_migration` |

일상 업무의 대부분은 세 테이블입니다: `entity`(사물), `param_def`(측정 사전), `measurement`(값). 여기서 시작하세요.

### 18개 테이블 전부 — 한 행이 무엇인가

방향을 잡기 위해 한 번만 읽으세요. 외울 필요 없습니다. **모든 테이블의 모든 컬럼과 예시는 동반 페이지 **[**00a · 테이블·컬럼 레퍼런스**](./00a-KO-table-column-reference.md)**에 있습니다** — SQL 쓰면서 찾아볼 곳이 거기입니다.

| 상자 | 테이블 | 한 행은… | 기본 키 |
| --- | --- | --- | --- |
| ITEM | `entity_type` | 품목 종류 하나 | `type_code` |
| ITEM | `entity_key_def` | 한 유형의 명명 규칙 중 '한 조각' (소스별) | (`type_code`, `source_code`, `version`, `seq`) |
| ITEM | `entity` | 추적되는 사물 하나 | `entity_id` |
| VALUE | `param_def` | 측정 종류 하나 — 사전 | `param_id` |
| VALUE | `param_condition` | 파라미터를 한정하는 조건 축 하나 | (`param_id`, `condition_key`) |
| VALUE | `measurement` | 스칼라 값 하나 | (`entity_id`, `param_id`) |
| VALUE | `observation` | 고정 자산의 시간 키 텔레메트리 한 점 | (`entity_id`, `param_id`, `ts`) |
| VALUE | `param_concept` | 소스를 가로지르는 표준 물리량 하나 | `concept_id` |
| VALUE | `param_alias` | 파라미터로 해석되는 동의어 하나 | (`param_id`, `alias`) |
| SPEC | `param_spec` | 어떤 컨텍스트에서 파라미터의 스펙 밴드 하나 | (`param_id`, `context_entity_id`, `spec_kind`, `valid_from`) |
| FILE | `file_layout_set` | 파일 컬럼 레이아웃 하나 (레인 태그 포함) | `layout_id` |
| FILE | `file_layout` | 레이아웃의 컬럼(채널·테스트) 하나 | (`layout_id`, `param_id`) |
| FILE | `file_ref` | Parquet 파일 하나를 가리키는 포인터 | `storage_uri` |
| LINK | `link` | 두 사물 사이의 관계 하나 | `link_id` |
| LINK | `link_type` | **허용된** 관계 종류 하나 + 그 규칙 | `link_kind` |
| PROVENANCE | `ingest_batch` | 적재 또는 수정 배치 하나 | `batch_id` |
| PROVENANCE | `measurement_audit` | CORRECTION이 보관한 대체된 값 하나 | *(없음 — append-only 로그)* |
| PROVENANCE | `schema_migration` | 적용된 DDL 패키지 하나 | `version` |

여기서 벌써 두 가지가 보입니다. *무엇이 존재할 수 있는가*를 기술하는 테이블 — `entity_type`, `entity_key_def`, `param_def`, `link_type`, `file_layout_set` — 은 전부 **레지스트리**입니다: 새 소스 온보딩은 여기에 행을 쓰는 것이 전부입니다. 그리고 다섯 테이블이 `batch_id`를 지닙니다 (`measurement`, `link`, `file_ref`, `param_spec`, `observation`) — DB 안의 어떤 숫자든 출처 파일까지 되짚을 수 있는 이유입니다.

## 4. 멘탈 모델: 트리, 도메인, 링크

### 4.1 모든 것은 트리 속의 엔티티다

추적되는 모든 *사물* — 랏(lot), 웨이퍼, 테스트 유닛, 장비(tool), 챔버, 레시피 — 은 `entity`의 한 행입니다. 엔티티는 `parent_id`로 트리를 이룹니다: 웨이퍼의 부모는 랏, 챔버의 부모는 장비.

각 엔티티가 갖는 것:

- `entity_id` — 의미 없는 번호. **조인은 항상 이걸로 합니다. 예외 없음.**
- `type_code` — 어떤 종류의 사물인가 (`LOT`, `WAFER`, `CHAMBER`, …). 유효한 유형 목록 자체가 `entity_type`의 데이터입니다.
- `local_id` — 부모 *안에서의* 이름 ("W07").
- `natural_key` — 사람이 읽는 전체 경로 ("LOT2291/W07"). **시스템이 파생 생성 — 절대 손으로 쓰지 않습니다.** 해석(resolve)·표시·접두 필터(`LIKE 'LOT2291/%'`)에만 씁니다. 조인에는 절대 쓰지 않습니다.
- `attrs` — 서술적 정보(시작 시각, 작업자, 장비명)를 담는 JSON 가방. **attrs는 절대 식별자가 아닙니다.** 사람들이 가장 먼저 깨는 규칙이라 뒤에서 다시 다룹니다.

> **실제 팩 (SCK).** 다이의 식별자는 키 파트 두 개로 만들어집니다 — 퓨즈에서 읽은 X·Y 좌표, 각각 정규화 규칙 `DIGITS_PAD3` — 그래서 `local_id`는 `X057Y028`, 전체 경로는 `STTV7X/W11/X057Y028`. 위치 기반이고, 안정적이며, 시간도 장비도 없습니다. 부품의 전자 시리얼 번호는 *테스트 시점에* 구워지고 패스마다 달라집니다 — 바로 그래서 식별자가 될 수 없고, 패스들을 잇는 것은 다이 좌표뿐입니다. 밥값 하는 명명 디테일 하나: SCK의 RUN 코드는 `AS_IS`로 그대로 보존됩니다 (`TEST_COD#<sessionTS>-<crc(filename)>`) — 한 랏에 같은 타임스탬프·같은 패스의 원본 데이터로그와 `_rep` 복제본이 공존할 수 있어서, 키에 파일명 CRC가 없으면 하나가 다른 하나를 조용히 덮어씁니다.

### 4.2 세 개의 분리된 트리 (도메인)

엔티티는 세 **도메인** 중 정확히 하나에 삽니다. 도메인은 *유형*마다 한 번 `entity_type.domain`에 정해지고 (`VARCHAR(16) NOT NULL DEFAULT 'PRODUCT'`), 모든 엔티티가 자기 유형에서 물려받습니다:

| 도메인 | 여기 속하는 것 | 예시 유형 |
| --- | --- | --- |
| **PRODUCT** | 만들어지고, 측정되고, 출하되는 것 | LOT · WAFER · SITE · RUN · DIE · FAB\_LOT · TRAY · MODULE · UNIT |
| **EQUIPMENT** | 만들거나 측정하는 것 | TOOL · CHAMBER · TESTER · 파이프라인 세그먼트 |
| **PROCESS** | 지시서와 그 버전 | RECIPE · RECIPE\_VERSION · RECIPE\_STEP · TEST\_PROGRAM · TEST\_PROG\_VER |

도메인은 장식이 아닙니다 — 정확히 세 가지를 결정하고, 그게 존재 이유입니다:

1. **트리 포함관계.** `entity_type.parent_type`이 한 도메인 안에 머물러서, `parent_id`는 **절대 도메인 경계를 넘지 않습니다**. 웨이퍼의 부모가 챔버일 수 없습니다. 이것이 불변식 I10이며, 그 트리 쪽은 매핑 설계 시점에 확인합니다 (강제 트리거는 없습니다 — 트리를 세우는 건 리졸버라서, 잘못된 트리는 거부된 INSERT가 아니라 로더 버그입니다).
2. **어떤 관계가 합법인가.** `link_type`의 모든 규칙이 도메인으로 쓰여 있습니다 — 다음 절. *교차*는 크게 실패하는 트리거가 강제합니다.
3. **스펙이 어디 붙는가.** `param_spec.context_entity_id`는 PROCESS 도메인 엔티티여야 하고, 아니면 트리거가 거부합니다 (D-023). 리밋이 테스트 프로그램 버전에 붙고 제품에는 절대 붙지 않는 이유입니다.

**새 유형의 도메인 고르기**, 질문 하나로: *만들어지거나 측정되는 것*인가 (PRODUCT), *만들거나 측정하는 것*인가 (EQUIPMENT), 아니면 *따르는 지시서*인가 (PROCESS)?

실제 사례 하나: **RECIPE\_STEP**은 PROCESS(지시서의 일부), **CHAMBER**는 EQUIPMENT(실행하는 쪽), 둘이 만나는 **RUN**은 PRODUCT(웨이퍼의 처리 패스). 도메인 셋, 사건 하나, 링크로 연결.

> **실제 팩.** SEMES의 트리는 `LOT → WAFER → {SITE · RUN}` — SITE와 RUN은 WAFER 아래의 *형제*입니다 (측정 사이트와 처리 패스는 서로 다른 종류의 자식). SCK는 소스 하나가 **두 개의** PRODUCT 트리를 나릅니다: 파이널 테스트 `LOT → RUN → UNIT`, 그리고 같은 STDF 파일 안의 QFPROM 퓨즈 리드백에서 재구성한 다이 출처 `FAB_LOT → WAFER → DIE`. 트리 둘, 소스 하나, 스키마 변경 0.

### 4.3 링크가 트리를 가로지른다

그러면 "웨이퍼 W07이 챔버 C03에서 레시피 버전 V02로 식각되었다"는 어떻게 표현할까요? `link` 한 행으로:

- `src_entity_id` = 웨이퍼 (PRODUCT)
- `dst_entity_id` = 챔버 (EQUIPMENT)
- `link_kind` = `PROCESSED_IN`
- `context_entity_id` = 레시피 버전 (PROCESS)

한 행이 *어떤 제품이, 어떤 장비에서, 어떤 지시서로* 를 동시에 답합니다.

**링크 종류는 자유인가? 아닙니다.** `link.link_kind`는 `link_type`을 참조하는 외래 키입니다 — 거기 등록되지 않은 종류는 아예 삽입할 수 없습니다. 그리고 등록된 각 종류는 자기 규칙을 갖고, 범용 가드 하나가 모든 INSERT·UPDATE에서 강제합니다. 베이스라인 DDL에 등록된 다섯 가지:

| link\_kind | src 도메인 | dst 도메인 | context 도메인 | 비순환 | 의미 |
| --- | --- | --- | --- | --- | --- |
| `ASSEMBLY` | PRODUCT | PRODUCT | — | TRUE | 여러 부품 → 조립품 하나 |
| `MERGE` | PRODUCT | PRODUCT | — | TRUE | 여러 소스 랏 → 병합된 랏 하나 |
| `SPLIT` | PRODUCT | PRODUCT | — | TRUE | 하나 → 여럿, **단** 출력이 소스 아래 이름을 받지 않을 때만 |
| `PROCESSED_IN` | PRODUCT | EQUIPMENT | PROCESS | FALSE | 제품이 장비에서 처리됨; 컨텍스트 = 실행된 RECIPE\_VERSION |
| `TESTED_ON` | PRODUCT | EQUIPMENT | PROCESS | FALSE | 제품이 장비에서 테스트됨; 컨텍스트 = 실행된 TEST\_PROG\_VER |

가드가 매번 검사하는 것: 출발 엔티티의 도메인이 `src_domain`과, 도착 엔티티가 `dst_domain`과 같은지; 컨텍스트가 주어졌다면 그 종류가 `context_domain`을 선언했는지(**NULL이면 그 종류는 컨텍스트 금지**)와 컨텍스트 엔티티가 그 도메인인지; `acyclic`이 TRUE면 그 종류의 기존 그래프를 걸어 순환을 닫는 엣지를 거부 — 그래서 조립품이 자기 자신을 포함하는 일이 생길 수 없습니다.

그러니 "자유인가 타입이 있나?"의 답은: **타입이 있지만, 그 타입 체계가 행입니다.** 새 관계 종류를 추가하는 것은 `link_type`에 INSERT 하나이지 새 브리지 테이블이 아닙니다 — 01 페이지의 보편성 테스트 중 하나입니다. 규칙이 데이터이고, 가드가 런타임에 그걸 읽습니다.

표는 두 계열로 읽으세요. *라우팅*(PROCESSED\_IN, TESTED\_ON)은 PRODUCT → EQUIPMENT를 가로지르며 그 사건을 지배한 PROCESS 컨텍스트를 함께 나릅니다. *계보*(ASSEMBLY, MERGE, SPLIT)는 PRODUCT 안에 머물고, 컨텍스트가 없으며, 비순환입니다. 기억해 둘 함정 하나: `SPLIT`은 출력이 소스 아래 이름을 받지 *않을* 때만 씁니다 — 자식이 부모 아래 이름을 받는 보통의 다이싱은 링크가 아니라 트리 포함관계(`parent_id`)입니다.

`link`는 도메인을 가로질러 사물을 연결하는 **유일한** 합법적 방법이며, 모든 관계의 유일한 진실 원천입니다(결정 D-033). 다중 홉 질문("이 모듈의 전체 조상")은 재귀 쿼리로 답하거나, `link`에서 재생성되는 *파생* 그래프 프로젝션으로 답합니다 — 파생물은 언제나 폐기 가능합니다.

> **실제 팩.** *챔버가 식별자가 될 수 없는 이유:* SEMES에서는 FOUP 하나의 웨이퍼 25장이 에처의 챔버 6개로 흩어집니다 — 같은 레시피 버전 아래에서 W07은 C03, W08은 C01에서 돌 수 있습니다. 챔버는 웨이퍼 단위 PROCESSED\_IN 링크이지, 절대 이름의 일부가 아닙니다. *대규모 계보:* SCK의 다이-유닛 계보는 ASSEMBLY 링크 210,890개; 유닛의 99.67%가 연결되었고, 연결 안 된 나머지(퓨즈가 아직 안 구워진 다이)는 **숨기지 않고 집계했습니다** — 숫자와 함께하는 fail-soft, 조용한 누락은 없음.

### 4.4 사전: param\_def

모든 측정 종류는 `param_def`의 한 행 — 사전(Dictionary)입니다. 파라미터가 갖는 것:

- `source` + `native_id` — 출처와 그 출처가 부르는 이름 (`'SEMES'` + `'SVID12'`).
- `grain` — 값이 붙는 엔티티 레벨 (WAFER 단위, LOT 단위, CHAMBER 단위…). 잘못된 grain으로 쿼리하면 에러가 아니라 *빈 결과*가 나옵니다. grain부터 확인하세요.
- `value_type` — FLOAT, STRING, BOOL, CATEGORICAL.
- `storage_class` — **레인(lane)**: 값이 물리적으로 사는 곳. 스키마에서 가장 중요한 컬럼입니다. 다음 장에서 다룹니다.

하나의 SVID, 하나의 정의(불변식 I9): 파라미터의 의미는 *오직* 여기 삽니다. 동의어(`param_alias`)와 소스 간 개념(`param_concept`)은 정의를 *가리킬* 뿐, 복제하지 않습니다.

> **실제 사전 행 (SEMES 팩).** `SVID12` RF Power (WAFER, TRACE+SCALAR) · `SVID14` Chamber Pressure (WAFER, TRACE+SCALAR) · `SVID31` Film Thickness (SITE, SCALAR) · `SVID88` Endpoint Signal (WAFER, TRACE) · `LOT-YLD` Yield (LOT, SCALAR). 파라미터 다섯에 grain/레인 조합이 넷 — 쿼리 전에 둘 다 확인해야 하는 이유입니다. SCK 팩에서 잡힌 I9 함정도 하나: PTR 테스트 `t1680001`(FLOAT)과 FTR 테스트 `f1680001`(BOOL)은 번호를 공유하는 **서로 다른 두 테스트**입니다 — 그래서 `native_id`에 `t`/`f` 네임스페이스 접두가 붙습니다. 네임스페이스를 합치면 Parquet 컬럼이 조용히 오염됩니다.

## 5. 값은 어디에 사는가: 다섯 레인

주니어 엔지니어의 가장 흔한 버그: `SELECT * FROM measurement WHERE param_id = …` 가 0행을 반환하면 "데이터가 없다"고 결론 내리는 것. 대부분 데이터는 멀쩡합니다 — 다른 **레인**에 살고 있을 뿐입니다. `param_def.storage_class`가 위치를 알려줍니다:

| 레인 | 용도 | 값이 있는 곳 |
| --- | --- | --- |
| **SCALAR** | (품목, 파라미터)당 값 하나: 수율, 빈(bin), 점수, 상태 | `measurement` 행 |
| **SCALAR\_BULK** | 초대량 동질 스칼라: 수천 테스트 × 20만 유닛 (STDF) | 부모당 Parquet 파일 하나(유닛당 행), `file_ref`가 가리킴. measurement 행은 **의도적으로 0개** |
| **TRACE** | 고속 신호 (주기 약 1초 미만): 10 Hz 챔버 트레이스 | 대상 엔티티당 Parquet 파일, `file_ref`가 가리킴. 원시 샘플은 **절대** 행이 되지 않음 |
| **TRACE+SCALAR** | 장비가 원시 트레이스 *와* 그 스칼라 요약을 함께 보고 | 요약은 `measurement`, 원시는 파일 |
| **TIMESERIES** | 고정 자산의 저속 텔레메트리 (SCADA 속도 이하): 파이프라인 센서, 수개월의 챔버 상태 | `observation` 행, (entity, param, **ts**) 키 |

온보딩 시 레인 선택 — 이 순서로 결정 트리를 타세요: *고속 신호인가?* → TRACE. *고정 자산의 저속 텔레메트리이고 SQL에서 필요한가?* → TIMESERIES. *품목당 대량 동질 스칼라인가?* → SCALAR\_BULK. *나머지 전부* → SCALAR.

> **실제 라우팅 (SCK 팩).** PTR/FTR 대량 데이터 → SCALAR\_BULK, RUN(데이터로그)당 Parquet 하나, 유닛당 행, 테스트당 컬럼 — 수억 행 대신 Parquet 2.4 GB. 빈과 합불 → SCALAR: measurement 행 634,779개, 유닛당 정확히 3개. STDF 자체의 TSR 요약 통계 → 의도적으로 **저장 안 함** (I6: PTR에서 재계산, 아니면 어긋남). SEMES 쪽: 엔드포인트 신호는 원시 전용 TRACE; RF 파워와 챔버 압력은 TRACE+SCALAR — 웨이퍼당 요약은 행으로, 10 Hz 원시는 파일로.

레인이 존재하는 이유: 10 Hz × 200채널 트레이스를 행으로 풀면(unpivot) 수십억 개의 쓸모없는 인덱스 항목이 됩니다(불변식 I5가 금지). STDF 결과 수억 개를 개별 행으로 넣으면 Postgres만 힘들고 이득이 없습니다 — 랏당 컬럼형 Parquet 하나가 더 작고 더 빠릅니다. 행은 필터하고 조인하는 값을 위한 것, 파일은 DuckDB/pandas로 스캔하는 벌크를 위한 것입니다.

행과 파일의 다리: `file_ref`(Parquet 파일당 한 행: 어느 엔티티에 앵커되고, 어떤 레이아웃이고, 어느 시간 창이고, 어느 티어인지)와 `file_layout`(어느 `param_id`가 어느 물리 컬럼인지 — 파일 자체는 이름/단위를 저장하지 않습니다. 그건 사전의 일).

### 모두가 헷갈리는 구분: measurement vs observation

- `measurement` — (품목, 파라미터)당 스칼라 하나. PK는 `(entity_id, param_id)`. **키에 시간이 없습니다**: 웨이퍼 두께는 그 웨이퍼에 대한 하나의 사실입니다. *제품*의 이력은 엔티티를 늘려 표현합니다 (RUN마다, 웨이퍼마다 각자 엔티티).
- `observation` — **고정 자산**(챔버, 파이프라인 세그먼트)의 텔레메트리. *같은* 엔티티가 *같은* 파라미터를 영원히 측정받는 경우. PK는 `(entity_id, param_id, ts)` — 시간이 키에 있고, 그게 핵심입니다.

반복 측정값을 저장하려고 시간당 가짜 엔티티를 만들고 싶어진다면 — 멈추세요. 바로 그걸 막으려고 `observation`이 존재합니다.

## 6. 공장 언어로 말하고, 모델 언어로 저장한다

현장 사람들은 PV, SV, SVID, 트래블, 코너라고 말합니다. 번역표입니다 (모니터에 붙여두세요):

| 현장 용어 | DB 저장 위치 |
| --- | --- |
| **PV** (실측값) | `measurement` 행 (SCALAR), 또는 `file_ref` 뒤의 Parquet (TRACE/SCALAR\_BULK) |
| **SV** (설정값 — 의도) | `param_spec` 행, `spec_kind='SETPOINT'`, 컨텍스트 = RECIPE\_VERSION. 설정값의 집은 정확히 **하나** — 절대 measurement로 저장하지 않음 |
| **테스트 리밋** | `param_spec` 행, `spec_kind='LIMIT'`, 컨텍스트 = TEST\_PROG\_VER |
| **레시피** | PROCESS 도메인의 `RECIPE` → `RECIPE_VERSION` 엔티티 |
| **SVID / 테스트 번호** | `param_def.native_id` |
| **테스트 코너** (같은 테스트를 25 °C와 125 °C에서) | 코너당 `param_def` 하나 + `param_condition` 행; 공유 `param_concept`이 묶음 |
| **랏 트래블 / 이력** | `link` 행 (PROCESSED\_IN / TESTED\_ON), 컨텍스트 = 실행된 레시피/프로그램 버전 |
| **BOM / 원자재 계보** | `link` 행 (ASSEMBLY / MERGE / SPLIT) |
| **"이 숫자는 어느 파일에서 왔나?"** | `measurement.batch_id` → `ingest_batch.source_uri` — 조인 한 번 |

깊은 패턴에 주목하세요: **PV와 SV는 절대 섞이지 않습니다.** 실측은 VALUE 상자에, 의도는 SPEC 상자에 — 그것을 지배하는 PROCESS 엔티티에 붙어 삽니다. 합불(pass/fail)은 언제나 둘 사이의 *조인*이지, 저장된 플래그가 아닙니다.

> **실제 스펙 행 (SCK 팩).** 리밋은 TEST\_PROG\_VER 엔티티에 붙습니다: `t1234` (Iddq) LIMIT 상한 5.0e-6 A · `t1001` (Continuity VDD) LIMIT 0.2–0.8 V — 로드에 이런 행이 2,786개. 다른 사이트가 같은 테스트 프로그램을 다른 리밋으로 돌리면, 자기만의 TEST\_PROG\_VER 엔티티와 자기만의 `param_spec` 행으로 온보딩됩니다 — 서로 다른 리밋이 행으로 공존, 스키마 변경 없음.

## 7. 실습 예제: 어느 웨이퍼의 하루

**실제 SEMES 팩의** 엔티티와 파라미터로 데이터를 끝까지 따라가 봅시다. 이야기: 랏 **LOT2291**의 웨이퍼 **7**이 장비 **ETCH-03**의 챔버 **C03**에서, 레시피 **ETCH-A** 버전 **V03**으로 식각됩니다. 장비는 10 Hz 챔버 압력 트레이스와 웨이퍼당 압력 요약을 기록하고 (`SVID14`, TRACE+SCALAR 레인), 이후 웨이퍼 위 사이트들에서 막 두께가 측정됩니다 (`SVID31`, grain SITE, SCALAR 레인). (엔티티 id와 파라미터는 팩의 실제 행이고, 아래의 구체적 수치는 예시입니다.)

**레지스트리 행 (SEMES 소스 온보딩 때 한 번 작성 — 웨이퍼마다가 아님):**

- `entity_type`: LOT (PRODUCT), WAFER (부모 LOT), SITE와 RUN (WAFER 아래 형제), TOOL (EQUIPMENT), CHAMBER (부모 TOOL), RECIPE (PROCESS), RECIPE\_VERSION (부모 RECIPE).
- `entity_key_def`: "WAFER는 필드 `wafer_no`로 명명, 접두 `W`, 규칙 `DIGITS_PAD2`" — 그래서 웨이퍼 7은 어느 파일에서 오든 매번 `W07`이 됩니다. 명명은 파서 코드가 아니라 *설정*입니다.
- `param_def` (실제 팩 행): (`SEMES`, `SVID14`, "Chamber Pressure", grain WAFER, TRACE+SCALAR)과 (`SEMES`, `SVID31`, "Film Thickness", grain SITE, SCALAR).
- `file_layout_set` + `file_layout`: TRACE 레인 레이아웃 하나; Parquet의 `press` 컬럼이 파라미터 SVID14.

**이 이벤트에 대해 로더가 쓰는 행:**

1. `ingest_batch`: 한 행 — 소스 SEMES, 파일 URI, kind INGEST. 그 `batch_id`가 아래 모든 행에 찍힙니다. (이력이 항상 먼저.)
2. `entity`: LOT2291 → 웨이퍼를 resolve-or-create (두께가 도착하면 SITE 자식들도). `natural_key`는 `LOT2291/W07`이 됩니다 — 리졸버가 명명 규칙에서 생성, 로더가 타이핑하지 않음.
3. `link`: 웨이퍼 → 챔버 `ETCH-03/C03`, kind PROCESSED\_IN, 컨텍스트 = 레시피 버전 엔티티 `ETCH-A/V03`, 소스가 제공하면 점유 시간 창(`window_start`/`window_end`)도 함께.
4. `measurement`: grain이 둘이라 집도 둘 — (웨이퍼, SVID14) = 웨이퍼당 압력 요약 (TRACE+SCALAR의 SCALAR 절반), 그리고 (사이트, SVID31) = 두께 512.3 nm는 웨이퍼가 아니라 *SITE 엔티티* `LOT2291/W07/S01`에 안착. 이 파일이 재전송되면 로더는 *같은* 행에 에러 없이 안착합니다 — 재전송이 *다른 값*을 담고 있으면 로드는 **크게 실패**합니다 (덮어쓰기는 CORRECTION 배치만 가능하고, 그마저 옛 값을 `measurement_audit`에 먼저 보관합니다).
5. 트레이스: 원시 10 Hz 압력 샘플은 Parquet 파일로; `file_ref` 한 행이 가리킵니다 (TRACE 레인, `sampling_hz` = 10) — TRACE+SCALAR의 TRACE 절반.
6. 설정값: `param_spec` (SVID14, 컨텍스트 = `ETCH-A/V03`, SETPOINT) — 레시피 버전 등록 때 작성, 웨이퍼마다가 아님.

**이제 모든 질문에 집이 하나씩 있습니다:**

- "W07의 평균 압력은?" → 웨이퍼 grain의 `measurement`.
- "압력 곡선 보여줘" → `file_ref`로 Parquet을 찾아 DuckDB/pandas로 읽기.
- "압력이 설정값대로였나?" → V03의 `param_spec` SETPOINT와 비교.
- "어느 챔버가, 어느 레시피로?" → `link` 행.
- "웨이퍼 전면의 두께 균일도는?" → W07 아래 SITE grain 행들 — grain의 실전.
- "512.3은 어느 파일에서 왔나?" → `batch_id` → `ingest_batch.source_uri`.

웨이퍼 7이 재식각되면: RUN 엔티티 `LOT2291/W07/ETCH#1`, `…/ETCH#2`가 생깁니다 — 두 번째 패스는 *다른* 챔버에서 돌 수도 있고, 그것은 구조(새 RUN + 자기 링크)로 담기지, 절대 타임스탬프 키가 아닙니다 (I1).

일어나지 **않은** 일에 주목하세요: `semes_wafer_etch` 같은 테이블은 만들어지지 않았습니다. 내일 다른 에처 고객이 와도 같은 스키마에 레지스트리 행이 늘어날 뿐입니다.

### 테스트 라인에서의 같은 패턴 (SCK, 60초)

파이널 테스트: LOT → RUN → UNIT. **RUN은 STDF 데이터로그 하나**입니다 (`local_id` = `TEST_COD#<sessionTS>-<crc(filename)>`): `PART_ID`는 데이터로그별 카운터라서, RUN 레벨이 없으면 한 랏의 두 데이터로그에서 온 부품들이 충돌합니다 — I1이 실제 버그를 푸는 장면입니다.

값: 유닛 211,593개, 유닛당 수천 개의 파라메트릭 테스트 — 값 수억 개. 이들은 measurement 행이 되지 *않습니다*: PTR/FTR은 **SCALAR\_BULK** — RUN당 Parquet 하나(유닛당 행, 테스트당 컬럼), 총 Parquet 2.4 GB, measurement 행 0개. 빈과 합불은 SCALAR 행 — 634,779개, 유닛당 3개. 테스트 리밋은 TEST\_PROG\_VER 단위의 `param_spec` LIMIT 행 2,786개.

계보: 같은 파일이 각 다이의 출처를 QFPROM 퓨즈 리드백으로 나르므로, 같은 데이터로그에서 두 번째 PRODUCT 트리 `FAB_LOT → WAFER → DIE`가 세워지고, DIE→UNIT ASSEMBLY 링크 210,890개가 더해집니다. 그 보상: 다이 `STWGUX/W15/X085Y040`은 **네 번** 테스트되었고 매번 하드 빈 5 — 전체 재테스트 이력이 조인 한 번입니다. 로드: 26/26 파일, 약 40 GB, 실패 0; 재실행 시 동일 카운트 (멱등성 증명). 같은 여섯 상자, 전혀 다른 산업. *(카운트는 v1.1 *`sck` 인스턴스의 as-built 수치 — 필드 테스트 Test 1이 v2.2에서 재검증합니다.)

## 8. 깨면 안 되는 규칙들

라이브 01 페이지에 구속력 있는 불변식 14개(I1–I14) — 헌법 — 가 있습니다. 개수와 원문은 거기서 확인하세요. 실무에서 부딪히는 방식대로 묶으면:

**식별자 (I1, I2, I3, I14) — 깨면 가장 아픈 것들:**

- **시간이나 장비로 식별자를 만들지 않는다** (I1). 월요일에 측정하고 화요일에 재측정한 웨이퍼는 웨이퍼 *하나*지 둘이 아닙니다. 처리 *패스*의 구분은 RUN 레벨에서 (`local_id = STEP + '#' + pass` — SEMES: `LOT2291/W07/ETCH#1`, `…/ETCH#2`), 절대 타임스탬프로 하지 않습니다. 시간과 장비는 `attrs`로.
- `natural_key`를 손으로 쓰지 않는다 (I2). 리졸버만이 명명 규칙에서 생성합니다.
- **명명 규칙 행을 수정하지 않는다** (I3). `entity_key_def`는 append-only: 명명이 바뀌면 새 버전을 INSERT하고 명시적 re-key 마이그레이션을 돌립니다. 제자리 수정은 다음 로드에서 조용히 모든 엔티티를 복제시킵니다.
- **하나의 **`type_code`, 하나의 사물 종류 (I14). 테스트 랏과 팹 랏을 둘 다 "LOT"으로 넣고 attrs 마커로 구분하면, 하류의 모든 생성 뷰가 두 종류를 조용히 섞습니다. 실제로 일어났던 일: SCK 첫 빌드가 OSAT 테스트 랏(`MIR.LOT_ID` 키, 자식 RUN)과 팹 랏(퓨즈 텍스트 키, 자식 WAFER)을 attrs 마커로 갈라 한 `type_code`에 넣었고 — 모든 "랏" 뷰가 둘을 섞었습니다. 수정: 별도 `FAB_LOT` 유형 신설, 영향받은 엔티티 10개를 제자리 재타입, 재적재 불필요. 그게 팩 교훈 L-SCK-01이고 — I14가 불변식이 된 이유입니다.

**값 읽고 쓰기 (I4, I5, I6, I11):**

- **조인은 항상 **`entity_id`로 (I4). `natural_key`는 해석/표시/접두 필터 전용.
- **원시 트레이스는 절대 행이 되지 않는다** (I5). 고속 질문은 DuckDB/pandas로 Parquet에서 답합니다.
- **부모 롤업을 부모의 measurement로 저장하지 않는다** (I6). 웨이퍼 값들의 랏 평균 = 읽을 때 계산하거나, 명확히 분리된 파생 테이블로. 저장된 롤업은 원본과 어긋나기 시작합니다.
- **(품목, 파라미터)당 스칼라 하나** (I11). 같은 고정 자산의 반복 값 → `observation`. 제품의 반복 값 → 엔티티 레벨(RUN)이 빠진 겁니다.

**변경 규율 (I7, I8, I12, I13):**

- **레지스트리 행을 DELETE하지 않는다** (I7) — `is_active = FALSE`로 비활성화.
- **파생물은 전부 폐기 가능** (I8) — 와이드 테이블, 뷰, 그래프 프로젝션. 절대 진실 원천이 아니고, 절대 거기에 다시 쓰지 않습니다.
- **로드는 멱등(idempotent); 값이 다른 재전송은 크게 실패** (I12). 조용한 덮어쓰기는 절대 없음. 의도적 수정은 옛 값을 먼저 보관하는 CORRECTION 배치로만.
- **절대 추측하지 않는다** (I13). 명명 규칙 누락, 미지의 정규화 규칙, 모호한 타임스탬프 → 로드는 *명확한 에러로 실패*합니다. 여기서 "최선을 다한 파싱(best-effort)"은 기능이 아니라 버그입니다. `attrs`의 모든 타임스탬프는 UTC ISO-8601이어야 하며 아니면 로드가 실패합니다.

**구조 (I9, I10):** 파라미터당 정의 하나 (I9); 도메인은 트리로 절대 못 넘고 `link`로만 (I10).

규칙이 당신을 막는 것처럼 느껴진다면, 대개 규칙이 *제대로 작동하는* 중입니다 — 11장을 보세요.

## 9. SQL을 쓰기 전: 세 가지 확인

번역표 위에 이것도 붙여두세요.

**확인 1 — 이 인스턴스의 스키마 버전은?**

```sql
SELECT version FROM schema_migration ORDER BY applied_at DESC LIMIT 1;
-- 테이블이 없으면 => v2.2 이전 인스턴스
```

스펙은 *목표*를 기술하지, 디스크 상태가 아닙니다. 인스턴스는 합법적으로 뒤처질 수 있습니다 (로드된 `sck` 인스턴스는 마이그레이션 전까지 아직 v1.1 — 08 페이지의 버전 보드 확인). v1.1 인스턴스에 v2.2 SQL을 쓰면 오후가 통째로 날아갑니다.

**확인 2 — 파라미터의 레인은?**

```sql
SELECT param_id, grain, value_type, storage_class
FROM param_def WHERE source = :src AND native_id = :nid;
```

TRACE나 SCALAR\_BULK 파라미터는 *설계상* `measurement` 행이 0개입니다. 파일 레인 파라미터에 대한 빈 스칼라 쿼리 = 데이터 누락이 아니라 당신의 버그.

**확인 3 — grain은?** `param_def.grain`이 값이 붙는 엔티티 레벨을 지정합니다. 웨이퍼 단위 값을 랏 레벨에서 트리 워크 없이 쿼리하면 아무것도 안 나옵니다.

그리고 두 가지 상시 습관: 레지스트리와 링크에서 `is_active` 필터하기 (소프트 폐기 ≠ 삭제), 사용자가 랏 없이 "웨이퍼 7"이라고 하면 — **어느 랏인지 물어보기**. 절대 추측하지 않기.

## 10. 첫 쿼리들

검증된 전체 쿡북은 라이브 03 페이지에 있습니다. 아래 넷이면 첫날의 80%를 커버합니다.

**사람 이름을 한 번 해석하고, 이후엔 id 사용:**

```sql
SELECT entity_id FROM entity WHERE natural_key = 'LOT2291/W07';
```

**스칼라 값과 그 출처:**

```sql
SELECT m.value_num, b.source_uri
FROM measurement m
JOIN ingest_batch b ON b.batch_id = m.batch_id
WHERE m.entity_id = :wafer_id AND m.param_id = :param_id;
```

**이 웨이퍼를 어느 챔버가, 어느 레시피로 처리했나:**

```sql
SELECT eq.natural_key AS chamber, rv.natural_key AS recipe_version
FROM link l
JOIN entity eq ON eq.entity_id = l.dst_entity_id
LEFT JOIN entity rv ON rv.entity_id = l.context_entity_id
WHERE l.src_entity_id = :wafer_id AND l.link_kind = 'PROCESSED_IN' AND l.is_active;
```

**랏 레벨 롤업 — 읽을 때 계산 (절대 다시 저장하지 않음):**

```sql
SELECT avg(m.value_num)
FROM entity child
JOIN measurement m ON m.entity_id = child.entity_id AND m.param_id = :param
WHERE child.parent_id = :lot_id;
```

**SCK 팩의 실제 쿼리 — 패스를 가로지르는 다이별 재테스트 이력** (팩의 대표 쿼리를 v1.1 형태에서 v2.2의 `link` 형태로 번역; 마이그레이션된 인스턴스에서만 실행):

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
-- DIE → ASSEMBLY link → UNIT → 부모 RUN 순서로 걷기; 로드에서 4,249개 다이가 다중 패스 이력 보유
```

파일 레인 데이터: 파일 찾기(`file_ref`를 `anchor_entity_id`나 시간 창으로), `tier` 먼저 확인 (HOT이 아닌 읽기는 의도적으로 크게 실패), 그다음 `file_layout`으로 컬럼을 매핑하며 DuckDB나 pandas로 Parquet 읽기.

## 11. 새 데이터가 도착하면 — 그리고 안 맞으면

### 런북 루프 (02 페이지 — 첫 소스 전에 전체를 읽으세요)

황금률: **코드보다 설계 먼저. 절대 파서부터 시작하지 않는다.**

1. **소스 조사** — 사물은 무엇인가, 무엇으로 명명되는가, 무엇이 어떤 grain/속도로 측정되는가, 리밋/설정값이 있는가, 코너와 재테스트는, 타임존은.
2. **매핑 초안** — 엔티티 트리, 명명 규칙, 모든 파라미터의 레인, 스펙, 관계.
3. **작성된 매핑에 사인오프 받기** (팩 스펙, 형식은 06 페이지). 리뷰에서 잡힌 매핑 실수는 수정 한 번, 로드 후에 잡히면 재로드에 수정 사이클까지.
4. **그다음에야 로더 작성** — 02 페이지의 로더 계약을 따라야 합니다: `ingest_batch`를 열고 모든 행에 스탬프; 멱등하게 resolve-or-create; 동일한 재전송은 조용히 안착, 다른 재전송은 크게 실패; 타임스탬프 검증; 검증 없이 커밋 금지.
5. **게이트 증명**: 전체 로드 그린; 재실행 시 동일 카운트에 에러 0; 고의로 변조한 재전송이 실패한 후 CORRECTION으로 성공; 카운트가 소스와 대사(reconcile). 그다음 06 페이지에 팩 등록.

### 데이터가 안 맞을 때 — 가장 중요한 습관

언젠가 스키마가 행으로 흡수 못 하는 데이터를 만납니다. 이것은 *예상된 일*입니다 — 04 페이지가 바로 그것을 위해 존재합니다. 규율:

1. **조용히 패치하지 않는다.** 컬럼을 추가하지 않는다. 포크하지 않는다.
2. **그 모양을 **`attrs`에 던져 넣지 않는다. "일단 attrs에 넣자"는 그 자체가 기록 대상 커버리지 결함입니다.
3. **04 페이지에 기록한다**: 어느 소스, 어떤 데이터 모양, 무엇을 시도했고, 어떤 규칙/가드가 막았는지. 이연 항목 표를 먼저 확인하세요 — 당신의 케이스가 이미 재개 트리거와 함께 기다리고 있을 수 있습니다.
4. 수정은 05 페이지의 비준된 결정(D-###)으로 도착합니다. 이전 결정을 대체(supersede)할 수도 있습니다.

처음엔 느리게 느껴집니다. 하지만 결정 48개를 지나고도 스키마가 포크 다섯 개가 아니라 여전히 하나인 이유가 바로 이것입니다. 결함을 기록하는 것이 장기적으로는 *가장 빠른 길*입니다.

> **이 루프를 실제로 한 바퀴 (SCK 코너).** SCK의 코너 데이터 — 같은 테스트를 다른 온도에서 — 는 당시 스키마에 맞지 않았습니다. `attrs`에 욱여넣지도, 로컬 컬럼으로 해킹하지도 않고, **로드에서 제외**한 채 백로그 항목 C-04로 기록했습니다. C-04는 나중에 결정 D-038(조건 모델: 조건별 `param_def` + `param_condition` + 공유 개념)이 되었고, 제외해 두었던 코너 파일들은 이제 필드 테스트 Test 2로서 `sck` 마이그레이션을 기다립니다. 기록 → 결정 → 로드: 루프가 끝까지 작동한 실례.

### 변경은 어떻게 일어나는가 (시스템과 싸우지 않기 위해)

- 페이지 01–08은 오직 05 페이지의 결정을 통해서만 바뀝니다. 로그는 append-only이며, 틀린 결정은 *대체*되지, 수정으로 지워지지 않습니다.
- 스키마 변경은 **오직** 번호 붙은 마이그레이션 스크립트로만 배포됩니다 (dev → verify → promote, 마지막에 `schema_migration` 행 삽입). **라이브 인스턴스에 손 타이핑 ALTER는 절대 금지. 예외 없음.** 작은 것도, 당신 것도 안 됩니다.
- 크게 실패하는 에러가 당신에게 도달하는 것(파티션 누락, HOT 아닌 티어, 가드 거부)은 시스템이 설계대로 작동하는 것입니다. 해법은 설계된 조치(파티션 생성, 복원-또는-수용 결정, 데이터 수정)이지, 재시도 루프나 우회가 아닙니다.

## 12. 첫 2주 계획

**1일차 — 오리엔테이션.** 이 매뉴얼 읽기. 그다음 라이브 01 페이지(DDL 전까지)와 02 페이지 훑어보기. 여섯 상자와 번역표는 암기.

**2일차 — 스펙 정독.** 01 페이지 전체를 DDL 블록과 불변식 14개까지 포함해 읽기. 스크래치 Postgres 16에 DDL 세 블록을 직접 실행 — 가드가 생성되는 걸 지켜보는 게 읽기보다 많이 가르칩니다. [00a · 테이블·컬럼 레퍼런스](./00a-KO-table-column-reference.md)를 DDL 옆에 띄워두세요.

**3일차 — 트리 실습.** 스크래치 인스턴스에 7장의 실습 예제를 손으로 INSERT: 유형, 명명 규칙, 엔티티, 링크 하나, measurement 하나, 스펙 하나. 그리고 일부러 깨보기: `local_id` 안의 `/`, 한 measurement 행에 값 두 개, 스펙에 PRODUCT 컨텍스트, 불법 링크(웨이퍼 PROCESSED\_IN 다른 웨이퍼). 모든 가드와 CHECK가 발동하는 걸 확인. (엔티티는 리졸버 방식 SQL로 넣되, 실무에서는 `natural_key`를 절대 손으로 쓰지 않는다는 걸 기억.)

**4일차 — 데이터 읽기.** 03 페이지 읽기. 실제 인스턴스에 세 가지 확인 실행 (읽어도 되는 인스턴스를 먼저 물어보기). 쿡북 쿼리 재현. SCALAR\_BULK 파라미터를 일부러 `measurement`에서 쿼리해 빈 결과 관찰 — 그 느낌을 기억하기.

**5일차 — 레인과 파일.** 실제 `file_ref` 하나 열어 `file_layout` 조인, DuckDB로 Parquet 읽기. 롤업을 읽을 때 계산해보기.

**2주차 — 실전 세계.** 04 페이지(이연 항목 숙지 — 반드시 만나게 됩니다), 05 페이지(결정 48개 훑기; D-026, D-030, D-033, D-034, D-038, D-042는 정독), 06 페이지(SCK 팩 행과 교훈 L-SCK-01), 07·08 페이지(필드 테스트 게이트와 마이그레이션 규율) 읽기. 그다음 작은 실제 과제 하나: 누군가를 위한 쿼리, 또는 진행 중인 필드 테스트 담당자 섀도잉. 구조적인 무언가를 제안하기 전에는 무회귀 규칙(no-regression rule)을 먼저 익히세요: 결정 로그 스캔, 요구사항 워크, 팩 교훈 스캔.

**졸업 시험:** 다른 신입에게 왜 `measurement` 키에 타임스탬프가 없는지, 그리고 타임스탬프가 필요해 보이는 각 상황에서 대신 무엇을 해야 하는지 설명하기. 설명할 수 있다면 모델을 이해한 것입니다.

## 13. 용어집

| 용어 | 의미 |
| --- | --- |
| **엔티티 (entity)** | 추적되는 모든 사물 (랏, 웨이퍼, 챔버, 레시피 버전) — `entity`의 한 행 |
| **도메인 (domain)** | 엔티티가 사는 트리: PRODUCT, EQUIPMENT, PROCESS |
| **grain** | 파라미터가 붙는 엔티티 레벨 (WAFER 단위, LOT 단위) |
| **레인 / storage class** | 파라미터 값의 물리적 위치: SCALAR, SCALAR\_BULK, TRACE, TRACE+SCALAR, TIMESERIES |
| **natural key** | 파생 생성되는 전체 경로 이름 (`LOT2291/W07`); 해석/표시/접두 전용 |
| **명명 규칙 (naming recipe)** | 유형의 `local_id`가 소스 필드에서 어떻게 만들어지는지 정의하는 `entity_key_def` 행들 |
| **정규화 (canonicalization)** | 소스 값을 id로 표준화하는 폐쇄 규칙 집합: AS\_IS, UPPER, DIGITS\_PAD2, DIGITS\_PAD3 |
| **팩 (pack)** | 온보딩된 소스 하나의 작성된 매핑 스펙 + 증빙 (06 페이지) |
| **배치 / 이력 (provenance)** | 모든 로드가 여는 `ingest_batch` 행; 그 id가 모든 작성 행에 찍힘 |
| **CORRECTION** | 값을 덮어쓰는 유일한 합법 경로: 옛 값을 `measurement_audit`에 먼저 보관하는 배치 |
| **커버리지 결함** | 행으로 안 들어가는 데이터; 04 페이지에 기록, 결정으로 수정 — 조용한 패치 금지 |
| **D-###** | 05 페이지의 비준된 결정 (append-only; 대체는 있어도 수정은 없음) |
| **I1–I14** | 01 페이지의 구속력 있는 불변식 — 헌법 |
| **L-###** | 06 페이지의 팩 교훈 (L-SCK-01이 불변식 I14가 됨) |
| **개념 (concept)** | 소스 간 표준 물리량 (`param_concept`), 예: CHAMBER\_PRESSURE; 단위 변환은 scale/offset으로 |
| **코너 / 조건** | 같은 물리량을 다른 조건에서 측정 → 조건당 `param_def` 하나 + `param_condition` 행, 공유 개념 |
| **가드 (guard)** | 불변식을 강제하는 크게-실패 DB 트리거 (도메인 규칙, 레인, 순환, 스펙 컨텍스트) |
| **티어 (tier)** | `file_ref` 생애주기: HOT / COLD / PURGED (purged여도 묘비 행은 영구 보존) |
| **인스턴스** | 고객 하나의 물리 DB (`sck`, `semes`, …) — 같은 스키마, 각자의 마이그레이션 일정 |

## 14. 한 장 치트시트

**모델:** 사물 = 세 도메인 트리(PRODUCT / EQUIPMENT / PROCESS)의 `entity` 행 · 트리 간 사실 = `link_type`에 등록된 종류를 갖는 `link` 행 · 측정값 = `param_def.storage_class` 레인에 따라 `measurement` / `observation` / `file_ref` 뒤의 Parquet · 의도 = `param_spec` · 모든 행에 `batch_id` 스탬프.

**SQL 전:** ① 인스턴스 버전 (`schema_migration`) ② 파라미터 레인 ③ 파라미터 grain.

**철칙:** 조인은 `entity_id` · 식별자에 시간/장비 금지 · natural\_key는 파생, 손으로 금지 · 트레이스는 행이 되지 않음 · 롤업은 계산, 저장 금지 · 설정값은 `param_spec`에만 · 조용한 덮어쓰기 금지 (CORRECTION만) · 레지스트리 DELETE 금지 (`is_active=FALSE`) · 파생물 = 폐기 가능 · 안 맞으면? → 04 페이지에 기록, `attrs`에 넣지 말 것 · 스키마 변경은 번호 붙은 마이그레이션 + D-### 로만 · 모호하면: **크게 실패, 절대 추측 금지**.

**어디를 볼까:** [00a 컬럼 레퍼런스](./00a-KO-table-column-reference.md) · [01 스펙·불변식](./01-schema-spec-v2.2.md) · [02 온보딩 런북·로더 계약](./02-data-engineer-runbook.md) · [03 쿼리 쿡북](./03-backend-query-conventions.md) · [04 알려진 한계](./04-known-limits-deferred.md) · [05 결정 로그](./05-decision-log.md) · [06 팩](./06-case-registry.md) · [07 필드 테스트](./07-field-test-plan.md) · [08 운영](./08-operations.md).

**막히면:** 실패 카탈로그(02 페이지) → 이연 항목(04 페이지) → 결정 로그(05 페이지) 순으로 확인. 그다음 질문하세요. 질문이 추측보다 언제나 낫습니다.
