# MD-Files — 프로젝트 구성 문서 모음

사내에서 담당한 3개 프로젝트의 **프론트엔드 → Spring Boot 백엔드 → DB → 배포 환경** 구성을, 코드 없이 문서만 보고도 같은 구조를 재구현할 수 있는 수준으로 정리한 저장소다. 로그인/인증, 사용자 관리, 메뉴·권한 관리 흐름을 특히 자세히 다룬다.

| 프로젝트 | 폴더 | 백엔드 리포 | 프론트 리포 | 한 줄 요약 |
|---|---|---|---|---|
| **SCK** (DUTCHBOY OSAT) | [`sck/`](./sck/README.md) | `sck-server-spring` (Spring Boot, MyBatis, PostgreSQL, Redis Pub/Sub → SSE) | `sck-server-react` (React 18 + Vite 7, 서브모듈) | 반도체 OSAT 테스트 장비 실시간 모니터링·규칙/알람·분석 플랫폼 |
| **SEMI** (Semi Common System) | [`semi/`](./semi/README.md) | `semi-spring` (Spring Boot 4, JDK 25, Flyway, MinIO) | `semi-react` (React + Vite, FSD, i18next 13 ns) | 반도체 공정 trace/target/일일 리포트 공통 시스템 |
| **DUTCHBOY** (센서/설비 AI 분석) | [`dutchboy/`](./dutchboy/README.md) | `dutchboy-sensor-api` (Spring Boot 3.1, Java 17, MyBatis, PostgreSQL + TimescaleDB, Flyway) | `front-monorepo` (pnpm + Turborepo, `apps/semes-v2`) | 설비 센서 데이터 대시보드·모델 분석·PM 관리 |

## 각 프로젝트 폴더 구성

| 파일 | 내용 |
|---|---|
| `README.md` | 프로젝트 개요, 전체 아키텍처, 배포 환경, **로그인·사용자·메뉴 end-to-end 흐름**(프론트 ↔ 백엔드 ↔ DB 를 한 장으로) |
| `backend.md` | Spring 백엔드: 기술 스택, 패키지 구조, 설정/환경변수, 인증, 사용자/메뉴/권한 API, 공통 인프라, 빌드/배포, 재구현 체크리스트 |
| `frontend.md` | React 프론트: 기술 스택, 디렉터리, 부트스트랩, 라우팅, 인증/세션, 사용자·메뉴 화면, 공통 컴포넌트, API 레이어, 상태, 테마/i18n, 빌드/배포 |
| `database.md` | DB: ERD, 시스템 테이블 DDL(사용자·역할·권한·메뉴·이력), 도메인 테이블 목록, 시드 데이터, 마이그레이션 |

## 작성 원칙

- 2026-10-06 기준 각 리포의 실제 코드를 읽고 작성했다(기준 커밋은 각 문서 상단 메타 블록에 표기). 코드에서 확인하지 못한 부분은 "(추정)" 으로 표시했다.
- **비밀값·사내 호스트·개인 식별 정보는 제외**했다. 환경변수는 이름만 적고 값은 `<PLACEHOLDER>` 로 바꿨으며, 사내 서버는 `<STAGING_HOST>` 같은 역할명으로 일반화했다.
- 다이어그램은 GitHub 이 렌더링하는 mermaid 로 작성했다.
