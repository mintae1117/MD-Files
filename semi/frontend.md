# semi-react 프론트엔드 기술 문서

| 항목 | 값 |
| --- | --- |
| 대상 리포 | `semi-react` (Semi Common System / PRISM 프론트엔드, 로컬 경로 `SEMES/semi-react`) |
| 기준 브랜치 · 커밋 | `feature/<개인 브랜치>` · `855af1b54d9fd9c78d45b957645753a1c46cdf22` (2026-10-02, `Merge branch 'feat/equipment-device-list' into 'dev'`) — 작성 시작 시점에 `dev`와 동일 커밋 |
| 주의 | 작성 도중 같은 브랜치가 `f79a4be9` (2026-10-05)로 전진했다. 본문은 전부 `855af1b5` 기준이며, 이후 차이는 §16.3에 요약했다. |
| 작성일 | 2026-10-06 |
| 짝 백엔드 문서 | `backend.md` (semi-spring, Spring Boot 세션 인증 서버 — 별도 문서) |
| 문서 목적 | 코드 없이 이 문서만으로 동일한 프론트엔드를 재구현할 수 있는 수준의 명세. 모든 서술은 소스에서 확인한 사실이며, 추정은 "(추정)"으로 표시한다. 근거 경로는 리포 루트 기준 `(src/...)`. |
| 보안 | 비밀값·사내 호스트·IP·실명은 `<PLACEHOLDER>`/역할명으로 일반화했다. |

> 알려진 사실 중 **코드로 검증해 정정한 것**: ① 기본 언어는 `en`이 아니라 **`ko`** 다(2026-08-27 변경, `src/shared/lib/i18n/constants.ts`). ② i18n 네임스페이스는 13개가 아니라 **27개**다(`src/shared/lib/i18n/resources.ts`). ③ 사이드 메뉴 라벨은 "menu.json 우선 + menuName 폴백"이 아니라 **DB `menuName`/`menuNameEn` 우선 → `menu:names.*` → `menu:<path>` → path 순 폴백**이다(`src/shared/lib/i18n/menuName.ts`).

## 목차

1. [개요](#1-개요)
2. [디렉터리 구조](#2-디렉터리-구조)
3. [설정/환경](#3-설정환경)
4. [앱 부트스트랩](#4-앱-부트스트랩)
5. [라우팅](#5-라우팅)
6. [인증/로그인](#6-인증로그인)
7. [사용자(계정) 관리 화면](#7-사용자계정-관리-화면)
8. [메뉴/권한](#8-메뉴권한)
9. [레이아웃/공통 컴포넌트](#9-레이아웃공통-컴포넌트)
10. [API 레이어/서버 상태](#10-api-레이어서버-상태)
11. [상태 관리](#11-상태-관리)
12. [스타일/테마/i18n](#12-스타일테마i18n)
13. [그 외 도메인 화면](#13-그-외-도메인-화면)
14. [테스트/품질](#14-테스트품질)
15. [빌드/배포](#15-빌드배포)
16. [재구현 체크리스트](#16-재구현-체크리스트)

---

## 1. 개요

### 1.1 역할

반도체 설비 데이터 플랫폼(브랜드명 PRISM / DUTCHBOY SEMI, 과거 SEMES Common System)의 **단독 SPA 프론트엔드**다. Spring Boot 백엔드(`semi-spring`, 세션 기반 인증)와 `/api/**`로 통신하며, 다음 세 묶음의 화면을 제공한다.

| 묶음 | 내용 | 진입 컨텍스트 |
| --- | --- | --- |
| 관리자 콘솔 | 법인/사이트·기준정보(공정·라인·장비·사업부)·수집원·레시피/센서·계정·역할·메뉴·Feature·모델(등록/계약/Config/관리)·운영 로그(감사·에러·성능·디스크) | `ADMIN/ADMIN` 의사 컨텍스트(SU) — 정적 네비게이션 `STATIC_NAV_ITEMS` |
| 테넌트(법인/사이트) 화면 | 대시보드 4종(Anomaly Score·진단 분석·챔버 가동현황·알람 현황), 트레이스 분석(TTTM), Data Chart, FDC/DCOP, EC Compare·Parameter·Comparator, 데일리 리포트, 알람 이벤트 등 | 세션 활성 법인/사이트 — DB 메뉴 트리(`/api/me/menus`) |
| 공용 셸 기능 | 워크스페이스 탭(keep-alive), 테마/언어/전체화면, 비밀번호 변경, VOC(문의) 버튼, 도움말(매뉴얼) 버튼, 관리 대상 선택기 | 모든 화면 |

### 1.2 기술 스택 (버전은 `package.json` 기준, `^` 캐럿 범위)

| 범주 | 라이브러리 | 버전 | 비고 |
| --- | --- | --- | --- |
| UI 런타임 | react / react-dom | ^18.3.1 | StrictMode **미사용** — react-grid-layout의 react-draggable이 StrictMode 이중 마운트에서 드래그가 깨짐 (`src/app/main.tsx`) |
| 언어 | typescript | ^5.8.3 | `strict`, `noUnusedLocals/Parameters`, `moduleResolution: bundler` |
| 빌드 | vite / @vitejs/plugin-react | ^5.4.19 / ^4.5.2 | dev 포트 3000, `/api` 프록시 |
| 라우터 | react-router-dom | ^6.30.1 | `createBrowserRouter` + 내부 `<Routes>`; `UNSAFE_LocationContext/NavigationContext`로 keep-alive 구현 |
| 클라이언트 상태 | zustand | ^5.0.14 | `persist` 미들웨어(localStorage/sessionStorage) |
| 서버 상태 | @tanstack/react-query | ^5.80.7 | `subscribed`(v5.69+) 옵션으로 숨김 탭 구독 차단 |
| 가상화 | @tanstack/react-virtual | ^3.14.8 | DataTable `virtualized` |
| HTTP | axios | ^1.7.9 | `withCredentials` + XSRF 쿠키/헤더 |
| 레이아웃/위젯 | react-grid-layout | 2 (`react-grid-layout/legacy` import) | WidgetBoard(창 드래그/리사이즈) |
| 드래그 | @dnd-kit/core · sortable · modifiers · utilities | ^6.3.1 · ^10.0.0 · ^9.0.0 · ^3.2.2 | 워크스페이스 탭 순서, SortableDataTable |
| 리사이즈 | react-rnd | ^10.5.3 | (사용처는 모달/패널 리사이즈 — 추정, 직접 확인 안 함) |
| 툴팁 | react-tooltip | ^6.0.8 | 앱당 단일 인스턴스 `AppTooltip` |
| 아이콘 | react-icons (Lucide `react-icons/lu`) | ^5.7.0 | 이모지/직접 SVG 금지 규칙 |
| CSS | tailwindcss / postcss / autoprefixer | ^3.4.17 / ^8.5.4 / ^10.4.21 | Tailwind **v3**, `darkMode: ['selector', '[data-theme="dark"]']`, 토큰 CSS 변수 매핑 |
| 차트 | echarts / echarts-for-react | ^5.6.0 / ^3.0.6 | 유일한 차트 라이브러리(recharts는 2026-07 제거) |
| 리치텍스트 | @tiptap/react · starter-kit · table · highlight · text-align · text-style · core · pm | ^3.27.1 | 데일리 리포트 편집기 |
| 폼/검증 | (라이브러리 없음) | — | `useState` draft + 수동 검증, 공통 훅 `useSelectionDraft`/`useOrderedList` |
| 날짜 | (라이브러리 없음) | — | `Intl.DateTimeFormat` 기반 자체 유틸 `shared/lib/dateFormat.ts`, `timeZone.ts`, `formatInstant.ts` |
| i18n | i18next / react-i18next | ^26.3.6 / ^17.0.10 | 리소스 인라인 번들, 27 ns, 기본 `ko` |
| 테스트 | vitest / jsdom / @testing-library/react · jest-dom · user-event | ^2.1.9 / ^29.1.1 / ^16.3.2 · ^6.9.1 · ^14.6.1 | 테스트 파일 300개 |
| 린트/포맷 | eslint / typescript-eslint / eslint-plugin-boundaries / eslint-plugin-react-hooks / eslint-import-resolver-typescript / prettier | ^10.6.0 / ^8.62.1 / ^6.0.2 / ^7.1.1 / ^3.10.1 / ^3 | resolver는 **v3 고정**(v4는 legacy resolve API 미동작으로 규칙이 조용히 무력화) |
| 런타임 | Node.js 20 LTS, pnpm 9 | — | CI 이미지 `node:20-bookworm`, `corepack prepare pnpm@9`. `package.json`의 `packageManager`는 `pnpm@8.15.9`로 적혀 있어 README/CI(9)와 불일치 — 실제 lockfile은 v9 형식(README) |

### 1.3 지원 브라우저

- `browserslist`/`build.target` 미지정 → Vite 5 기본 타깃(`modules`: Chrome 87+, Firefox 78+, Safari 14+, Edge 88+)이 적용된다 (추정, 설정 부재에서 유추).
- 코드가 전제하는 브라우저 API: Fullscreen API(`requestFullscreen`), `BroadcastChannel`(창 간 컨텍스트 동기화), `ResizeObserver`, `sessionStorage/localStorage`, `(display-mode: fullscreen)` 미디어쿼리. 실사용 VOC에 Firefox 154/Linux 사례가 있어 Chromium 외 Firefox도 지원 대상이다 (`src/widgets/app-shell/ui/AppShell.tsx` 주석).
- `index.html`은 `lang="ko"`, 첫 페인트 전 테마를 `<html data-theme>`에 적용하는 인라인 스크립트를 가진다.

### 1.4 pnpm 워크스페이스 특이점

- `pnpm-workspace.yaml`이 `packages: [ "." ]`(자기 자신 하나)인 **단일 패키지 워크스페이스**라 `pnpm add <pkg>`는 거부된다 → **`pnpm add -w <pkg>`** 가 필수 (`CLAUDE.md`).
- `allowBuilds: { esbuild: true }`로 esbuild 포스트인스톨 빌드만 허용.
- pnpm 엄격 `node_modules` 때문에 ESLint resolver를 이름이 아닌 `require.resolve(...)` 절대 경로로 지정한다 (`eslint.config.js`).
- 의존성 설치 후 Vite/esbuild 오류가 나면 `pnpm install && pnpm rebuild esbuild` (README).

---

## 2. 디렉터리 구조

### 2.1 루트

```
semi-react/
├─ index.html                 # 엔트리 HTML(테마 FOUC 방지 스크립트, <title>PRISM)
├─ package.json / pnpm-lock.yaml / pnpm-workspace.yaml
├─ vite.config.ts             # dev proxy, alias @, vitest 설정 동거
├─ vitest.manifest.config.ts  # CI 전용: feature manifest JSON 덤프
├─ tsconfig.json / tsconfig.node.json
├─ tailwind.config.cjs / postcss.config.cjs
├─ eslint.config.js / .prettierrc / .gitmessage / .git-blame-ignore-revs
├─ .env.example / .env(git 제외)
├─ .gitlab-ci.yml             # verify(MR) / deploy(dev push)
├─ CLAUDE.md / AGENTS.md / README.md / MENU-GUIDE.md
├─ contract/report-view/*.json  # FE↔BE 계약 픽스처(리포트 사이트 소스)
├─ docs/                      # 설계 문서(ADR), superpowers/{plans,specs}
├─ scripts/check-i18n.mjs     # en/ko 키 패리티 + 한글 하드코딩 검사
├─ scripts/ci/feature-sync.sh, featureManifest.dump.ts
├─ scripts/create-dashboard-menus.mjs   # 대시보드 4메뉴 생성 1회성 스크립트
├─ public/docs/*.html         # ERD·파이프라인 정적 문서(관리자 도구에서 새 탭으로 열림)
├─ .claude/skills/*, .claude/agents/*, .agents/skills/*  # 에이전트 작업 절차서
└─ src/
```

### 2.2 FSD 레이어 (`src/`)

```
src/
├─ app/                      # 진입·라우팅·전역 스타일 (레이어 최상위)
│  ├─ main.tsx               # ReactDOM.createRoot + Provider 트리
│  ├─ routes/AppRoutes.tsx   # <Routes> 정의, 가드(ProtectedRoutes/LoginRoute/AdminSiteRoute)
│  ├─ routes/BackendMenuView.tsx  # 메뉴 기반 디스패치(featureCode → 페이지)
│  ├─ routes/TenantDataRoute.tsx  # 사업부 데이터 접근 게이트
│  ├─ reportSources/         # 리포트 소스 정의 전수 로드(테스트 전용) + 계약 테스트
│  └─ styles/{tokens,global,dashboardPanels}.css
├─ pages/        (42 슬라이스) 라우트 단위 화면. ui/ (+ model/ lib/ api/)
├─ widgets/      (3)  app-shell · model-version · process-scope
├─ features/     (5)  password-change · report-source-action · target-selector · trace-preset · workspace-tabs
├─ entities/     (34) 도메인 단위: api/ model/ lib/ ui/
├─ shared/       도메인 무관: api/ config/ lib/(i18n 포함) types/ ui/
├─ assets/       로고·배경 이미지 (app·pages·widgets에서만 import 가능)
├─ test/setup.ts vitest 셋업
└─ vite-env.d.ts ImportMetaEnv 선언
```

#### 레이어 책임과 임포트 규칙 (ESLint `boundaries/dependencies`가 강제, `eslint.config.js`)

| 레이어 | 책임 | import 허용 대상 |
| --- | --- | --- |
| `app` | 엔트리, Provider, 라우트 정의, **여러 page를 조합하는 디스패치**(BackendMenuView), 전역 CSS | pages·widgets·features·entities(배럴 `index.ts`만), shared(세그먼트 배럴 `*/index.ts`만), assets |
| `pages` | 라우트 1개 = 슬라이스 1개. thin 래퍼 지향(공유 CRUD 본문은 entities/ui로) | widgets·features·entities(배럴), shared, assets |
| `widgets` | 여러 화면이 쓰는 복합 블록(앱 셸, 모델 버전 패널, 공정 스코프 툴바) | features·entities(배럴), shared, assets |
| `features` | 사용자 인터랙션 단위(탭, 비밀번호 변경, 관리 대상 선택, 프리셋, 리포트 소스 저장) | entities(배럴), shared |
| `entities` | 도메인 API 모듈·모델·공유 UI(예: `MasterDataManager`) | shared만 |
| `shared` | 도메인 지식 없는 재사용 코드 | (외부 없음; `@/assets`도 불가 → 로고 "종류"만 반환하는 `brand.ts` 패턴) |

- 같은 레이어의 **다른 슬라이스 import 금지**(`default: disallow`). 같은 슬라이스 내부는 상대경로(`../api/x`)로 참조(순환 방지 관례).
- 외부는 반드시 배럴 경로(`@/entities/auth`, `@/shared/ui`)만 사용. `@/shared/ui/Button` 같은 딥 import는 에러.
- 경로 alias는 **`@/*` = `src/*` 하나뿐** (`tsconfig.json paths`, `vite.config.ts resolve.alias`).

#### 슬라이스 목록

| 레이어 | 슬라이스 |
| --- | --- |
| pages (42) | account-business-division, account-management, admin-home, alarm-event, alarm-status-dash, anomaly-score-dash, audit-log, business-management, chamber-status-dash, collector, company-management, comparator, daily-report, dashboard, data-chart, dcop-analysis, diagnosis-analysis-dash, disk-usage, ec-compare, equipment-management, error-log, fdc-analysis, feature-management, line-management, login, master-data, menu-management, model-cfg, model-contract, model-management, model-registry, parameter, performance-dashboard, performance-log, process-management, production-overview, recipe-management, recipe-step-management, role-management, sensor-master, trace-analysis, trace-log |
| widgets (3) | app-shell(`AppShell`), model-version(`ModelStatusBoard`, `ModelVersionPanel`, `ModelDetailPanel`, `TrainPreflight`, `ReplayCompare`, `ModelTrainToolbar`, `EqpGroupFilter`), process-scope(`useSiteScope`, `useProcessScope`, `ProcessScopeToolbar`) |
| features (5) | password-change, report-source-action, target-selector, trace-preset, workspace-tabs |
| entities (34) | account, account-business-division, alarm-event, alarm-status, audit-log, auth, business-division, chamber-status, collector, company, config-c, daily-report, data-chart, disk-usage, ec-compare, error-log, fdc, feature, master-data, menu, model-cfg, model-registry, overview, paramatcher, performance, qa, recipe-master, report, role, sensor-master, target, trace, virtual-data, voc |
| shared | api(`http.ts`), config(`routes.ts`, `brand.ts`, `internalHost.ts`, `manual.ts`, `qa.ts`), lib(i18n, queryKeys, access, theme, resetStores, widgetLayoutStore, pageActivity, 차트 유틸 다수, CRUD 훅), types(`api.ts` 2,086줄 DTO), ui(42개 컴포넌트) |

### 2.3 네이밍 규칙

| 대상 | 규칙 | 예 |
| --- | --- | --- |
| 슬라이스 디렉터리 | kebab-case | `account-management` |
| 페이지 컴포넌트 | `XxxPage` PascalCase, 파일명 동일 | `AccountManagementPage.tsx` |
| API 모듈 | `xxxApi` 객체 export, 파일 `entities/<x>/api/<x>.ts` | `accountApi`, `menuApi` |
| zustand 스토어 | `useXxxStore`, 파일 `model/<x>Store.ts` | `useTargetStore`, `useWorkspaceTabsStore` |
| 쿼리 키 | `queryKeys.<name>(...)` (`shared/lib/queryKeys.ts` 한 곳) | `queryKeys.accounts(companyId)` |
| DTO 타입 | 백엔드 DTO와 **같은 필드명**, `XxxResponse`/`XxxRequest` | `MeResponse`, `AccountCreateRequest` |
| i18n ns | 슬라이스명과 1:1 kebab-case, 파일 `locales/{en,ko}/<ns>.json` | `trace-analysis.json` |
| 페이지 스코프 CSS | `.page-<slice>` 루트 클래스 | `.page-trace`, `.page-anomaly-score-dash` |
| WidgetBoard pageKey | `<slice>-v<N>` — 기본 배치 바꿀 때 N 증가 | `account-management-v2`, `parameter-v9` |
| localStorage 키 | `semi-*` 접두 | `semi-language`, `semi-theme`, `semi-widget-layout` |
| 커밋 | `<type>(<scope>): <JIRA-KEY> <subject>` (`.gitmessage`) | `feat(master-data): RSDSEP-35 ...` |

---

## 3. 설정/환경

### 3.1 환경변수 (`.env.example`, `src/vite-env.d.ts`)

| 변수 | 용도 | 기본값/예 | 사용처 |
| --- | --- | --- | --- |
| `VITE_API_TARGET` | **dev 서버 전용** `/api` 프록시 대상 | `http://localhost:8081` (`.env.example`); 로컬 `.env`에는 `<API_HOST>` | `vite.config.ts server.proxy` |
| `VITE_API_BASE_URL` | 배포 번들에서 same-origin이 아닌 API origin이 필요할 때만 | 비움(= same-origin `/api`) | `src/shared/api/http.ts baseURL` |
| `VITE_BASE_PATH` | Vite `base` (서브 경로 배포) | `/` | `vite.config.ts base`; 런타임은 `import.meta.env.BASE_URL`(관리자 도구 정적 문서 링크) |
| `import.meta.env.DEV` | 개발 모드 분기(리포트 소스 레지스트리 중복 등록 시 throw, 템플릿 탭 디버그) | — | `src/entities/report/model/descriptorRegistry.ts`, `src/pages/daily-report/ui/TemplateTab.tsx` |

- `.env`, `.env.local`, `.env.*.local`은 gitignore. `loadEnv(mode, cwd, '')`로 접두 없이 전부 읽으므로 **프로세스 환경변수가 `.env` 값을 덮는다**(verify 스킬이 이 성질로 격리 vite를 띄움).
- 비밀값은 프론트 환경변수에 없다. CI 변수 `DSEP_SU_LOGIN`/`DSEP_SU_PASSWORD`(masked)는 배포 후 Feature 동기화용 SU 계정이며 번들에 포함되지 않는다.

### 3.2 `vite.config.ts` 핵심

```ts
base: env.VITE_BASE_PATH || '/',
define: { 'process.env': {} },        // react-draggable이 process.env 참조 → 빈 객체로 치환
plugins: [react()],
test: { environment: 'jsdom', setupFiles: ['./src/test/setup.ts'], include: ['src/**/*.test.{ts,tsx}'], css: false },
resolve: { alias: { '@': path.resolve(__dirname, 'src') } },
server: { port: 3000, proxy: { '/api': { target: env.VITE_API_TARGET || 'http://localhost:8081', changeOrigin: true, secure: false } } },
```

- 출력은 Vite 기본(`dist/`, `assets/index-<hash>.js`). 수동 청크 분할(`manualChunks`) 설정은 **없다**. CI는 `index.html`에서 `index-*.js` 파일명을 grep해 배포 로그에 남긴다.
- `public/docs/*.html`이 `dist/docs/`로 그대로 복사된다.

### 3.3 `tsconfig.json` 특이점

- `target ES2020`, `lib [ES2020, DOM, DOM.Iterable]`, `jsx react-jsx`, `module ESNext`, `moduleResolution bundler`, `resolveJsonModule`(로케일 JSON import), `isolatedModules`, `noEmit`, `strict`, `noUnusedLocals`, `noUnusedParameters`, `noFallthroughCasesInSwitch`, `paths { "@/*": ["./src/*"] }`, `include ["src"]`, references → `tsconfig.node.json`(vite.config.ts 전용, `composite`).
- `pnpm build`는 `tsc && vite build` — 타입 에러가 빌드를 막는다.

### 3.4 ESLint / Prettier 요약 (`eslint.config.js`, `.prettierrc`)

| 규칙 | 수준 | 의미 |
| --- | --- | --- |
| `js.configs.recommended` + `tseslint.configs.recommended` | — | 기본 |
| `react-hooks/rules-of-hooks` | error | |
| `react-hooks/exhaustive-deps` | warn | 차트 `theme` 의존성은 오탐이라 줄 단위 disable + 사유 주석 허용 |
| `boundaries/dependencies` | error | FSD 레이어 방향·cross-slice·배럴 우회 금지(§2.2 표) |
| `no-alert` | error | `alert/confirm/prompt`는 Chromium이 전체화면을 해제하므로 금지 → `ConfirmModal`/`useToast` |
| `@typescript-eslint/no-explicit-any` | error | `unknown` + 타입 가드 |
| `@typescript-eslint/consistent-type-imports` | warn (`inline-type-imports`) | `import { type X }` |
| lint 통과 기준 | **경고 0** | `pnpm lint` = `eslint src` |

Prettier: `printWidth 80, tabWidth 2, semi, doubleQuote(singleQuote:false), trailingComma all, arrowParens always, endOfLine lf`. 코드베이스는 문자열에 백틱(`` `...` ``)을 광범위하게 쓴다(Prettier가 보존). 서식 전용 커밋은 `.git-blame-ignore-revs`에 등록.

### 3.5 `contract/` 폴더

FE가 생성해 BE(`semi-spring`) 테스트가 소비하는 **계약 픽스처**다 (`contract/README.md`).

| 픽스처 | 생성 테스트(FE) | 소비 테스트(BE) |
| --- | --- | --- |
| `report-view/save-requests.json` | `src/app/reportSources/saveRequestContract.test.ts` — 등록된 리포트 소스 정의 전부를 `toViewSaveRequest`로 조립 | `ReportViewContractTest` → `ViewDefinitionValidator` |
| `report-view/chart-nodes.json` | `src/pages/daily-report/ui/editor/viewChartContract.test.ts` — 차트 11종의 FE 허용 옵션을 전부 켠 노드 직렬화 | `ReportViewContractTest` → `DailyReportDocumentValidator` |

갱신 절차: `pnpm vitest run -u <두 테스트>` → BE 리포 `src/test/resources/contract/report-view/`에 복사 → BE `./gradlew test --tests '*ReportViewContractTest'` → 두 리포 MR에 함께 커밋. FE CI는 생성 결과가 커밋본과 다르면 실패하고, 로컬에서 옆에 BE 리포가 있으면 그 사본과도 비교한다.

---

## 4. 앱 부트스트랩

### 4.1 엔트리 (`src/app/main.tsx`)

```tsx
initTheme();                 // localStorage semi-theme → <html data-theme>
initI18n();                  // localStorage semi-language(없으면 ko) → i18next.init (동기)
initBrandDocumentTitle();    // hostname → document.title (PRISM | DUTCHBOY SEMI)

const queryClient = new QueryClient({ defaultOptions: { queries: {
  staleTime: 15_000,
  retry: (failureCount, error) => !isClientError(error) && !isTimeoutError(error) && failureCount < 1,
  refetchOnWindowFocus: false,
}}});
const router = createBrowserRouter([{ path: '*', element: <App /> }]);  // 데이터 라우터(useBlocker용)

ReactDOM.createRoot(root).render(
  <QueryClientProvider client={queryClient}>
    <ToastProvider>
      <AuthProvider>
        <RouterProvider router={router} />
      </AuthProvider>
    </ToastProvider>
  </QueryClientProvider>,
);
```

Provider 트리 순서와 이유:

| 순서 | Provider | 이유 |
| --- | --- | --- |
| 0 | 모듈 레벨 `initTheme/initI18n/initBrandDocumentTitle` | 첫 렌더 전에 적용(FOUC·언어 깜빡임 방지). i18n 리소스가 인라인이라 `init`이 동기로 끝난다 |
| 1 | `QueryClientProvider` | `AuthProvider`가 `useQueryClient()`로 401 시 `clear()` 수행 |
| 2 | `ToastProvider` | `AuthProvider`가 세션 만료 토스트에 `useToast()` 사용 |
| 3 | `AuthProvider` (`src/entities/auth/model/useAuth.tsx`) | `/api/me` 부트스트랩, 401 전역 핸들러 등록, 유휴 세션 감시, 창 간 컨텍스트 동기화. Context를 제공하지 않고 zustand `useAuthStore`에만 기록 |
| 4 | `RouterProvider` | `path: '*'` 하나로 `App`에 위임 → `App` 안의 `<Routes>`가 실제 라우팅. 데이터 라우터를 쓰는 이유는 `useBlocker`(미저장 이탈 방지) |
| 5 | (`AppShell` 내부) `MenuProvider` | `/api/me/menus` 결과를 path→menu 맵으로 제공 |
| 6 | (`WorkspaceTabView` 내부) `PageActiveProvider` + `UNSAFE_*Context` | keep-alive 탭별 표시 여부/고정 location 주입 |

- 전역 에러 바운더리(`ErrorBoundary`)는 **없다**. 오류 처리는 ① axios 응답 인터셉터(401 전역), ② React Query 에러 → `QueryState`/`DataTable isError`/토스트, ③ `BackendMenuView`의 메뉴 로드 실패 블록으로 분산된다.
- 초기 데이터 로드: `AuthProvider`의 `authApi.me()` 1회(401이면 `user=null`) → `AppShell`의 `myMenus`(`/api/me/menus`)·`contextOptions`(`/api/me/sites`) 2개 쿼리. `contextOptions` 쿼리 키는 `['contextOptions', accountId]`로 AppShell·AdminSiteRoute·useManagementTarget·useWorkspaceTabs·useActiveContextCodes가 **공유**하므로 요청은 1회만 나간다.

### 4.2 전역 CSS 로드 순서

`main.tsx`가 `./styles/global.css`(안에서 `@import "./tokens.css"` + `@tailwind base/components/utilities`) → `./styles/dashboardPanels.css` 순으로 import한다. 페이지 전용 CSS는 `src/pages/daily-report/ui/reportDocument.css` 하나뿐이고 나머지는 전부 `global.css`(9,890줄)에 `.page-<slice>` 스코프로 들어 있다.

---

## 5. 라우팅

### 5.1 라우트 상수 (`src/shared/config/routes.ts`)

`ROUTES` 객체 하나가 모든 경로의 정본이다. 메뉴 라벨은 `menu` 네임스페이스에 **경로 문자열을 키**로 둔다(`menu.json`의 `"/system/accounts": "Account Management"`).

### 5.2 전체 라우트 표

레이아웃: `AppShell` = 사이드바 + 탑바(탭 스트립) + `WorkspaceTabView`. 보호 = `ProtectedRoutes`(세션 필수). 게이트 열의 의미: `tenantData` = `TenantDataRoute`(사업부 매핑 없으면 안내 블록), `AdminSiteRoute` = ADMIN/ADMIN 컨텍스트가 아니면 `BackendMenuView`로 폴백. 권한 열은 `featureManifest.ts`의 `requiredRoleLevel`(정적 네비게이션은 `STATIC_NAV_ITEMS`와 동일)이며, 실제 접근 판정은 서버가 한다.

| path | 페이지 컴포넌트 | 레이아웃 | 보호 | 게이트 | Feature 코드 / 필요 등급 |
| --- | --- | --- | --- | --- | --- |
| `/login` | `LoginPage` | 없음(`.login-page`) | 비보호(`LoginRoute`: 세션 있으면 `state.from` 또는 `/admin`으로 replace) | — | — |
| `/` | `<Navigate to="/admin" replace>` | AppShell | 보호 | — | — |
| `/admin` | `AdminHomePage` | AppShell | 보호 | 비관리자 컨텍스트면 홈 메뉴로 `<Navigate>` | `ADMIN_HOME` / SU(정적 nav는 USER) |
| `/blank` | `null` | AppShell | 보호 | 탭 등록 대상 아님 | — |
| `/dashboard` | `DashboardPage` | AppShell | 보호 | tenantData | `DASHBOARD` / USER |
| `/dashboard/trace-log` | `TraceLogPage` | AppShell | 보호 | tenantData | (manifest 없음 — DASHBOARD 규칙 `/api/loads/target/**` 공유) |
| `/dashboard/anomaly-score` | `AnomalyScoreDashPage` | AppShell | 보호 | tenantData | `ANOMALY_SCORE_DASH` / USER |
| `/dashboard/diagnosis-analysis` | `DiagnosisAnalysisDashPage` | AppShell | 보호 | tenantData | `DIAGNOSIS_ANALYSIS_DASH` / USER |
| `/dashboard/chamber-status` | `ChamberStatusDashPage` | AppShell | 보호 | tenantData | `CHAMBER_STATUS` / USER |
| `/dashboard/alarm-status` | `AlarmStatusDashPage` | AppShell | 보호 | tenantData | `ALARM_STATUS` / USER |
| `/dashboard/production-overview` | `ProductionOverviewPage` | AppShell | 보호 | tenantData | `PRODUCTION_OVERVIEW` / ADMIN (nav 숨김 `hiddenInNav`) |
| `/sensor/trace-analysis` | `TraceAnalysisPage` | AppShell | 보호 | tenantData | `TRACE_ANALYSIS` / USER |
| `/sensor/data-chart` | `DataChartPage` | AppShell | 보호 | tenantData | `DATA_CHART` / USER |
| `/sensor/fdc-analysis` | `FdcAnalysisPage` | AppShell | 보호 | tenantData | `FDC_ANALYSIS` / USER |
| `/sensor/dcop-analysis` | `DcopAnalysisPage` | AppShell | 보호 | tenantData | `DCOP_ANALYSIS` / USER |
| `/sensor/ec-compare` | `EcComparePage` | AppShell | 보호 | tenantData | `EC_COMPARE` / USER |
| `/sensor/daily-report` | `DailyReportPage` | AppShell | 보호 | tenantData | `DAILY_REPORT` / USER |
| `/sensor/sensor-management` | `SensorMasterPage` | AppShell | 보호 | — | `SENSOR_MANAGEMENT` / USER |
| `/system/performance-logs` | `PerformanceLogPage` | AppShell | 보호 | — | `PERFORMANCE_LOG_VIEW` / USER |
| `/system/performance-dashboard` | `PerformanceDashboardPage` | AppShell | 보호 | — | `PERFORMANCE_DASHBOARD_VIEW` / USER |
| `/system/disk-usage` | `DiskUsagePage` | AppShell | 보호 | tenantData | `DISK_USAGE` / ADMIN |
| `/system/collectors` | `CollectorPage` | AppShell | 보호 | — (관리자/테넌트 양쪽 진입, 내부 분기) | `COLLECTOR` / USER |
| `/system/business-divisions` | `BusinessManagementPage` | AppShell | 보호 | — (양쪽 진입) | `BUSINESS_DIVISION_MANAGE` / ADMIN |
| `/system/companies` | `CompanySiteManagementPage` | AppShell | 보호 | AdminSiteRoute | `COMPANY_MANAGE` / SU |
| `/system/master-data` | `MasterDataPage` | AppShell | 보호 | AdminSiteRoute | `MASTER_DATA_MANAGE` / ADMIN (nav 숨김) |
| `/system/processes` | `ProcessInfoManagementPage` | AppShell | 보호 | AdminSiteRoute | `PROCESS_INFO_MANAGE` / ADMIN |
| `/system/lines` | `LineManagementPage` | AppShell | 보호 | AdminSiteRoute | `LINE_MANAGE` / ADMIN |
| `/system/equipment` | `EquipmentManagementPage` | AppShell | 보호 | AdminSiteRoute | `EQUIPMENT_MANAGE` / ADMIN |
| `/system/accounts` | `AccountManagementPage` | AppShell | 보호 | AdminSiteRoute | `ACCOUNT_MANAGE` / ADMIN |
| `/system/account-divisions` | `AccountBusinessDivisionPage` | AppShell | 보호 | AdminSiteRoute | `ACCOUNT_BUSINESS_DIVISION_MANAGE` / ADMIN |
| `/system/menus` | `MenuManagementPage` | AppShell | 보호 | AdminSiteRoute | `MENU_MANAGE` / ADMIN |
| `/system/roles` | `RoleManagementPage` | AppShell | 보호 | AdminSiteRoute | `ROLE_MANAGE` / ADMIN |
| `/system/features` | `FeatureManagementPage` | AppShell | 보호 | AdminSiteRoute | `FEATURE_MANAGE` / SU |
| `/system/audit-logs` | `AuditLogManagementPage` | AppShell | 보호 | AdminSiteRoute; 메뉴 디스패치 시 `canAccessRoleLevel(user,'ADMIN')` 추가 검사 | `AUDIT_LOG_VIEW` / ADMIN |
| `/system/error-logs` | `ErrorLogPage` | AppShell | 보호 | AdminSiteRoute | `ERROR_LOG_VIEW` / SU |
| `*` (명시 라우트 없음) | `BackendMenuView` | AppShell | 보호 | 메뉴 트리에서 `urlPath === pathname`인 MENU 노드를 찾아 `canAccess`면 featureCode로 페이지 선택 | 아래 표 |

**명시 라우트 없이 메뉴 디스패치로만 열리는 화면** (`src/app/routes/BackendMenuView.tsx` `FEATURE_CODE_BY_PATH` + `PAGE_BY_FEATURE_CODE`):

| path | Feature 코드 | 페이지 | 등급 | 비고 |
| --- | --- | --- | --- | --- |
| `/system/model-cfg` | `MODEL_CFG` | `ModelCfgPage` | USER | 모델 생성 파라미터 |
| `/system/model-registry` | `MODEL_REGISTRY` | `ModelRegistryPage` | USER | |
| `/system/model-contracts` | `MODEL_CONTRACT` | `ModelContractPage` | ADMIN | |
| `/sensor/model-management` | `MODEL_MANAGE` | `ModelManagementPage` | USER | 생성 실행·운영 전환 |
| `/sensor/recipe-management` | `RECIPE_MANAGEMENT` | `RecipeManagementPage` | USER | |
| `/sensor/recipe-step-management` | `RECIPE_STEP_MANAGEMENT` | `RecipeStepManagementPage` | USER | |
| `/sensor/parameter` | `PARAMETER` | `ParameterPage` | USER | ParaMatcher TTTM 비교 |
| `/sensor/comparator` | `COMPARATOR` | `ComparatorPage` | USER | 설정 백업(.bak) 비교 |
| `/sensor/alarm-event` | `ALARM_EVENT` | `AlarmEventPage` | USER | `TenantDataRoute`로 감싸서 렌더 |

`PAGE_BY_FEATURE_CODE`에는 위 외에도 관리자 콘솔 화면 전부(`COMPANY_MANAGE` … `ERROR_LOG_VIEW`, `COLLECTOR`, `BUSINESS_DIVISION_MANAGE`, 대시보드 4종, `DAILY_REPORT`, `SENSOR_MANAGEMENT`)가 등록돼 있어, **테넌트 `tb_co_menu`에 행만 만들면 어떤 화면이든 그 컨텍스트에서 렌더된다**(MENU-GUIDE "테넌트 메뉴에 두면 안 되는 화면" 경고의 근거). 코드 매핑에 없는 featureCode는 "오픈 예정"(`menuView.comingSoon`) 블록.

### 5.3 가드 구현 (`src/app/routes/AppRoutes.tsx`)

```tsx
function ProtectedRoutes() {
  const { user, isBootstrapping, sessionEndReason } = useAuth();
  const location = useLocation();
  if (isBootstrapping) return <LoadingBlock label={t('loading.session')} />;
  if (!user) {
    const rememberFrom = sessionEndReason !== 'logout';   // 만료·딥링크만 복귀 위치 기억
    return <Navigate to={ROUTES.login} replace
      state={rememberFrom ? { from: `${location.pathname}${location.search}` } : undefined} />;
  }
  if (isPasswordChangeOnly(user)) return <PasswordChangeGate />;   // 비밀번호 변경 전용
  return <AppShell />;                                             // 하위는 WorkspaceTabView가 렌더
}

function AdminSiteRoute({ children }) {
  const contextOptionsQuery = useQuery({ queryKey: queryKeys.contextOptions(user?.accountId), ... });
  if (contextOptionsQuery.isLoading) return <LoadingBlock .../>;
  if (!isAdminSiteContext(user, contextOptionsQuery.data ?? [])) return <BackendMenuView />; // 차단이 아니라 폴백
  return <>{children}</>;
}

function readReturnPath(state: unknown): string | null {   // '/'로 시작하고 '//'가 아닌 내부 경로만 허용
  ...
}
```

- `isAdminSiteContext`: 활성 법인 코드와 활성 사이트 코드가 둘 다 `ADMIN`일 때 true (`src/shared/lib/access.ts`).
- `isPasswordChangeOnly`: `passwordChangeRequired || authState === 'CHANGE_PASSWORD_ONLY'`.
- `TenantDataRoute`: `hasBusinessDivisionAccess(user)` = SU이거나 `businessDivisions.length > 0`. 아니면 `EmptyBlock`(businessDivisionGate) 안내. **화면 안내일 뿐, 실제 방어는 서버**.

### 5.4 lazy 로딩 / 404

- **코드 스플리팅 없음**: `AppRoutes.tsx`와 `BackendMenuView.tsx`가 모든 페이지를 정적 import한다. `React.lazy`/`import()` 사용처는 없고(`import.meta.glob`은 테스트 전용 `allDescriptors.ts`), 단일 번들로 배포된다.
- 404 전용 페이지 없음. 모르는 경로는 `*` → `BackendMenuView`가 메뉴 트리에서 못 찾으면 `EmptyBlock(menuView.notFound)`("메뉴를 찾을 수 없습니다")를 `Page > Panel` 안에 그린다. 메뉴 로드 403이면 `AccessDeniedPage`.

### 5.5 탭 시스템과 라우트의 관계 (`src/features/workspace-tabs`, 설계 `docs/workspace-tabs-architecture.md`)

- `AppShell`은 `<Outlet/>` 대신 **`WorkspaceTabView`** 를 렌더한다. 이것이 `useOutlet()`으로 현재 라우트 요소를 잡아 **경로별 캐시(Map)** 에 넣고, 활성 경로만 보이게(`hidden` 속성) 하며 나머지는 마운트를 유지한다.
- 탭 = `{ id: '${companyId}:${siteId}:${path}', path, companyId, siteId, companyName, siteName, label }`. 같은 경로라도 컨텍스트가 다르면 별개 탭.
- 탭 등록은 **경로 변경 시 자동**(`useWorkspaceTabs`의 effect): 메뉴 트리에 있는 경로(`menuAtPath.companyId === activeCompanyId`) 또는 관리자 컨텍스트의 `STATIC_NAV_ITEMS` 경로만. `/blank`, 404, 리다이렉트 경유지는 등록되지 않는다. 컨텍스트 키(`accountId:companyId:siteId`)가 직전 사이클과 다르면 그 사이클은 건너뛴다(ADR-6).
- 탭 클릭: 같은 컨텍스트면 `navigate(path)`; 다르면 `auth.switchContext({companyId, siteId})` 성공 후 `navigate`. `hasUnsavedChanges()`면 `ConfirmModal` 먼저.
- 탭 닫기: 활성 탭이면 왼쪽 이웃 → 오른쪽 이웃 → `/blank`. 모든 창 닫기 → `/blank`.
- keep-alive 핵심(`WorkspaceTabView.tsx`, 핵심 발췌):

```tsx
const MAX_KEEPALIVE_ENTRIES = 5;   // 12 → 5 (2026-08-20 OOM 감사)
const cacheRef = useRef(new Map<string, CacheEntry>());        // 최근 사용순(LRU) Map
const contextKey = `${accountId}:${activeCompanyId}:${activeSiteId}`;
if (prevContextKeyRef.current !== contextKey) { prevContextKeyRef.current = contextKey; cacheRef.current.clear(); }
// 컨텍스트 전환 커밋 프레임에는 페이지 영역을 비운다(suspended) → queryClient.clear()가 "구독 0" 상태에서 실행
const suspended = resumedContextKey !== contextKey;
// 열린 탭 경로만 캐시에 남긴다
for (const key of cacheRef.current.keys()) if (key !== pathname && !openPaths.has(key)) cacheRef.current.delete(key);
// 캐시된 탭으로 복귀 시 stale 활성 쿼리만 재조회(cancelRefetch:false — queryFn이 AbortSignal을 안 받아 취소가 네트워크를 못 끊음)
useEffect(() => { if (returningToCachedRef.current) void queryClient.refetchQueries({ type: 'active', stale: true }, { cancelRefetch: false }); }, [pathname]);
if (!suspended) { cacheRef.current.delete(pathname); cacheRef.current.set(pathname, { element: outlet, locationContext }); }
while (cacheRef.current.size > MAX_KEEPALIVE_ENTRIES) cacheRef.current.delete(cacheRef.current.keys().next().value);
useEffect(() => { requestAnimationFrame(() => window.dispatchEvent(new Event('resize'))); }, [pathname]); // RGL·echarts 재측정
return <>{[...cacheRef.current.entries()].map(([path, entry]) => (
  <div className="page-keepalive" hidden={path !== pathname} key={`${contextKey}:${path}`}>
    <UNSAFE_NavigationContext.Provider value={isActive ? navigation : frozenNavigation}>   {/* 숨김 탭은 no-op navigator */}
      <UNSAFE_LocationContext.Provider value={isActive ? liveLocationContext : entry.locationContext}> {/* 숨김 탭은 캡처 시점 location 고정 */}
        <PageActiveProvider value={isActive}>{entry.element}</PageActiveProvider>
      </UNSAFE_LocationContext.Provider>
    </UNSAFE_NavigationContext.Provider>
  </div>))}</>;
```

- 숨김 탭의 쿼리 구독은 `usePageActive()`(`src/shared/lib/pageActivity.ts`)를 `useQuery({ subscribed })`에 넣어 끊는다 — 구독이 남으면 `gcTime`이 흐르지 않아 대형 응답이 영구 상주한다(§10.4).
- 탭 목록은 `sessionStorage` `semi-workspace-tabs`에 **계정별 파티션**(`byAccount[accountId]`)으로 persist(창별 격리, 재로그인 유지, 로그아웃 리셋 제외). 과거 localStorage 키는 모듈 로드 시 삭제.

---

## 6. 인증/로그인

### 6.1 전제: 세션 + CSRF, 토큰 없음

- 백엔드는 **Spring Security 세션** 기반. 프론트는 access token을 저장하지 않는다(브라우저 저장소에 토큰 0개). 세션 쿠키는 `withCredentials: true`로 자동 전송.
- CSRF: axios 내장 XSRF 지원(`withXSRFToken: true, xsrfCookieName: 'XSRF-TOKEN', xsrfHeaderName: 'X-XSRF-TOKEN'`)으로 **쿠키 값을 헤더로 되돌린다**. 첫 요청(`GET /api/me`)이 쿠키를 받아오므로 로그인 POST 전에 별도 토큰 요청이 필요 없다(CI 스크립트 `feature-sync.sh`도 같은 순서: `GET /api/me`로 쿠키 → `POST /api/auth/login`).
- 인증 상태의 기준은 `MeResponse`(`/api/me` 또는 로그인 응답)이며 zustand `useAuthStore.user`에 보관한다.

### 6.2 로그인 화면 (`src/pages/login/ui/LoginPage.tsx`)

| 항목 | 내용 |
| --- | --- |
| 레이아웃 | 전체 배경 이미지 + 스크림, 중앙 카드(`.login-card`, 760px, 2열): 왼쪽 브랜드 패널(심볼, `brand.loginHeadline`, `brand.loginSubtitle`), 오른쪽 폼 패널(로고 + 폼) |
| 브랜드 | `currentBrandMarks()`(`src/shared/config/brand.ts`)가 `window.location.hostname` **정확 일치**로 결정. 기본 PRISM; 검증 도메인(`<STAGING_HOST>`)과 로컬(`localhost`, `127.0.0.1`, `[::1]`)은 DUTCHBOY SEMI. 로고 파일은 FSD 경계 때문에 페이지가 종류별로 import |
| 필드 | `loginId`(`autoComplete=username`, autoFocus), `password`(`type=password`, `autoComplete=current-password`) — `Field` + `.control` |
| 검증 | 둘 중 하나라도 비면 `page.error.missingCredentials` 인라인 `.form-error` (서버 호출 없음) |
| 제출 | `auth.login({loginId, password})`. 버튼 `disabled={isSubmitting}`, 라벨 `page.submitting`/`page.submit` |
| 실패 | `getApiErrorMessage(error, t('page.error.loginFailed'))` — 서버 `message` 우선, 폼 아래 표시. 토스트 없음 |
| 전체화면 UX | submit 제스처 안에서 `document.documentElement.requestFullscreen()`을 **먼저** 호출(await 뒤로 미루면 제스처 소비로 거부). 이미 전체화면(`isAnyFullscreen()`, F11 포함)이면 요청하지 않음. 로그인 실패 시 이번에 요청했고 실제 전체화면이면 `exitFullscreen()`으로 되돌림 |
| 성공 후 이동 | 페이지는 navigate하지 않는다. `user`가 채워지면 `LoginRoute`가 `state.from`(내부 경로만) → `/admin` 순으로 replace. 지난 세션의 마지막 화면은 **복원하지 않음**(컨텍스트·권한을 다시 받는 자리이므로) |

### 6.3 API 호출과 응답 처리 (`src/entities/auth/api/auth.ts`)

| 메서드 | HTTP | 응답 | 비고 |
| --- | --- | --- | --- |
| `login(req)` | `POST /api/auth/login` `{loginId, password}` | `MeResponse` | |
| `logout()` | `POST /api/auth/logout` (본문 없음) | void | |
| `me()` | `GET /api/me` | `MeResponse` | 부트스트랩·세션 확인 |
| `changePassword(req)` | `PATCH /api/me/password` `{currentPassword, newPassword}` | `MeResponse` | |
| `contextOptions()` | `GET /api/me/sites` | `CompanyOptionResponse[]` (`{id, code, name, active, sites:[{id, code, name, active, timeZone}]}`) | 코드·타임존 해석의 유일한 출처 |
| `switchContext(req)` | `PATCH /api/me/active-context` `{companyId, siteId}` | `MeResponse` | |

`MeResponse` (`src/shared/types/api.ts`):

```ts
interface MeResponse {
  accountId: number; accountType: 'GLOBAL'|'COMPANY'|'SITE'; loginId: string; accountName: string;
  activeCompanyId: number|null; activeCompanyName: string|null; activeSiteId: number|null; activeSiteName: string|null;
  roleLevels: ('USER'|'ADMIN'|'SU')[]; authState: 'NORMAL'|'CHANGE_PASSWORD_ONLY'; passwordChangeRequired: boolean;
  roles: { roleId; roleName; roleLevel; companyId|null; siteId|null }[];
  businessDivisions?: { id; code; name }[];   // 활성 사이트에서의 데이터 접근 권한(구서버엔 없음)
}
```

### 6.4 HTTP 클라이언트 인터셉터 (`src/shared/api/http.ts`, 핵심 발췌)

```ts
export const http = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || '', timeout: 30000,
  withCredentials: true, withXSRFToken: true,
  xsrfCookieName: 'XSRF-TOKEN', xsrfHeaderName: 'X-XSRF-TOKEN',
});
let unauthorizedHandler: (() => void) | null = null;          // shared는 entities/auth를 import 못 함 → 레지스트리 주입
export function setUnauthorizedHandler(h) { unauthorizedHandler = h; }
let lastServerContactMs = Date.now();                          // 유휴 세션 감시용 "서버 마지막 응답 시각"
export function getLastServerContactMs() { return lastServerContactMs; }

http.interceptors.response.use(
  (response) => { lastServerContactMs = Date.now(); return response; },
  (error: unknown) => {
    if (axios.isAxiosError(error) && error.response) lastServerContactMs = Date.now(); // 4xx도 세션 타이머를 갱신하므로 기록
    if (isUnauthorized(error)) unauthorizedHandler?.();       // 401 → 등록된 전역 핸들러
    return Promise.reject(error);
  },
);
function buildConfig(options?: RequestOptions): AxiosRequestConfig {
  const headers: Record<string, string> = {};
  if (options?.menuId != null) headers['X-Menu-Id'] = String(options.menuId);   // 명시적으로 넘길 때만
  return { headers, signal: options?.signal, timeout: options?.timeoutMs };
}
```

- 요청 인터셉터 **없음**(헤더 주입은 `buildConfig`로 호출부가 명시). 토큰 갱신(refresh) **없음**(세션). 재시도는 axios가 아니라 React Query `retry`(5xx·네트워크만 1회).
- 401 핸들러(`AuthProvider`)의 판정: **현재 `user`가 있을 때만** 만료로 처리 — `sessionEndReason='expired'` → `setUser(null)` → `queryClient.clear()` → `resetClientStores()` → `toast.error(login:session.expiredToast)`. 로그인 실패·부트스트랩 401은 `user`가 없어 무시되고 호출부 에러 처리로 흐른다.
- 403은 전역 처리하지 않고 호출부가 `isForbidden(error)`로 `AccessDeniedPage`/토스트 등 결정.

### 6.5 세션/상태 저장 위치

| 데이터 | 저장소 | 키 | 로그아웃 시 |
| --- | --- | --- | --- |
| 세션 사용자 `MeResponse` | zustand `useAuthStore` (메모리) | — | `null` |
| 서버 세션·CSRF | 브라우저 쿠키(백엔드 발급) | `JSESSIONID`(추정) · `XSRF-TOKEN` | 서버 `logout`이 무효화 |
| 계정별 마지막 활성 컨텍스트 | localStorage | `semi.lastSession.<accountId>` → `{companyId, siteId}` | 유지(개인화) |
| 워크스페이스 탭 | sessionStorage | `semi-workspace-tabs` (`byAccount`) | 유지 |
| 테마 / 언어 / 위젯 배치 | localStorage | `semi-theme` / `semi-language` / `semi-widget-layout` | 유지 |

### 6.6 로그아웃 (`useAuth().logout`)

1. `setSessionEndReason('logout')` (재로그인 시 위치 복원 안 함)
2. `setUser(null)` **먼저** — 만료된 세션의 logout 401이 전역 만료 토스트로 이어지지 않게
3. `POST /api/auth/logout` (실패해도 흐름을 막지 않음; 401은 무시, 그 외 warn)
4. `finally`: `queryClient.clear()` + `resetClientStores()`(등록된 모든 클라이언트 store reset) + 페이지 전체화면이면 `exitFullscreen()`
5. `ProtectedRoutes`가 `user=null`을 보고 `/login`으로 이동 (state 없음)

리셋 범위: `registerStoreReset`에 등록된 store(관리 대상 선택, 각 페이지 필터/선택/뷰 store, 모델 학습 컨트롤, 리포트 재캡처) — 테마·언어·위젯 배치·탭 목록·마지막 컨텍스트는 **제외**.

### 6.7 세션 만료 UX (`AuthProvider`)

| 트리거 | 동작 |
| --- | --- |
| 임의 API 401 (세션 있을 때) | 위 6.4 핸들러 → 로그인 화면으로, 토스트 "세션 만료", `state.from`에 현재 위치 기억 → 재로그인 후 복귀 |
| 탭 포커스/표시 복귀 | 직전 서버 통신이 30초(`FOCUS_CHECK_MIN_GAP_MS`) 이전이면 `GET /api/me`로 즉시 확인. 응답이 현재 user와 다르면(역할/컨텍스트 변경) `setUser` 갱신 |
| 방치 | 60초마다 "서버 무통신 시간 > 3시간 1분(`SESSION_IDLE_TIMEOUT_MS`, 서버 timeout 3h보다 약간 김)"이면 `GET /api/me` — 타임아웃 전에 호출하면 유휴 세션을 연장해 버리므로 넘긴 뒤에만 |
| 다른 창의 컨텍스트 전환 | `BroadcastChannel('semi-auth-context')` 메시지 수신 → 같은 계정이고 컨텍스트가 다르면 `applyContextSwitch` + 토스트 `contextSyncedToast` |

### 6.8 내 정보/권한 로드 시점과 컨텍스트 전환

- 부트스트랩: `AuthProvider` 마운트 시 `/api/me` 1회 → `isBootstrapping=false`. 그동안 `ProtectedRoutes`/`LoginRoute`는 `LoadingBlock`.
- 로그인 직후: `restoreLastContext(user)` — `semi.lastSession.<accountId>`가 있고 현재와 다르면 `switchContext` 호출 후 그 결과를 `setUser`(기본 컨텍스트로 잠깐 렌더되는 깜빡임 방지). 실패하면 저장값 삭제 후 진행.
- `applyContextSwitch(nextUser)`: `flushSync(() => { setUser(nextUser); resetClientStores(); })` → `queryClient.clear()` 순서가 중요(keep-alive 화면을 먼저 언마운트해 구독 0 상태에서 캐시를 비움). 이후 `BroadcastChannel`로 다른 창에 알림.
- 권한 판정 헬퍼(`src/shared/lib/access.ts`): `highestRoleLevel`, `hasRoleLevel(user, lvl)`, `canAccessRoleLevel(user, required)`(USER 1 < ADMIN 2 < SU 3), `canManageCompanies`(SU), `isPasswordChangeOnly`, `hasActiveSite`, `hasBusinessDivisionAccess`, `isSiteBoundAdmin`(SU 아니고 `accountType==='SITE'`), `isAdminContextCode`, `isAdminSiteContext`.
- 비밀번호 변경 강제: `isPasswordChangeOnly(user)`면 `ProtectedRoutes`가 `PasswordChangeGate`(`PasswordChangeModal mode=required`, 로그아웃 버튼만)를 렌더하고 메뉴/컨텍스트 쿼리는 `enabled=false`.

### 6.9 로그인 시퀀스

```mermaid
sequenceDiagram
  participant B as Browser(LoginPage)
  participant A as useAuth/AuthProvider
  participant H as axios http
  participant S as semi-spring

  Note over A,S: 앱 시작: AuthProvider가 GET /api/me (401이면 user=null, XSRF-TOKEN 쿠키 수신)
  B->>B: submit 제스처 안에서 requestFullscreen()
  B->>A: login({loginId, password})
  A->>H: POST /api/auth/login (X-XSRF-TOKEN 헤더 자동)
  H->>S: 요청 (세션 쿠키 생성)
  S-->>H: 200 MeResponse
  A->>A: readLastContext(accountId) → 있고 다르면
  A->>S: PATCH /api/me/active-context {companyId, siteId}
  S-->>A: MeResponse(복원된 컨텍스트) / 실패 시 clearLastContext
  A->>A: setSessionEndReason(null); setUser(next); invalidateQueries()
  Note over B: LoginRoute: user 감지 → Navigate(state.from ?? /admin)
  B->>S: GET /api/me/menus, GET /api/me/sites (AppShell)
  Note over B: isPasswordChangeOnly(user)면 AppShell 대신 PasswordChangeGate
  alt 로그인 실패
    S-->>H: 4xx {message}
    H-->>A: reject (user 없음 → 401 핸들러 무시)
    A-->>B: getApiErrorMessage → .form-error, exitFullscreen()
  end
```

---

## 7. 사용자(계정) 관리 화면

경로 `/system/accounts`, 컴포넌트 `AccountManagementPage` (`src/pages/account-management/ui/AccountManagementPage.tsx`, 738줄). Feature `ACCOUNT_MANAGE`(ADMIN). 관리자 콘솔 전용(`AdminSiteRoute`).

### 7.1 골격

- `Page` > (`companyId` 없으면 `PageHeader` + 안내/로딩 `Panel`) 또는 `WidgetBoard pageKey="account-management-v2"`.
- 헤더 바(`actions`): `.board-filter-toolbar` = `ManagementTargetSelector`(법인·사이트, SU+관리자 컨텍스트에서만 표시) + 오른쪽 끝 `HelpButton`(`ManualContent` 4절: 생성 절차·목록·상세·초기화) + `VocButton screenKey="account-management"`.
- 위젯 2개(둘 다 `fullHeight`): `list`(x0 w5) / `detail`(x5 w7). 제목은 창 크롬이 소유하고 `renderWidgetTitleMeta`로 목록엔 건수 `Badge tone=info`, 상세엔 모드/선택 loginId + dirty `•`.
- 대상 해석 중(`target.isResolving`)엔 `LoadingBlock(common:state.resolvingTarget)`, 해석 끝났는데 미선택이면 `EmptyBlock`(에러 아님).

### 7.2 목록 (`DataTable`)

| 컬럼 key | 헤더 i18n | render | filterValue(헤더 정렬/필터 기준) |
| --- | --- | --- | --- |
| `loginId` | `system:account.loginIdHeader` | 그대로 | loginId |
| `accountName` | `system:account.accountNameHeader` | 그대로 | accountName |
| `accountType` | `system:account.accountTypeHeader` | `typeSite`/`typeCompany` 라벨(GLOBAL은 원문) | 라벨 |
| `status` | `system:account.statusHeader` | `Badge`: ACTIVE→`success`/`common:state.active`, LOCKED→`warning`/`statusLocked`, DISABLED→`default`/`common:state.inactive` | 라벨 |

- 데이터: `GET /api/companies/{companyId}/accounts` (`queryKeys.accounts(companyId)`), **서버 페이징 없음**(전량). 사이트 범위 필터는 클라이언트: `siteId`가 선택돼 있으면 `account.siteId == null || account.siteId === siteId`만 표시(2026-09-29 규칙).
- 검색은 `DataTable` 헤더 캐럿(정렬/체크박스·검색)으로만. 별도 검색창 없음.
- 첫 행 자동 선택 `useAutoSelectFirstRow({ rows, isFetching, selectedId, enabled: mode==='edit' && companyId, onSelect, onClear })`.
- 같은 행 재클릭 무시(편집 중 draft 롤백 방지). 다른 행 클릭·신규는 dirty면 `ConfirmModal(common:confirm.discard, danger)` 뒤 실행(`confirmLeave`).

### 7.3 생성/수정 폼 (`detail` 위젯, `PanelBody.account-detail-body` + `.form-grid`)

| 필드 | 컨트롤 | 규칙 |
| --- | --- | --- |
| 계정 유형 `accountType` | `select` COMPANY/SITE | 수정 모드 비활성. 사이트 관리자(`isSiteBoundAdmin`)는 COMPANY 선택 불가(기본 SITE). 사이트 미선택이면 SITE 불가. 바꾸면 `siteId`(SITE면 현재 siteId) 재설정 + 역할 선택 초기화 |
| 로그인 ID `loginId` | input, required | 수정 모드 비활성, 저장 시 trim |
| 이름 `accountName` | input, required | trim |
| 이메일/전화/팩스 | input | `nullableText`(공백 → null) |
| 역할 `roleIds` | 체크박스 목록(`Field group`) | 선택지 = `GET /api/companies/{companyId}/roles?siteId=` 중 COMPANY 계정이면 `!role.siteId`, SITE 계정이면 `!role.siteId || role.siteId === draft.siteId`. 라벨 `roleName / roleLevel [사이트 접미사]`. 비면 `noAssignableRoles` 안내 |
| 안내 | `.detail-note` `resetNote` | 비밀번호 초기화 설명 |

- 클라이언트 검증: `companyId` 필수, SITE 계정은 `siteId` 필수(아니면 throw → 토스트). 그 외 형식 검증은 서버(400 message를 토스트).
- dirty 판정: `JSON.stringify(draft)` vs `baseline`(roleIds는 정렬 배열로 직렬화). 저장 버튼 `disabled={!isDirty || pending || (edit && !selected)}`, title `common:hint.nothingToSave`.
- **저장 2단계**: 수정 모드는 `PATCH .../accounts/{id}` 성공 후 `PUT .../accounts/{id}/roles`. 2단계 실패는 `RoleAssignError(updatedAccount, reason)`로 throw → onError에서 selectedId 유지 + 목록 invalidate + 토스트 `partialRoleAssignError`("정보는 저장됐지만 역할 배정 실패"). 생성 모드는 `POST`에 `roleIds` 포함(1회).
- 성공 시: 토스트 `saveSuccess`, `selectedId=saved.id`, `mode='edit'`, draft/baseline 재동기화, `invalidateQueries(accounts)`.

### 7.4 삭제 / 비활성화 / 잠금 / 비밀번호 초기화

| 액션 | 위치 | 확인 | API |
| --- | --- | --- | --- |
| 삭제 | 상세 제목줄 `Button ghost sm` | `ConfirmModal danger` `deleteConfirm` | `DELETE /api/companies/{c}/accounts/{id}` → 선택 해제, 목록 invalidate |
| 비밀번호 초기화 | 상세 제목줄 `Button sm` | `ConfirmModal` (`resetConfirmTitle` + `resetNote`) | `PATCH .../password-reset` → 응답 계정의 `passwordChangeRequired=true` (다음 로그인 때 변경 강제) |
| 잠금/해제 | API 모듈에는 `lock(reason)`/`unlock` (`PATCH .../lock`, `.../unlock`)이 있으나 **이 화면에는 버튼이 없다**(2026-10 기준). 상태는 Badge로만 표시 | — | — |
| 비활성화(DISABLED) | 화면 토글 없음. `AccountUpdateRequest`에 `active` 필드가 없어 상태 변경은 서버 측(잠금/삭제)에서만 | — | — |
| 신규 | 목록 제목줄 `Button secondary sm` `newAccountButton` | dirty면 confirm | `mode='create'`, `createEmptyDraft(siteId, isSiteAdmin)` |

### 7.5 역할/Feature 지정 UI

- 계정 ↔ 역할: 위 체크박스 목록(다중). 역할 ↔ 메뉴 권한은 **역할 관리 화면**(`/system/roles`)에서, 메뉴 ↔ feature는 **메뉴 관리 화면**(`/system/menus`)에서 지정한다(§8).
- 계정 ↔ 사업부(데이터 접근 권한)는 별도 화면 `/system/account-divisions`(`AccountBusinessDivisionPage`): 사이트의 계정별 사업부 체크 → `PUT /api/companies/{c}/sites/{s}/accounts/{id}/business-divisions`.

### 7.6 API 매핑 표 (`src/entities/account/api/accounts.ts`, 모든 호출에 `{ menuId }` 전달)

| 함수 | 메서드/경로 | 요청 | 응답 |
| --- | --- | --- | --- |
| `list(companyId)` | `GET /api/companies/{c}/accounts` | — | `AccountResponse[]` |
| `create(companyId, req)` | `POST /api/companies/{c}/accounts` | `{accountType:'COMPANY'|'SITE', siteId?, loginId, accountName, email?, phoneNo?, faxNo?, roleIds[]}` | `AccountResponse` |
| `update(c, id, req)` | `PATCH /api/companies/{c}/accounts/{id}` | `{accountName, email?, phoneNo?, faxNo?}` | `AccountResponse` |
| `updateRoles(c, id, req)` | `PUT /api/companies/{c}/accounts/{id}/roles` | `{roleIds[]}` | `AccountResponse` |
| `resetPassword(c, id)` | `PATCH .../password-reset` `{}` | — | `AccountResponse` |
| `lock(c, id, {reason})` | `PATCH .../lock` | `{reason}` | `AccountResponse` |
| `unlock(c, id)` | `PATCH .../unlock` `{}` | — | `AccountResponse` |
| `delete(c, id)` | `DELETE .../accounts/{id}` | — | `AccountResponse` |

`AccountResponse`: `{ id, accountType, companyId|null, siteId|null, loginId, accountName, email?, phoneNo?, faxNo?, status:'ACTIVE'|'LOCKED'|'DISABLED', passwordChangeRequired, securityVersion, roles: RoleSummaryResponse[] }`.

### 7.7 관련 파일

- `src/pages/account-management/ui/AccountManagementPage.tsx` (화면)
- `src/entities/account/api/accounts.ts`, `src/entities/role/api/roles.ts` (API)
- `src/features/target-selector/*` (법인/사이트 선택), `src/entities/menu/model/MenuContext.tsx` (`useMenuId(ROUTES.accounts)`)
- `src/shared/lib/useAutoSelectFirstRow.ts`, `src/shared/lib/text.ts`(`nullableText`)
- 로케일 `system.json`의 `account.*` 키 (`src/shared/lib/i18n/locales/{en,ko}/system.json`)

---

## 8. 메뉴/권한

### 8.1 개념 (MENU-GUIDE.md + 코드)

| 개념 | 정의 | 프론트 표현 |
| --- | --- | --- |
| Feature | 화면 1개 + 그 화면이 호출할 API rule 묶음. 코드(`featureCode`)·기본 경로·필요 등급 | `src/entities/feature/lib/featureManifest.ts`의 `getFeatureManifest()`(i18n 지연 평가라 함수) → `/api/features/sync` |
| MenuNode | 법인/사이트별 메뉴 트리 노드. `FOLDER` 또는 `MENU`(feature + urlPath 필수) | `MenuNodeResponse` |
| RoleMenuPermission | 역할별 메뉴 `canView`(사이드바 표시)·`canAccess`(진입) | `RoleMenuPermissionResponse` |
| SU | 메뉴 권한과 무관하게 전부 접근 | `hasRoleLevel(user,'SU')` |
| 메뉴 소스 2개 | **관리자 콘솔(ADMIN/ADMIN)**: `STATIC_NAV_ITEMS` 배열이 정본(DB 안 읽음) / **테넌트**: `tb_co_menu`(DB)가 정본 | `AppShell`이 `isAdminContext`로 분기 |

### 8.2 메뉴 트리 fetch → 사이드바 렌더링 흐름 (`src/widgets/app-shell/ui/AppShell.tsx`)

```mermaid
flowchart LR
  A[useQuery myMenus\nGET /api/me/menus] --> B[filterTenantMenusByRole\n(비ADMIN이면 /system/audit-logs 제거)]
  B --> C[MenuProvider\nbuildMenuPathMaps / findHomeMenu]
  C --> D{isAdminSiteContext?}
  D -- yes --> E[staticSidebarNodes\nLISTED_STATIC_NAV_ITEMS × canAccessRoleLevel\n→ group 폴더(static:group:*)]
  D -- no --> F[toBackendSidebarNodes\nFOLDER: canView && 자식>0\nMENU: urlPath && canView && canAccess]
  E --> G[SidebarTree]
  F --> G
  G --> H[NavLink to=path / 폴더 토글\nisActiveBranch면 강제 펼침]
```

메뉴 응답 JSON → 사이드바 노드 변환(핵심 발췌, `toBackendSidebarNode`):

```tsx
function toBackendSidebarNode(menu: MenuNodeResponse, namespace: string, t): SidebarNavNode | null {
  const children = menu.children.map((c) => toBackendSidebarNode(c, namespace, t)).filter(isSidebarNavNode);
  if (menu.nodeType === 'FOLDER') {
    if (!menu.canView || children.length === 0) return null;            // 빈 폴더·숨김 폴더 제거
    return { id: `${namespace}:folder:${menu.id}`, label: translateMenuName(t, null, menu), path: null, children };
  }
  if (!menu.urlPath || !menu.canView || !menu.canAccess) return null;   // canView만 있고 canAccess 없으면 숨김
  return { id: `${namespace}:menu:${menu.id}`, label: translateMenuName(t, menu.urlPath, menu), path: menu.urlPath, children: [] };
}
// namespace = `context:${activeCompanyId}:${activeSiteId}` — 컨텍스트가 바뀌면 노드 id가 달라져 펼침 상태가 섞이지 않는다
```

`MenuNodeResponse` (`src/shared/types/api.ts`):

```json
{ "id": 12, "companyId": 3, "siteId": 7, "parentId": 5, "featureId": 21,
  "nodeType": "MENU", "menuName": "트레이스 분석", "menuNameEn": "Trace Analysis",
  "urlPath": "/sensor/trace-analysis", "displayOrder": 2, "active": true, "home": false,
  "canView": true, "canAccess": true, "children": [] }
```

| 응답 필드 | 화면 구조 매핑 |
| --- | --- |
| `nodeType=FOLDER` + `children` | 사이드바 접이식 폴더(`.sidebar-folder`, `aria-expanded`) |
| `nodeType=MENU` + `urlPath` | `NavLink`; `WorkspaceTabView` 캐시 키; `BackendMenuView` 디스패치 키; `useMenuId(path)`의 `X-Menu-Id` |
| `menuName` / `menuNameEn` | `translateMenuName`: 현재 언어가 `en*`이면 `menuNameEn.trim() || menuName` |
| `home` | `findHomeMenu`: 첫 `home && MENU && urlPath && canAccess` 노드 → `AdminHomePage`가 비관리자 컨텍스트에서 이 경로로 `<Navigate>` |
| `canView` / `canAccess` | 위 변환 규칙. `canAccess=false`인 경로를 직접 열면 `BackendMenuView`가 `AccessDeniedPage` |
| `featureId` | `BackendMenuView`가 경로 매핑에 없을 때 `GET /api/features`로 featureCode 역조회 |

### 8.3 메뉴명 번역 폴백 (`src/shared/lib/i18n/menuName.ts`, 핵심 발췌)

```ts
export function translateMenuName(t, path: string | null, source: { menuName; menuNameEn? } | null): string {
  const dbName = source ? pickByLanguage(source) : '';          // DB 값이 정본(메뉴 관리에서 바꾼 이름이 즉시 반영)
  if (dbName) return t(`menu:names.${dbName}`, { defaultValue: dbName });   // 잘 알려진 한글 폴더명 → 영문(names 맵)
  if (path) { const byPath = t(`menu:${path}`, { defaultValue: '' }); if (byPath) return byPath; } // 정적 nav(DB 행 없음)
  return path ?? '';
}
```

- 정적 네비게이션 항목은 **DB 행이 있으면 그 이름**(`backendMenuByPath.get(item.path)`), 없으면 `null`을 넘겨 `menu:<path>` 매핑으로 떨어뜨린다(경로 문자열을 source로 넘기면 그게 DB 이름으로 취급됨).
- 정적 폴더 라벨은 `menu:groups.<group>`(masterData=기준정보, recipeSensor=레시피/센서, permission=권한, model=모델, operation=운영). 기본 펼침은 `masterData`만(`STATIC_NAV_DEFAULT_EXPANDED_GROUPS`).
- 탭 라벨은 `pageTitle`(위 폴백으로 구한 현재 화면 제목)을 스냅샷으로 저장하고 렌더 시 다시 번역.

### 8.4 권한 필터링 요약

| 지점 | 판정 |
| --- | --- |
| 사이드바(관리자 콘솔) | `LISTED_STATIC_NAV_ITEMS.filter(item => canAccessRoleLevel(user, item.requiredRoleLevel))`, `hiddenInNav` 제외 |
| 사이드바(테넌트) | 서버가 준 `canView/canAccess` + 비ADMIN의 감사 로그 제거 |
| 라우트 | `ProtectedRoutes`(세션) → `AdminSiteRoute`(관리자 컨텍스트) / `TenantDataRoute`(사업부) → `BackendMenuView`(`menu.canAccess`, `AUDIT_LOG_VIEW`는 ADMIN 등급 추가) |
| API | `X-Menu-Id` 헤더(현재 feature의 `apiRules`에 포함된 요청만) — 서버 `MenuAccessFilter`가 rule 검증. 헤더 없으면 서버가 활성 컨텍스트 메뉴로 폴백 검증(fail-open, WARN) / `/api/report/**`는 fail-closed |
| 데이터 | 사업부(`businessDivisions`)는 서버가 행 단위로 거름. 화면 게이트는 안내용 |

`X-Menu-Id` 사용 패턴: 페이지가 `const menuId = useMenuId(ROUTES.xxx)`로 현재 경로의 메뉴 id를 얻어 API 호출 `options`에 `{ menuId }`로 넘긴다(37개 파일에서 사용). VOC·QA처럼 화면에 묶이지 않은 공용 API는 넘기지 않는다.

### 8.5 메뉴 관리 화면 (`/system/menus`, `MenuManagementPage`, 1,085줄)

- `WidgetBoard` 2창: `tree`(x0 w5) / `detail`(x5 w7), `fullHeight`. 헤더: `ManagementTargetSelector` + 도움말 + VOC.
- 데이터: `GET /api/companies/{c}/sites/{s}/menus/tree` (`queryKeys.managementMenus`), `GET /api/features` (`queryKeys.features`).
- 트리: `flattenMenuRows`로 depth 포함 평탄화 → 행마다 `draggable` + `onDragStart/onDragOver/onDrop`(**네이티브 HTML5 DnD**). 드롭 위치 `before | after | inside`(inside는 FOLDER에만). 드롭 결과를 `MenuOrderItem[] {menuId, parentId, displayOrder}`로 만들어 `PATCH .../menus/reorder`.
- 상세 Draft `{ nodeType:'FOLDER'|'MENU', featureId, menuName, menuNameEn, urlPath, active }`. feature 선택(`applyFeatureToDraft`) 시 menuName/urlPath가 비어 있거나 "이전 feature의 기본값 그대로"면 `feature.featureName`/`feature.defaultPath`로 채운다.
- 저장: 신규 FOLDER → `POST .../menus/folders {parentId, menuName, menuNameEn, displayOrder}`; 신규 MENU → `POST .../menus {parentId, featureId, menuName, menuNameEn, urlPath, displayOrder}`; 수정 → `PATCH .../menus/{id}` (`MenuUpdateRequest`). 삭제 → `DELETE`(ConfirmModal). 홈 지정 → `PATCH .../menus/{id}/home`. `active` 토글은 `Toggle`.
- 첫 행 자동 선택, dirty 보호(저장 비활성·이동 확인·`•`), 목록 제목줄 신규 버튼, 상세 제목줄 삭제(ghost)·저장(primary).

### 8.6 역할 관리 화면 (`/system/roles`, `RoleManagementPage`, 1,045줄)

- Draft `{ siteSpecific, roleSiteId, roleName, roleLevel:'USER'|'ADMIN', description, active, permissionIds:Set<number> }`. 범위 select COMPANY/SITE(사이트 역할이면 `roleSiteId`).
- 목록 `GET /api/companies/{c}/roles?siteId=`; 선택 역할의 권한 `GET .../roles/{id}/menu-permissions?siteId=`; 메뉴 트리 `.../menus/tree`를 `flattenMenuPermissionRows`로 "1.2.3" 순번 라벨 + `ancestorFolderIds`를 붙여 체크 목록으로.
- 저장 2단계: `POST`/`PATCH` 역할 → `PUT .../roles/{id}/menu-permissions?siteId=` `{ permissions: [{menuId, canView:true, canAccess:true}] }` — **체크된 메뉴는 canView·canAccess를 둘 다 true로 보낸다**(부모 폴더가 체크 해제돼 `isBlockedByFolder`면 제외). 2단계 실패는 `PermissionSaveError(savedRole)` → create 모드였다면 edit 모드로 전환해 재시도가 중복 생성이 되지 않게 함.
- `system` 역할은 삭제 불가(`systemRoleUndeletableTitle`).

### 8.7 Feature 관리 화면 (`/system/features`, `FeatureManagementPage`, SU)

- 단일 `list` 위젯(`fullHeight`): `GET /api/features` → 컬럼 featureCode/featureName/defaultPath/requiredRoleLevel/`syncStatus` Badge(SYNCED=success, 그 외 warning)/active.
- 제목줄 "Manifest 동기화" 버튼 → `POST /api/features/sync { features: getFeatureManifest() }` (**항상 전체 manifest**). 응답 `{addedCount, updatedCount, missingCount}` 토스트.
- 주의: 낡은 번들에서 누르면 그 번들에 없는 feature가 `MISSING`(비활성)이 된다(2026-09-15 실사고). 배포가 최신인지 먼저 확인.

### 8.8 Feature manifest 현황 (`getFeatureManifest()` 기준, 40개)

| featureCode | defaultPath | 등급 | 주요 apiRules(요약) |
| --- | --- | --- | --- |
| DASHBOARD | /dashboard | USER | GET /api/loads/target/** |
| TRACE_ANALYSIS | /sensor/trace-analysis | USER | GET /api/sensor/trace-analysis/**, GET .../recipe-step-management/key-steps, GET /api/model-versions/card, GET/PATCH /api/sensor/data-chart/sensor-limits, GET/POST/DELETE .../trace-analysis/presets/**, POST /api/report/views(+preview-draft) |
| DATA_CHART | /sensor/data-chart | USER | GET/PATCH /api/sensor/data-chart/**, GET /api/sensor/trace-analysis/** |
| DCOP_ANALYSIS | /sensor/dcop-analysis | USER | GET /api/sensor/fdc/** |
| FDC_ANALYSIS | /sensor/fdc-analysis | USER | GET /api/sensor/fdc/**, GET .../trace-analysis/filter-options |
| DAILY_REPORT | /sensor/daily-report | USER | GET/POST/PATCH/DELETE /api/daily-report/**, GET/POST/PUT/DELETE /api/report/views/**, GET/POST/PUT/PATCH/DELETE /api/report/view-groups/** |
| SENSOR_MANAGEMENT | /sensor/sensor-management | USER | GET/POST/PATCH/DELETE /api/sensor/sensor-management/**, GET .../processes/** |
| RECIPE_MANAGEMENT | /sensor/recipe-management | USER | GET/PATCH/POST /api/sensor/recipe-management/**, GET .../processes/** |
| RECIPE_STEP_MANAGEMENT | /sensor/recipe-step-management | USER | GET/POST/PUT/PATCH/DELETE .../recipe-step-management/**, GET .../processes/** |
| PARAMETER | /sensor/parameter | USER | GET /api/sensor/parameter/** |
| COMPARATOR | /sensor/comparator | USER | GET /api/sensor/comparator/** |
| ALARM_EVENT | /sensor/alarm-event | USER | GET /api/sensor/alarm-events(/**), 리포트 저장 2규칙 |
| ANOMALY_SCORE_DASH | /dashboard/anomaly-score | USER | GET /api/sensor/trace-analysis/** |
| DIAGNOSIS_ANALYSIS_DASH | /dashboard/diagnosis-analysis | USER | GET alarm-status/**, chamber-status/**, trace-analysis/**, 리포트 저장 2규칙 |
| CHAMBER_STATUS | /dashboard/chamber-status | USER | GET /api/sensor/chamber-status/** |
| ALARM_STATUS | /dashboard/alarm-status | USER | GET /api/sensor/alarm-status/**, 리포트 저장 2규칙 |
| EC_COMPARE | /sensor/ec-compare | USER | GET /api/sensor/config-compare/** |
| MODEL_MANAGE | /sensor/model-management | USER | GET/POST /api/model-versions/**, GET processes/**, business-divisions/**, GET /api/model-cfg/**, POST /api/model-cfg/steps/init |
| ADMIN_HOME | /admin | SU | GET /api/me, /api/me/sites |
| COMPANY_MANAGE | /system/companies | SU | GET/POST/PATCH/DELETE /api/companies/** |
| MASTER_DATA_MANAGE | /system/master-data | ADMIN | processes/lines/equipment-groups CRUD, GET business-divisions |
| MODEL_CFG | /system/model-cfg | USER | GET/PUT/POST/DELETE /api/model-cfg/**, POST bootstrap, POST /api/model-versions/replay, GET processes/** |
| MODEL_REGISTRY | /system/model-registry | USER | GET/POST/PATCH/DELETE /api/model-registry/**, GET processes/** |
| MODEL_CONTRACT | /system/model-contracts | ADMIN | GET/POST/PATCH/DELETE /api/model-contracts/** |
| PROCESS_INFO_MANAGE / LINE_MANAGE / EQUIPMENT_MANAGE | /system/processes, /lines, /equipment | ADMIN | 각 마스터 CRUD(장비는 lines GET, processes GET, business-divisions GET 포함) |
| FEATURE_MANAGE | /system/features | SU | GET/POST /api/features/** |
| AUDIT_LOG_VIEW | /system/audit-logs | ADMIN | GET /api/admin/audit-logs/**, login-logs/**, 사이트 스코프 audit/login-logs |
| ERROR_LOG_VIEW | /system/error-logs | SU | GET /api/admin/error-logs |
| PERFORMANCE_LOG_VIEW | /system/performance-logs | USER | GET /api/admin|me/performance-logs, performance-slowest |
| PERFORMANCE_DASHBOARD_VIEW | /system/performance-dashboard | USER | GET /api/admin|me/performance-dashboard |
| PRODUCTION_OVERVIEW | /dashboard/production-overview | ADMIN | GET /api/overview/production |
| DISK_USAGE | /system/disk-usage | ADMIN | GET disk-usage, POST scan, GET scan/**, history, storage-mng GET/PUT |
| BUSINESS_DIVISION_MANAGE | /system/business-divisions | ADMIN | GET/POST/PATCH/DELETE .../business-divisions/** |
| COLLECTOR | /system/collectors | USER | GET/POST /api/collectors, PATCH/DELETE /api/collectors/**, GET processes, business-divisions |
| MENU_MANAGE | /system/menus | ADMIN | GET/POST/PATCH/DELETE .../sites/*/menus/**, GET /api/features/** |
| ACCOUNT_MANAGE | /system/accounts | ADMIN | GET/POST/PUT/PATCH/DELETE .../accounts/**, GET .../roles/** |
| ACCOUNT_BUSINESS_DIVISION_MANAGE | /system/account-divisions | ADMIN | GET .../account-business-divisions, PUT .../accounts/*/business-divisions, GET business-divisions/** |
| ROLE_MANAGE | /system/roles | ADMIN | GET .../sites/*/menus/**, GET/POST/PUT/PATCH/DELETE .../roles/** |

> `/dashboard/trace-log`는 manifest에 없다(DASHBOARD feature의 `/api/loads/target/**` 규칙으로 커버). `MenuAccessFilter`는 **요청의 `X-Menu-Id`가 가리키는 feature의 rule만** 보므로, 다른 feature에 같은 규칙이 있어도 이 화면의 rule에 없으면 403이다 — 그래서 `TRACE_ANALYSIS`에 모델 카드·스텝 규칙이 중복 등록돼 있다.

### 8.9 새 페이지를 추가해 메뉴에 노출시키는 절차 (MENU-GUIDE + `.claude/skills/feature-manifest/SKILL.md`)

1. **페이지 슬라이스 생성**: `src/pages/<slice>/ui/<Xxx>Page.tsx` + `src/pages/<slice>/index.ts` 배럴. 공통 UI는 `@/shared/ui` 배럴에서만. 루트에 `<Page className="page-<slice>">`.
2. **로케일 추가**: `src/shared/lib/i18n/locales/{en,ko}/<slice>.json` 생성 → `resources.ts`에 양쪽 import/등록(키 집합 일치). `menu.json` 양쪽에 `"/<path>": "<라벨>"` 추가. 필요하면 `system.json`의 `feature.manifest.<CODE>.{name,description}`.
3. **라우트 상수**: `src/shared/config/routes.ts` `ROUTES`에 경로 추가.
4. **라우팅 등록**(둘 중 하나 이상):
   - 명시 라우트: `src/app/routes/AppRoutes.tsx`에 `<Route path={ROUTES.x} element={...}/>`. 테넌트 데이터 화면이면 `tenantData(<XPage/>)`, 관리자 콘솔 전용이면 `<AdminSiteRoute>`로 감싼다.
   - 메뉴 디스패치: `src/app/routes/BackendMenuView.tsx`의 `FEATURE_CODE_BY_PATH[ROUTES.x] = 'CODE'` 와 `PAGE_BY_FEATURE_CODE.CODE = () => <XPage/>` 추가. 명시 라우트가 없는 화면은 **반드시** 여기에 있어야 한다(없으면 "오픈 예정").
5. **관리자 콘솔 사이드바 노출**(해당 시): `STATIC_NAV_ITEMS`에 `{ path, featureCode, requiredRoleLevel, group }` 추가 — 같은 group 항목은 배열에서 **붙어 있어야** 한다(`staticNav.test.ts`가 검증). 나열만 빼려면 `hiddenInNav: true`.
6. **Feature manifest 등록**: `src/entities/feature/lib/featureManifest.ts`에 `{ featureCode, featureName: i18n.t(...), defaultPath: ROUTES.x, requiredRoleLevel, description, apiRules:[{httpMethod, pathPattern}] }`. 이 화면이 `X-Menu-Id`와 함께 부를 모든 API 패턴을 적는다(보조 조회 포함). 리포트 소스 저장 버튼이 있으면 `REPORT_VIEW_SAVE_RULES` 스프레드.
7. **화면에서 `useMenuId(ROUTES.x)`** 로 menuId를 얻어 API 호출마다 `{ menuId }` 전달.
8. **검증**: `pnpm type-check`, `pnpm lint`(경고 0), `node scripts/check-i18n.mjs`, `pnpm test`(featureManifest.test / staticNav.test / i18n.test 통과).
9. **백엔드 반영**: SU로 로그인 → `/system/features` → "Manifest 동기화"(전체 manifest). dev 머지 시 CI `build_and_deploy`가 `scripts/ci/feature-sync.sh`로 자동 수행.
10. **메뉴 행 생성**:
    - 테넌트: `/system/menus`에서 해당 사이트에 MENU 노드 생성(feature 선택 → 이름/URL 자동 채움) 후 `/system/roles`에서 역할에 권한 체크.
    - 관리자 콘솔: `ADMIN/ADMIN` 컨텍스트의 `tb_co_menu` 행도 있어야 이름·menu id를 얻는다(없으면 명시 라우트 없는 화면은 "메뉴를 찾을 수 없습니다", 있는 화면은 `X-Menu-Id` 누락 WARN). 시드는 BE Flyway 마이그레이션 또는 메뉴 관리 화면.
11. **금지**: `COMPANY_MANAGE`·`FEATURE_MANAGE`·`ERROR_LOG_VIEW`를 테넌트 메뉴에 두지 말 것(반쪽 화면/카탈로그 노출/빈 화면).

---

## 9. 레이아웃/공통 컴포넌트

### 9.1 App shell (`src/widgets/app-shell/ui/AppShell.tsx`, 849줄)

```
<MenuProvider menus error isLoading isError>
  <div.app-shell>
    <aside.sidebar [is-collapsed]>            ← 브랜드 로고(호스트별) + 닫기 버튼, <nav.sidebar-nav><SidebarTree/></nav>
    <div.shell-main>
      <header.topbar>
        [사이드바 열기 버튼]                    ← 접힌 상태에서만
        <WorkspaceTabBar pageTitle/>           ← 탭 스트립(비밀번호 변경 전용 상태면 <h1>pageTitle</h1>)
        <div.topbar-actions><div.user-menu>   ← 아바타(이니셜) → 팝오버: 이름/ID, 법인 select(SU), 사이트 select(SU·COMPANY 계정)/표시,
                                                  내 사업부(비SU), 전체화면 세그먼트, 테마 세그먼트, 언어 세그먼트, 비밀번호 변경, 로그아웃
      </header>
      <main.page-shell><WorkspaceTabView/></main>
    </div>
    <AppTooltip/>                              ← react-tooltip 단일 인스턴스
    {PasswordChangeModal (required | manual)} {ConfirmModal 미저장 컨텍스트 전환}
  </div>
</MenuProvider>
```

- **브레드크럼은 없다**. 페이지 제목은 탭 라벨이 대신한다(ADR-8). 대시보드의 드릴다운 위치는 `ScopeBreadcrumb` 컴포넌트(페이지 내부).
- 컨텍스트 전환(헤더 셀렉터): 법인 select는 `pendingCompanyId`로 잠시 보류 → 사이트 선택 시 `auth.switchContext` → 성공 토스트 + `navigate('/admin')`. 미저장이면 `ConfirmModal`.
- 전체화면은 `fullscreenOwner()`(`none|page|browser`)로 F11과 페이지 전체화면을 구분, F11이면 버튼 잠금 + 힌트.

### 9.2 공통 컴포넌트 카탈로그 (`src/shared/ui`, 배럴 `index.ts`)

| 컴포넌트 | 용도 | 주요 props | 파일 |
| --- | --- | --- | --- |
| `Page`, `PageHeader` | 페이지 루트(`.page`), 액션만 있는 헤더 | `className`; `actions` | `Page.tsx` |
| `Panel`, `PanelHeader`, `PanelBody`, `PanelFooter`, `SubPanel` | master/detail 패널 골격 | `as`; `title, meta, actions` | `Panel.tsx` |
| `WidgetBoard` | react-grid-layout 기반 창 보드(드래그/리사이즈/닫기/배치 저장) | `pageKey, widgets: WidgetDef[], renderWidget(id), renderWidgetHeader?, renderWidgetTitleMeta?, actions?, showResetLayout?, layoutControls?` | `WidgetBoard.tsx` |
| `DataTable` | 목록 표(헤더 정렬/필터, 가상화, 셀 툴팁, 무한 스크롤) | `columns, rows, getRowKey, selectedRowKey(s), onRowClick, isLoading, isError, emptyTitle, errorTitle, onErrorAction, cellTitles, rowClassName, disableColumnFilters, onDisplayRowsChange, virtualized, virtualRowHeight(36), scrollToRow, headerRows, colGroup, tableStyle, onEndReached, endReachedThresholdPx(120)` | `DataTable.tsx` |
| `SortableDataTable` | dnd-kit 드래그 정렬 표 | `DataTable` 부분집합 + `onRowsChange, isReorderDisabled, getRowLabel` | `SortableDataTable.tsx` |
| `ColumnFilterPopover` | 헤더 캐럿 팝오버(정렬/검색/체크박스) | `anchorEl, sortDir, search, selected, options?, onSort/onSearch/onSelect/onReset/onClose` | `ColumnFilterPopover.tsx` |
| `QueryState` | 로딩→에러(재시도)→빈→콘텐츠 판정 래퍼 | `isLoading, isError, isEmpty, loadingLabel, errorTitle, onRetry, emptyTitle, compact, children(함수 가능)` | `QueryState.tsx` |
| `LoadingBlock`, `EmptyBlock`, `ErrorBlock` | 상태 블록 | `label` / `title, description, actionLabel, onAction, compact` | `StateBlocks.tsx` |
| `AccessDeniedPage`, `TenantRequiredPage` | 접근 거부/테넌트 필요 안내 | `title?, message?` | `AccessState.tsx` |
| `Button` | 버튼 | `variant: primary\|secondary\|danger\|ghost`, `size: sm\|md\|icon` | `Button.tsx` |
| `Badge`, `ActiveBadge` | 상태 칩; 활성/비활성 공통 표기 | `tone: default\|success\|warning\|danger\|info, bordered` / `active` | `Badge.tsx` |
| `Field` | 라벨+컨트롤 래퍼 | `label, required, hint, group(boolean\|'radiogroup'), labelHidden, className` | `Field.tsx` |
| `MultiSelectField` | 검색형 다중 선택 | `options, value, onChange, placeholder, summary, selectAllLabel, clearLabel, emptyLabel` | `MultiSelectField.tsx` |
| `Toggle` | 온/오프 스위치(`.ui-toggle`) | `checked, onChange, disabled, label, ariaLabel` | `Toggle.tsx` |
| `Tabs` | 화면 전환 탭(`.ui-tabs`) | `items[{key,label}], value, onChange, ariaLabel, disabled` | `Tabs.tsx` |
| `Pagination` | 서버 페이징 푸터(`.ui-pagination`) | `page, totalPages, onChange` | `Pagination.tsx` |
| `Modal` | body 포털 다이얼로그(포커스 트랩, ESC 스택) | `title, titleExtra, footer, onClose, className, closeOnBackdrop` | `Modal.tsx` |
| `ConfirmModal` | 확인 다이얼로그 | `title, description, confirmLabel, danger, isPending, onConfirm, onClose` | `ConfirmModal.tsx` |
| `PdfPreviewModal` | PDF blob 미리보기 | `title, blob, onClose` | `PdfPreviewModal.tsx` |
| `ToastProvider` / `useToast` | 토스트(3.5초 자동 소멸) | `success/error/info(message)` | `ToastProvider.tsx` |
| `HelpButton` + `ManualContent` + `ManualReleaseNotes` | 도움말(매뉴얼) 모달(`title "화면 사용법"`, 여러 쪽) | `title, icon, modalClassName` / `intro, sections[{title, body}], vocHint, prelude` | `HelpButton.tsx`, `ManualContent.tsx` |
| `ActionMenu` | 드롭다운 액션 | `items, label` | `ActionMenu.tsx` |
| `AppTooltip` / `APP_TOOLTIP_ID` | 공통 툴팁(요소에 `data-tooltip-id/-content`) | — | `Tooltip.tsx` |
| `TruncatedText`, `RowCountBadge`, `KpiGrid`, `CollapsibleCard`, `ScopeBreadcrumb`, `VerdictValue`, `EquipmentBlock`·`SubsystemBars`·`DiffBars`, `TreePanel`·`TreeBadges`, `ExportButton`, `Sparkline` | 표시 보조 | 각 파일 참조 | `*.tsx` |
| `EChart` | echarts-for-react 래퍼(ResizeObserver로 `resize({width:'auto',height:'auto'})`), `ref.getInstance()` | `EChartsReactProps` | `EChart.tsx` |
| `ChartStepToggle`, `ChartGroupLegend`, `ChartPointStylePicker`, `ChartDrawBar`, `ChartRefLineEditor`, `ChartToolboxIcon` | 차트 보조 컨트롤(스텝 표시/범례/점 스타일/그리기/기준선) | 각 파일 참조 | `Chart*.tsx` |

### 9.3 DataTable 규약

- 컬럼 `filterValue`를 주면 헤더에 캐럿이 붙고, 고유값이 **`MAX_FILTER_CHECKBOX_ITEMS = 20` 이하면 체크박스 다중 선택(정확 일치), 넘으면 부분일치 검색 입력**(`ColumnFilterPopover.tsx`, front-monorepo 공통 DataTable과 동일 값). `sortValue`로 정렬 키만 분리 가능, `filterTokens`로 다중 토큰 컬럼 지원, `filterable:false`로 캐럿만 제거.
- 상태 처리는 자체 props(`isLoading/isError/emptyTitle`)로 하고 `QueryState`로 감싸지 않는다.
- 가상화: 서버 페이징이 불가능한 수천 건 이상에서만 `virtualized`, 행 높이 균일(기본 36px), `getRowKey`는 정렬·필터 후에도 안정.
- `onDisplayRowsChange(visible, total)`로 필터 결과 건수를 위젯 제목(`RowCountBadge`)이나 CSV 내보내기에 연결.
- 사용 예(계정 목록):

```tsx
const columns: DataTableColumn<AccountResponse>[] = [
  { key: 'loginId', header: t('system:account.loginIdHeader'), render: (r) => r.loginId, filterValue: (r) => r.loginId },
  { key: 'status', header: t('system:account.statusHeader'),
    render: (r) => <Badge tone={meta[r.status].tone}>{meta[r.status].label}</Badge>, filterValue: (r) => meta[r.status].label },
];
<DataTable columns={columns} rows={scopedAccounts} getRowKey={(r) => r.id} selectedRowKey={selectedId}
  onRowClick={(r) => confirmLeave(() => select(r))} isLoading={q.isLoading} isError={q.isError}
  emptyTitle={t('system:account.emptyTitle')} />
```

### 9.4 WidgetBoard 규약 (`WidgetDef`)

| 필드 | 의미 |
| --- | --- |
| `id`, `title`, `hideTitle` | 창 식별·제목(창 크롬이 제목을 소유 → 콘텐츠 `PanelHeader`는 제거) |
| `default: {x,y,w,h}` | 12컬럼, `rowHeight 24`, margin `[12,12]`; 목록/상세 h20 기준 |
| `minW`, `minH` | 리사이즈 하한(fullHeight에서는 시스템 계산 바닥값) |
| `autoHeight` (+`maxH`) | 콘텐츠 실측 높이(ResizeObserver)로 창 높이 결정, 가로만 리사이즈 |
| `fullHeight` | 보드 가시 영역의 남은 높이를 채움(표를 담는 마지막 행 위젯에 사용). autoHeight와 동시 선언 금지 |
| `heightGroup` | 같은 줄 창끼리 높이 통일 |
| `fixed` / `locked` / `draggable:false` / `resizable:false` / `closable:false` | 고정(모두 불가) / 잠금(이동·리사이즈 불가, 닫기 가능) / 드래그만 금지 / 리사이즈 핸들 없음 / 닫기 금지 |
| `fillRowNeighbor` | 이 창의 폭을 바꾸면 이웃 창이 나머지 폭을 채움(좌/우 2창 배치) |
| `reserveScrollbarGutter` | 스크롤바 공간 예약 |

배치는 `useWidgetLayoutStore`(localStorage `semi-widget-layout`, `byPage[pageKey] = { positions, hiddenIds }`)에 저장. **기본 배치를 바꾸면 pageKey의 `-vN`을 올려** 저장된 옛 배치가 새 기본값을 덮지 않게 한다. 로그아웃 리셋 대상 아님. 위젯 콘텐츠 최상위가 `Panel`이면 창 크롬이 패널 테두리를 자동 제거.

### 9.5 모달/토스트/확인창 패턴

- 네이티브 `alert/confirm/prompt` 금지(ESLint `no-alert`) — 전체화면이 해제됨.
- `Modal`은 `document.body` 포털(WidgetBoard의 transform이 fixed 기준을 바꾸기 때문). 열릴 때 첫 포커스 가능 요소로 포커스, 닫히면 복원. ESC는 문서 레벨 리스너 + **모달 스택**(마지막에 열린 것만 닫힘). Tab 포커스 트랩. 폼 모달은 `closeOnBackdrop` 기본 false.
- `ConfirmModal`: 삭제/이탈/컨텍스트 전환 확인. `danger`면 확인 버튼 `danger` variant. 비동기면 `isPending`.
- `useToast().success|error|info(message)`: 우하단 `.toast-region`(`aria-live=polite`), 3.5초. 부분 성공은 "~는 저장됐지만 ~ 실패" 형태.
- 미저장 보호: `useUnsavedChangesGuard(isDirty)`(전역 카운터) + `useBlocker`(라우터 이동) + 탭/컨텍스트 전환 전 `hasUnsavedChanges()` 확인 → `ConfirmModal(common:confirm.discard)`.
- 세션 필요 이미지/바이너리는 `<img src="/api/...">` 금지 → `getBlob` + `URL.createObjectURL` + 언마운트 시 revoke(VOC 첨부, 리포트 이미지).

### 9.6 이상점수(Anomaly Score) 표기 표준 (`src/entities/trace/lib/anomalyScore.ts`, CSS `.score-pill`)

| 구간 | 톤 | 색 | 표기 |
| --- | --- | --- | --- |
| `score >= 80` (`WARNING_SCORE`) | `danger` | `--danger` 테두리 1px + `--danger-tint` 배경 | Warning |
| `50 <= score < 80` (`NOTICE_SCORE`) | `warning` | `--warning` + `--warning-tint` | Notification |
| `0 < score < 50` | `success` | `--success` + `--success-tint` | Normal |
| `score == 0` (센서 점수) | `null` → 무색(투명 테두리 유지로 박스 크기 고정) | — | 측정값 없음 |
| 웨이퍼 점수 0 | `waferScoreTone` → `success` (0도 색) | — | 미추론은 `inferred` 플래그로 "-"(`SCORE_NOT_AVAILABLE`) |

- 숫자는 항상 소수 2자리 `formatAnomalyScore` (`0.00`). 차트 마크는 `toneItemStyle(tone)` = `{ color: --<tone>-tint, borderColor: --<tone>, borderWidth: 1 }`로 pill과 같은 "파스텔 tint + 진한 테두리 1px". 0점 마크는 `--surface` 채움 + `--text-muted` 테두리, 미추론은 `--surface-muted` + `--text-muted`.
- 전 화면(TTTM·대시보드·Data Chart·센서 그리드) 공통. 2026-08에 trace-analysis의 60 경계를 대시보드 기준 50으로 통일.

---

## 10. API 레이어/서버 상태

### 10.1 HTTP 클라이언트 (`src/shared/api/http.ts`)

| 설정 | 값 |
| --- | --- |
| `baseURL` | `VITE_API_BASE_URL \|\| ''` → 기본 same-origin(dev는 Vite 프록시, 운영은 nginx가 `/api`를 백엔드로 프록시) |
| `timeout` | 30,000ms (호출별 `timeoutMs`로 연장: VOC 업로드 60s, 리포트 바이너리/미리보기/업로드는 모듈 상수) |
| 쿠키/CSRF | `withCredentials`, `withXSRFToken`, `XSRF-TOKEN` → `X-XSRF-TOKEN` |
| 언래핑 | 모든 헬퍼가 `response.data`를 그대로 반환(서버가 envelope 없이 DTO를 직접 내려줌) |

헬퍼 함수(모두 `RequestOptions { menuId?, signal?, timeoutMs? }` 수용):

| 함수 | 용도 |
| --- | --- |
| `getJson<T>(url, opts)` / `postJson<T,B>` / `patchJson` / `putJson` / `deleteJson` | JSON 요청 |
| `deleteJsonWithBody<T,B>(url, body)` | 본문 있는 DELETE(비밀번호 등 비ASCII) |
| `postEmpty(url)` | 본문 없는 POST(logout) |
| `getBlob` / `postBlob` → `BinaryResponse { blob, contentType, fileName }` | 바이너리. 에러 응답이 JSON이면 `ApiErrorResponse`로 재파싱(`throwNormalizedBinaryError`). `fileName`은 `Content-Disposition`의 `filename*`(RFC 5987) 우선 |
| `postMultipart<T>(url, formData, { onUploadProgress })` | multipart. JSON part는 `new Blob([JSON.stringify(req)], {type:'application/json'})`로 넣어 BE `@RequestPart`가 파싱 |
| `downloadBinary(res, fallbackName)` | `<a download>` 트리거 |
| 에러 유틸 | `getApiErrorMessage(err, fallback)`(서버 `message` → axios message → fallback), `apiErrorCode`, `isApiErrorCode`, `isUnauthorized`(401), `isForbidden`(403), `isConflict`(409), `isClientError`(4xx), `isTimeoutError`(`ECONNABORTED`/`ETIMEDOUT`, 응답 없음) |

### 10.2 API 모듈 규칙

- 컴포넌트에서 `axios`/`fetch` 직접 호출 금지. `entities/<domain>/api/<domain>.ts`에 `xxxApi` 객체로 두고, 화면은 `useQuery/useMutation` + `queryKeys`로만 호출(`.claude/skills/api-integration`).
- 경로는 모듈 내 `basePath(...)` 헬퍼로 조립. 쿼리스트링은 `URLSearchParams` 또는 `buildQuery(params)`.
- 타입은 `src/shared/types/api.ts`에 백엔드 DTO와 **같은 필드명**으로. enum 문자열은 백엔드 원문 유지(번역 금지).
- `menuId`는 현재 feature의 `apiRules`에 포함되는 요청에만 명시 전달(§8.4). VOC/QA는 전달하지 않음.
- 주요 엔드포인트 prefix 맵:

| 도메인(entities) | prefix |
| --- | --- |
| auth | `/api/auth/login`, `/api/auth/logout`, `/api/me`, `/api/me/sites`, `/api/me/active-context`, `/api/me/password`, `/api/me/menus` |
| company / master-data / business-division / menu / role / account / account-business-division / storage-mng | `/api/companies/{c}`, `/api/companies/{c}/sites/{s}/{processes\|lines\|equipment-groups\|business-divisions\|menus\|storage-mng\|audit-logs\|login-logs\|account-business-divisions}`, `/api/companies/{c}/{accounts\|roles}` |
| feature | `/api/features`, `/api/features/sync` |
| trace / data-chart / fdc / paramatcher / config-c / ec-compare / sensor-master / recipe-master / alarm-event / alarm-status / chamber-status | `/api/sensor/{trace-analysis\|data-chart\|fdc\|fdc/dcop\|parameter\|comparator\|config-compare\|sensor-management\|recipe-management\|recipe-step-management\|alarm-events\|alarm-status\|chamber-status}` |
| target (대시보드/트레이스 로그) | `/api/loads/target/{,etch-thickness,sensors,series,thickness-map,trace-log,trace-series,wafer-metrics}` |
| daily-report / report | `/api/daily-report/...`(scope 쿼리 `withScope`), `/api/report/views`, `/api/report/view-groups` |
| model-cfg / model-registry | `/api/model-cfg`, `/api/model-versions`, `/api/model-registry`, `/api/model-contracts` |
| overview / collector / disk-usage / performance / audit-log / error-log / virtual-data | `/api/overview/production`, `/api/collectors`, `/api/admin/disk-usage`, `/api/{admin\|me}/performance-*`, `/api/admin/{audit,login,error}-logs`, `/api/admin/virtual-data/shift-to-today` |
| voc / qa | `/api/voc`, `/api/voc/{id}/{answer\|edit}`, `/api/voc/attachments/{id}`, `/api/qa/runs` |

### 10.3 React Query 키·정책

- **모든 키는 `src/shared/lib/queryKeys.ts` 한 곳**(1,087줄, 약 190개 팩토리). 인라인 리터럴 키는 `pre-mr-check`가 금지(예외: `dcop-analysis`에 `['dcop-params', ...]` 인라인 키가 남아 있음 — 규칙 위반 사례).
- 키 형식: `[name, ...scope, ...params] as const`. 컨텍스트 의존 데이터는 `companyId/siteId`(또는 `accountId`)를 키에 넣어 전환 시 자동 분리. 예: `myMenus(accountId, companyId, siteId)`, `contextOptions(accountId)`, `accounts(companyId)`, `roles(companyId, siteId)`, `traceGraph(...)`, `anomalyHeatmap(...)`.
- 전역 기본값(`main.tsx`): `staleTime 15s`, `refetchOnWindowFocus false`, `retry` = 4xx·타임아웃 제외 1회(타임아웃을 재시도하면 nginx 3600s 동안 서버가 두 벌 완주하는 사고를 막기 위함).
- 화면별 `staleTime`: 사전·옵션류 5분(`DataChartSensorBrowser`, `useKeySteps`, trace sensors, FDC dictionary), 대시보드 카드 60s, 모델 cfg 30s~60s, 스냅샷 `Infinity`, 편집 직전 조회 `0`.
- 서버 상태를 zustand에 복제하지 않는다(단일 소유). 캐시 무효화는 mutation `onSuccess`에서 `invalidateQueries({ queryKey: queryKeys.x(...) })` prefix 매칭.

### 10.4 대형 쿼리 메모리 정책 (keep-alive와의 상호작용)

| 상수 | 값 | 적용 |
| --- | --- | --- |
| `TRACE_HEAVY_QUERY_GC_TIME` | 60,000ms | trace graph-data(풀해상도)·sensor-compare·grid-data (`src/pages/trace-analysis/ui/shared.ts`) |
| `ANOMALY_HEATMAP_GC_TIME` | 60,000ms | anomaly-heatmap(7일 7만 행·JSON 20MB) (`src/entities/trace/api/trace.ts`) |
| `DATA_CHART_HEAVY_GC_TIME` | 60,000ms | Data Chart 통계/원본/타임라인/브라우저 (`src/pages/data-chart/lib/queryConfig.ts`) |
| `COMPARE_GC_TIME` | 60,000ms | Parameter·Comparator 비교 응답(5~7MB) |
| `EC_COMPARE_GC_TIME` | 60,000ms | EC Compare 4개 쿼리 |

규칙: 대형/폴링 쿼리는 `gcTime: 60_000` + **`subscribed: usePageActive()`**. `enabled` 게이팅은 구독이 남아 GC가 안 풀리므로 반드시 `subscribed`여야 한다(2026-08-20 OOM 감사). keep-alive 상한은 5화면.

### 10.5 공통 응답/에러 JSON

```jsonc
// 성공: DTO 직접 반환(envelope 없음)
{ "accountId": 1, "loginId": "<id>", "roleLevels": ["SU"], "authState": "NORMAL", ... }

// 페이지 응답 PageResponse<T>
{ "content": [ ... ], "totalElements": 1234, "page": 0, "size": 50, "totalPages": 25 }

// 에러 ApiErrorResponse (application/json 또는 application/problem+json)
{ "status": 403, "message": "Menu access is denied.", "occurredAt": "2026-10-02T09:00:00+09:00", "code": "MENU_ACCESS_DENIED" }
// code 또는 errorCode 중 하나가 올 수 있다(apiErrorCode가 둘 다 본다)
```

에러 처리 분기: 401 → 전역 만료(§6.4), 403 → `isForbidden`으로 `AccessDeniedPage`/토스트, 409 → `isConflict`(삭제 충돌 안내), 400/기타 → `getApiErrorMessage` 토스트 또는 인라인 `.form-error`, 타임아웃 → 호출부가 한국어 안내로 치환.

### 10.6 실시간/폴링 구독

WebSocket·SSE는 **사용하지 않는다**. 모두 React Query 폴링:

| 화면 | 방식 |
| --- | --- |
| 챔버 가동현황 | `refetchInterval: refreshIntervalMs > 0 ? ms : false`(사용자 선택 주기), `refetchIntervalInBackground: false`, `subscribed: isPageActive` |
| 디스크 사용량 스캔 | 스캔 트리거 후 `refetchInterval: 2000`으로 `GET .../scan/{id}` 상태 폴링 |
| 모델 생성/추론 실행 | `refetchInterval: (q) => 진행 중 run이 있을 때만 RUN_POLL_INTERVAL_MS` (Airflow 부담 방지) |
| 데일리 리포트 발행 | `useGenerationTracking`: `QUEUED/RUNNING` 동안 `GENERATION_POLL_INTERVAL_MS` |
| 세션 | §6.7 유휴 감시(60s 타이머 + 포커스 복귀) |

---

## 11. 상태 관리

원칙(`CLAUDE.md`): 서버 상태는 React Query, 클라이언트 상태(선택·필터·UI 토글) 중 **여러 화면/탭이 공유하는 것만** zustand, 한 컴포넌트 안의 값(폼 draft, 모달 open, 페이지네이션)은 `useState`. 서버 통신이 얽힌 액션은 store가 아니라 훅(`useAuth`, `useManagementTarget`, `useWorkspaceTabs`)에 둔다.

### 11.1 전역 스토어 표

| 스토어 (파일) | 상태 | 액션 | persist | 로그아웃 리셋 |
| --- | --- | --- | --- | --- |
| `useAuthStore` (`entities/auth/model/authStore.ts`) | `user: MeResponse\|null`, `isBootstrapping`, `sessionEndReason: 'logout'\|'expired'\|null` | `setUser`, `setBootstrapping`, `setSessionEndReason` | 없음 | `logout`이 직접 `setUser(null)` |
| `useTargetStore` (`features/target-selector/model/targetStore.ts`) | 관리 대상 `ownerKey, companyId, siteId` | `setCompanyId`(사이트 초기화), `setSiteId`, `syncOwner(ownerKey)`(세션 신원 바뀔 때만 초기화), `reset` | 없음 | 등록 |
| `useWorkspaceTabsStore` (`features/workspace-tabs/model/tabStore.ts`) | `byAccount[accountId] = { tabs, activeTabId }` | `upsertTab, setActiveTab, removeTab, moveTab, removeAllTabs` | sessionStorage `semi-workspace-tabs` v1 | **제외** |
| `useWidgetLayoutStore` (`shared/lib/widgetLayoutStore.ts`) | `byPage[pageKey] = { positions{id: rect}, hiddenIds[] }` | `setPositions, closeWidget, openWidget, resetLayout` | localStorage `semi-widget-layout` v1 | **제외** |
| `useThemeStore` (`shared/lib/theme.ts`) | `theme: 'light'\|'dark'` | `setTheme`(html data-theme + localStorage), `toggleTheme` | localStorage `semi-theme`(수동) | **제외** |
| 언어 | i18next 인스턴스 + localStorage `semi-language`(`useLanguage`) | `changeLanguage` | 수동 | **제외** |
| `useDashboardFilterStore` (`pages/dashboard/model`) | `lotId, waferFilter, stepFilter` | setters, `reset` | 없음 | 등록 |
| `useTraceLogFilterStore` (`pages/trace-log/model`) | `lotId, wafer, step` | setters, `reset` | 없음 | 등록 |
| `useTraceFilterStore` (`pages/trace-analysis/model`) | `fromDate, toDate, lineCd, eqpGrpCd, procCd, eqpCd, dvcCd` | setters, `reset` | 없음 | 등록 |
| `useTraceSelectionStore` (`pages/trace-analysis/model`) | 버킷 `selected/reference`, `selectedSensors`, `selectedGroup`, 색상 | `addSelected, addReference, toggleSelected, removeEntry, resetBucket, clearBuckets, replaceReference, startHandoff, set*Color, reset` | 없음 | 등록 |
| `useTraceViewStore` (`pages/trace-analysis/model`) | 차트 모드/스케일/축, 스텝 밴드, 모달, 제외 목록, 색/선종류, 전체화면, 기준선 등 30여 필드 | setters, `reset` | localStorage `semi-trace-view` | 등록 |
| `useEcCompareStore` (`pages/ec-compare/model`) | 기준/대상 설비·챔버, 표/값 필터, draft 선택 | setters, `reset` | 없음 | 등록 |
| `useParameterFilterStore` (`pages/parameter/model`) | applied(기준 설비·대상들) / draft / 결과 필터(status, treePath, treeSearch) | setters, `reset` | 없음 | 등록 |
| `useComparatorStore` (`pages/comparator/model`) | applied(eqpCd, snapshotDt, targetEqpCd) / draft / 결과 필터(verdict, changedOnly, tree) | setters, `reset` | 없음 | 등록 |
| `useDataChartStore` (`pages/data-chart/model`) | `tabs[]`(scope/stat/raw/timeline/browser setup, 색), `activeTabId`, `skipWaferPrompt` | `addTab, openHandoffTab, removeTab, patch*, set*Color, reset` | localStorage `semi-data-chart` | 등록 |
| `useDcopFilterStore` (`pages/dcop-analysis/model`) | 기간·라인·설비·유닛·웨이퍼·파라미터 키 | setters, `reset` | 없음 | 등록 |
| `useFdcTabStore` (`pages/fdc-analysis/model`) | `tabs[]`(conditions/applied), `activeTabId` | `addTab, removeTab, setActiveTab, patchConditions, resetConditions, apply, reset` | localStorage `semi-fdc-analysis` | 등록 |
| `useContractTestDraftStore` (`pages/model-registry/model`) | `drafts{key: body}` | `setDraft, clearDraft, reset` | 없음 | 등록 |
| `useTrainControlsStore` (`widgets/model-version/model`) | 데이터 구간·사업부·autoChampion·선택·필터·포커스 | setters, `reset` | 없음 | 등록 |
| `useRecaptureStore` (`features/report-source-action/model`) | 리포트 재캡처 `target` | `start, clear` | 없음 | 등록 |

`registerStoreReset(reset)`(`shared/lib/resetStores.ts`)은 모듈 로드 시 각 store가 호출하고, `resetClientStores()`는 로그아웃·세션 만료·컨텍스트 전환에서 호출된다.

### 11.2 탭 keep-alive 스토어와 메모리 주의점

- 탭(북마크)은 무제한, **마운트 유지는 최근 5개**(`MAX_KEEPALIVE_ENTRIES`). 밀려난 화면은 언마운트되고 재진입 시 로컬 상태가 초기화된다 — 중요한 상태는 persist store(`semi-trace-view`, `semi-data-chart`, `semi-fdc-analysis`)에 둔다.
- 숨김 화면은 `display:none`이지만 마운트 상태라 `useQuery` 구독이 살아 있다 → `subscribed: usePageActive()`로 끊지 않으면 gcTime이 흐르지 않고 폴링도 계속 돈다.
- 탭 복귀 시 `refetchQueries({type:'active', stale:true}, {cancelRefetch:false})` — `cancelRefetch` 기본값(true)은 queryFn이 AbortSignal을 안 받아 좀비 다운로드를 쌓는다(72k행 히트맵 24초 왕복 × 31건 중첩 → OOM 실측).
- 컨텍스트 전환은 캐시 전체 clear + 활성 화면 리마운트(`key`에 contextKey 포함) → 이전 법인의 draft/useBlocker가 넘어오지 않는다.
- 숨김 화면의 `<Navigate>`/`navigate()`는 no-op navigator로 무력화(백그라운드 리다이렉트가 라우터를 낚아채던 버그).
- 미저장 변경은 전역 카운터(`useUnsavedChangesGuard`)로만 알 수 있어 "어느 탭이 dirty인지" 모른다 — 다른 탭이 dirty여도 닫기 확인이 뜬다(의도된 보수적 동작).

---

## 12. 스타일/테마/i18n

### 12.1 디자인 토큰 (`src/app/styles/tokens.css`, 303줄)

3계층: ① Primitive(`--slate-50..950`, `--blue-*`, `--green-*`, `--amber-*`, `--yellow-*`, `--red-*`, `--white/--black`, 브랜드 `--navy-900`, `--blue-brand-1/2`, 채널 `--rgb-*`) → ② Semantic(화면이 참조) → ③ Scale.

| 시맨틱 토큰 | 라이트 값(예) | 용도 |
| --- | --- | --- |
| `--background`, `--surface`, `--surface-muted`, `--surface-raised` | slate-50, #fff, slate-100, #fff | 배경/면 |
| `--border`, `--border-strong` | slate-250, #b8c2d1 | 테두리 |
| `--text-strong`, `--text-body`, `--text-muted`, `--text-inverse` | #17202e, slate-700, slate-500, #fff | 글자 |
| `--primary`, `--primary-hover`, `--primary-active`, `--primary-tint`, `--primary-tint-strong`, `--on-primary` | blue-600 … | 선택됨/이동 버튼 |
| `--success(-tint)`, `--warning(-tint)`, `--danger(-tint)` | green-700/100, amber-700/100, red-600/100 | 판정 색 |
| `--chamber-status-processing`, `--chamber-status-nodata`, `--alarm-level-recovery` | yellow-500, black, #7c3aed | 도메인 전용 |
| `--chart-axis/grid/label`, `--chart-1..12`, `--heatmap-1..7` | 직접 색상값 | echarts 주입용(`getChartColors()`) |
| 채널 `--scrim-rgb`, `--on-dark-rgb`, `--focus-rgb`, `--focus-ring` | | `rgb(var(--x-rgb) / a)` |
| Scale `--radius-xs/sm/md/lg/pill`(3/4/6/8/999px), `--shadow-xs..xl` | | |

규칙: 화면/CSS는 시맨틱·스케일 토큰만(`hex/rgb` 하드코딩 금지, Primitive 직접 참조 금지). Tailwind 색/radius/shadow는 `tailwind.config.cjs`에서 토큰 변수에 매핑돼 있어 `bg-surface`, `text-text-muted`, `rounded-md` 등도 토큰 위에서 동작.

### 12.2 테마

- `:root` = 라이트, `[data-theme="dark"]`(tokens.css 227행~)에서 시맨틱 매핑만 교체(Primitive 불변). Tailwind `darkMode: ['selector', '[data-theme="dark"]']`.
- 초기화: `index.html` 인라인 스크립트가 `localStorage.semi-theme` → 없으면 `prefers-color-scheme`으로 `<html data-theme>`를 첫 페인트 전에 설정; `initTheme()`가 같은 로직을 store와 동기화.
- 전환: 사용자 메뉴 세그먼트 → `useThemeStore.setTheme`. 차트는 CSS 변수를 못 쓰므로 `getChartColors()`/`readCssColor('--token')`로 해석하고 `useThemeStore((s)=>s.theme)`를 `useMemo` 의존성에 넣어 재계산(ESLint exhaustive-deps 오탐은 줄 단위 disable).
- `prefers-reduced-motion: reduce` 미디어쿼리로 트랜지션 축소.

### 12.3 스타일 작성 규칙과 반응형

- 공통 요소·컴포넌트 기본 스타일은 CSS(`global.css`, `.ui-toggle/.ui-tabs/.ui-pagination/.panel/.data-table/.widget-window/...`), 화면별 배치·상태 조정은 Tailwind 유틸리티. `.control`은 `width:100%`라 툴바 인라인 컨트롤 폭은 `style={{width:n}}`로 지정(Tailwind `w-*`가 전역 CSS에 진다).
- 페이지 스코프: `.page-trace`, `.page-anomaly-score-dash`, `.page-dashboard`, `.page-ec-compare`, `.page-fdc-analysis`, `.page-dcop-analysis`, `.page-admin-home` 등 루트 클래스 아래에 선택자 작성.
- 반응형은 제한적(관리 콘솔 데스크톱 전제): `@media (max-width: 1200px)` 3곳, `1100px` 2곳, `760px` 4곳, `720px` 1곳 — 비교 화면 요약 블록/대시보드 컬럼 축소 정도. 모바일 레이아웃 없음.
- 타이포: body 13px/1.5. 대시보드 4화면은 13/12/11/10px, 무게 700/800만(`dashboardPanels.css` 머리 주석). 비교 화면 캡션 10px/700, 표 본문 11~12px.
- 아이콘 `react-icons/lu`, 색은 `currentColor`.

### 12.4 i18n 구조 (`src/shared/lib/i18n/`)

| 항목 | 값 |
| --- | --- |
| 라이브러리 | i18next + react-i18next, 리소스 **인라인 번들**(`resources.ts`가 54개 JSON import) → 지연 로드 없음, `init` 동기 |
| 언어 | `LANGUAGES = ['en','ko']`, **`DEFAULT_LANGUAGE = 'ko'`**(2026-08-27, 폐쇄망 국내 공장 납품), `fallbackLng`도 ko |
| 저장 키 | localStorage `semi-language`. 사용자가 한 번 고르면 기본값보다 우선 |
| 전환 | `useLanguage()` → `{ language, changeLanguage(lang) }` = `i18n.changeLanguage` + localStorage 저장. UI는 사용자 메뉴 세그먼트(English / 한국어) |
| defaultNS | `common` |
| 타입 | `i18next.d.ts`의 `CustomTypeOptions`로 `resources.en` 기준 키 타입 체크, `returnNull: false` |
| 보간 | `escapeValue: false`; 한국어 조사 포매터 `{{label, eul}}`(을/를), `{{label, i}}`(이/가) — `attachParticle`이 받침 판정 |
| 네임스페이스(27) | `common`, `menu`, `app-shell`, `login`, `dashboard`, `trace-log`, `data-chart`, `dcop-analysis`, `fdc-analysis`, `trace-analysis`, `target`, `window-dashboard`, `master-data`, `model-cfg`, `system`, `daily-report`, `report`, `ec-compare`, `alarm-event`, `voc`, `qa`, `sensor-master`, `business-management`, `recipe-management`, `recipe-step-management`, `parameter`, `comparator` |
| 키 수 | 언어당 leaf 5,694개(가장 큰 ns: `system` 1,256, `daily-report` 901, `model-cfg` 583, `window-dashboard` 422, `trace-analysis` 419) |

- ns ↔ 슬라이스 매핑은 대체로 1:1이지만 예외가 있다: `system`은 관리자 콘솔 화면 전체(계정·역할·메뉴·Feature·로그·관리 홈 등)를 담고, `window-dashboard`는 해체된 위젯 대시보드의 후속 4화면(anomaly-score/diagnosis-analysis/chamber-status/alarm-status)이 공유하며, `target`은 dashboard/trace-log 엔티티 UI, `report`는 리포트 소스 기능이 쓴다.
- 사용 패턴: `const { t } = useTranslation(['system','common'])` 후 `t('system:account.saveSuccess')`. 컴포넌트 밖(모듈 스코프·클래스·store)에서는 `i18n.t(...)`를 **호출 시점에** 평가한다 — `featureManifest`가 함수(`getFeatureManifest()`)인 이유. **모듈 스코프에서 `t()`를 미리 계산해 상수로 두면 언어 전환이 반영되지 않으므로 금지.**
- 백엔드 enum·로그 기술 값(INSERT/SUCCESS/SYNCED, HTTP method 등)은 번역하지 않음. 단 활성/비활성은 `ActiveBadge`/`Toggle` 공통 표기.
- 메뉴 라벨 폴백은 §8.3.

### 12.5 `scripts/check-i18n.mjs`

1. `locales/en`과 `locales/ko`의 파일 목록이 같은지, 각 파일의 **leaf 키 집합**이 같은지 검사(`[parity]` 오류).
2. `src/**/*.{ts,tsx}`(테스트·locales 제외)에서 블록 주석을 지운 뒤 한글(`[가-힣]`)이 남은 줄을 찾는다. 줄 주석(`//`) 뒤만 한글이거나 `i18n-exempt` 마커가 있으면 통과(`[korean]` 오류).
3. 하나라도 실패하면 `exit 1`. CI 게이트에는 포함돼 있지 않고(`type-check/lint/test`만) 로컬 규칙(`CLAUDE.md`: UI 문구를 만졌으면 통과시킨다).

---

## 13. 그 외 도메인 화면

공통 골격(대부분 화면): `Page` > `WidgetBoard`(헤더 바 = 조회 조건 `.board-filter-toolbar` + 오른쪽 끝 `HelpButton`·`VocButton`) > 위젯 창들. 대상 법인/사이트는 `useManagementTarget()`(관리자 콘솔 SU는 셀렉터로 고르고, 테넌트 계정은 세션 활성 컨텍스트가 곧 대상) 또는 코드가 필요한 화면은 `useSiteScope()/useProcessScope()`(`widgets/process-scope`, `contextOptions`로 id→code 해석). 위젯 id는 기준 커밋의 `WidgetDef[]`에서 확인한 값이다.

### 13.1 대시보드 (`/dashboard`, `DashboardPage`)

ETCH 계열 타깃 데이터 대시보드. 위젯 `etch`(x0 w6 h14, `EtchThicknessChart`) · `waferMetric`(x6 w6 h14, `WaferMetricChart`) · `thicknessMap`(전폭, `autoHeight`, `ThicknessMapView`) — 차트 컴포넌트는 `entities/target/ui`. 필터는 `useDashboardFilterStore`(`lotId`, `waferFilter` 'all'|'1'..'10', `stepFilter` 'all'|'TPE10'|'MTM40'|'EPO10'). 상단 표 3종(Wafer/Snapshot/Measurement)은 `SHOW_DASHBOARD_TABLES = false`로 숨김. API `GET /api/loads/target?…`, `/etch-thickness`, `/wafer-metrics`, `/thickness-map`(`entities/target/api/target.ts`). Feature `DASHBOARD`.

### 13.2 트레이스 로그 (`/dashboard/trace-log`, `TraceLogPage`)

위젯 `filter`(h5) + `list`(h18). LOT 입력칸은 로컬 상태, 적용값은 `useTraceLogFilterStore`(`lotId`, `wafer`, `step`). `GET /api/loads/target/trace-log?…`를 **서버 페이징**(`PAGE_SIZE = 100`, `Pagination`)으로 조회해 `DataTable`에 표시. 시각은 `value.replace('T',' ').slice(0,23)`로 표기.

### 13.3 트레이스 분석 / TTTM (`/sensor/trace-analysis`, `TraceAnalysisPage`, 2,029줄)

웨이퍼(또는 슬롯)를 SELECTED/REFERENCE 두 버킷에 담아 센서 트레이스를 겹쳐 비교하는 핵심 분석 화면. 위젯: `warn-chambers`(이상 챔버 TOP) · `kpi` · `target`(웨이퍼 표, score pill) · `selected` · `reference` · `in-chamber` · `tttm` · `sensors`(센서 목록/그룹) · `chart`(그리드 모드는 센서별 미니차트, 타임라인 모드는 웨이퍼N×센서N). 상태: `useTraceFilterStore`(기간·라인·설비군·공정·설비·챔버), `useTraceSelectionStore`(버킷·센서·색), `useTraceViewStore`(차트 모드·스케일·축·스텝 밴드·기준선 등, localStorage `semi-trace-view`). 화면을 열면 **이상 챔버 1위로 자동 진입해 상위 웨이퍼 한 장을 SELECTED에 담는다**(`autoEntryRef`). API(`entities/trace/api/trace.ts`, prefix `/api/sensor/trace-analysis`): `filterOptions`(/filter-options), `warningChambers`, `wafers`, `sensors`, `sensorAverage`, `graphData`(/graph-data, 서버 `maxPoints` 다운샘플), `sensorCompare`, `gridData`, `inChamberWafers`(/in-chamber), `tttmWafers`(/tttm), `waferCompare*`(/wafer-compare/…: distribution·auto-limits·population), `presets` CRUD(`features/trace-preset/TracePresetModal`). 대형 쿼리는 `TRACE_HEAVY_QUERY_GC_TIME` 60s + `subscribed`. 차트 포인트: echarts만, 축 조작은 `useChartAxisZoom`(휠=가로 확대, 눈금 띠 휠=그 축, 드래그=이동, Shift+드래그=상자 확대, 우클릭=한 단계 되돌리기, 더블클릭=전체), **옵션에 `sampling` 금지**(LTTB가 1점 스파이크를 지움 — 스파이크 보존은 서버 `TraceSeriesDownsampler`의 구간별 최저·최고 2점), 세로축 기본 고정, 점수 구간색은 §9.6. Data Chart로 넘기는 링크(`TraceDataChartLink`) 제공.

### 13.4 Data Chart (`/sensor/data-chart`, `DataChartPage`)

센서 통계·원본 트레이스를 **카드로 쌓아 비교**(dutchboy data-chart 이식). 위젯 `browser`(센서 훑어보기 미니 차트) · `charts`(카드 목록, 9칸) · `stat`(Statistics 조회 조건, 레시피 단위) · `timeline` · `raw`(웨이퍼 단위). 카드 종류 `kind: 'stat' | 'raw' | 'timeline'`. 탭 여러 개(`useDataChartStore.tabs`, localStorage `semi-data-chart`), TTTM에서 넘어온 웨이퍼는 `openHandoffTab`으로 새 탭에 담김. API `/api/sensor/data-chart/{stats, steps, recipes, recipe-stats}` + 웨이퍼·센서·원본 트레이스는 trace-analysis API 재사용 + `PATCH .../sensor-limits`(규격·관리 한계 수정). 판이 여러 개인 통계 카드는 판마다 `DataChartStatGroupChart`로 나눠 `useChartAxisZoom`을 하나씩 건다. `DATA_CHART_HEAVY_GC_TIME` 60s. 배치 키 `data-chart-v7`.

### 13.5 위젯 대시보드 해체분 4화면 (`window-dashboard` 네임스페이스 공유, 공통 규칙은 `src/app/styles/dashboardPanels.css` 머리 주석)

| 화면 | 구성 | API | 차트/포인트 |
| --- | --- | --- | --- |
| **Anomaly Score Dash** `/dashboard/anomaly-score` (`AnomalyHeatmapPanel`, `.page-anomaly-score-dash`) | 위젯 `summary` · `chamber-risk`(`entities/trace/ui/ChamberRiskSection`) · `recipe-risk` · `recipe-heatmap`, 설비 추세 `EquipmentTrendSection` | `GET /api/sensor/trace-analysis/anomaly-heatmap`, `…/anomaly-heatmap` 셀 조회(`anomalyHeatmap`, `anomalyHeatmapCells`) — 집계 없이 웨이퍼 1장=1행(7일 7만 행) | `ANOMALY_HEATMAP_GC_TIME` 60s + `subscribed`. 순위 기본 기준은 **건수가 아니라 비율**(처리량 편향 방지, `docs/anomaly-score-dashboard.md`). 히트맵 빨강↔초록, 드릴다운 위치 `ScopeBreadcrumb` |
| **진단 분석 Dash** `/dashboard/diagnosis-analysis` (`AnomalyWidgetDashBoard`) | 위젯 `nav`(탐색) · `detail`(상세) + 카드 `summary`, `chamberHealth`, `chamberStatus`, `alarmLevel`, `alarmEventSeverity`, `alarmEventRecovery`, `alarmEventTop`, `chamberRisk`, `hourly`, `bar`, `chart`. `draggable:false` + `fullHeight`, 좌우 폭만 조절(`fillRowNeighbor`) | alarm-status / chamber-status / alarm-events summary / anomaly-heatmap, 리포트 `GET/POST /api/dashboard/diagnosis-analysis/report`(`render`, `texts`, `saveTexts`, 카드별 데이터) | 리포트는 `DiagnosisReportModal` + `PdfPreviewModal`. `ReportSourceAction`(데일리 리포트 사이트 소스 저장). 카드 staleTime 60s |
| **챔버 가동현황** `/dashboard/chamber-status` (`ChamberStatusBoard`) | 위젯 `summary`(`KpiGrid` 6타일) · `line`(전체·라인별) · `group`(모델별, 6칸씩) · `equipment`(설비별) · `grid`(표). 카드 클릭=포커스(아래만 좁힘), 순위 상한 라인1·모델1·설비3 | `GET /api/sensor/chamber-status` | 갱신 주기 select `0/1/5/10/30/60분`(`refetchInterval`, 백그라운드 정지, `subscribed`). 상태색: ABORT `--danger`, READY `--success`, PROCESSING `--chamber-status-processing`(노랑), 데이터 없음 `--chamber-status-nodata`(검정). KPI 이미지 저장 `buildKpiImage`. 현재 상태 판정 규칙은 `docs/chamber-status-dashboard.md`(키별 최신 `occur_datetime`, 동일 시각은 Start 우선) |
| **알람 현황** `/dashboard/alarm-status` (`AlarmStatusPanel`) | 위젯 `summary` · `trend` · `lines` · `groups` · `equipments` · `codes` · `list`(챔버 가동현황과 같은 구조) | `GET /api/sensor/alarm-status/{summary, list, export}` (목록 `PAGE_SIZE = 50`, 기본 기간 6일) | 알람 레벨 색은 CSS `.alarm-level--N`과 `alarmLevels.ts` 토큰 맵이 같은 토큰을 봐야 함(레벨 4 Auto Recovery = `--alarm-level-recovery` 보라). `ReportSourceAction` |

공통: 타이포 13/12/11/10px·무게 700/800만, 파랑(`--primary`)은 "선택됨"과 "다른 화면으로 이동"(TTTM 버튼)만, 순위 배지는 중립색, 위젯 `locked`(순서 고정)이지만 닫기·위젯 추가·배치 초기화는 허용.

### 13.6 Production Overview (`/dashboard/production-overview`, `ProductionOverviewPage`, nav 숨김)

사이트 단위 라인별 전일 가동 지표 + 지도 + 이상 챔버 상위. 위젯 `map-korea` · `map-usa`(라인의 `latitude/longitude`를 지도 점으로, 미입력 라인은 지도에서만 빠짐) · `summary` · `output-tree`(생산량 구성). `GET /api/overview/production?…`. 사이트는 관리 대상 셀렉터로 고르고(CLEAN·ETCH) 스코프는 서버가 강제(`SiteScopeResolver`). 판정 표기는 "경고/알림/정상".

### 13.7 설비 설정 비교 3화면 — EC Compare · Parameter · Comparator

| 화면 | 비교 대상 | API | 스토어 |
| --- | --- | --- | --- |
| EC Compare `/sensor/ec-compare` | ETCH(E-PLUS) `tb_semi_config`를 챔버 축으로 | `/api/sensor/config-compare/{lines, equipment-groups, equipments, chamber, common}` | `useEcCompareStore` |
| Parameter `/sensor/parameter` | ParaMatcher 기준값 대비 실측(TTTM) | `/api/sensor/parameter/{lines, equipment-groups, equipments, compare}` | `useParameterFilterStore` |
| Comparator `/sensor/comparator` | 설비 설정 백업(.bak, `tb_semi_config_c`) 스냅샷 간 | `/api/sensor/comparator/{lines, equipment-groups, equipments, compare, snapshot-dates, compare-snapshot}` | `useComparatorStore` |

공통 골격(`CLAUDE.md`): 위젯 `summary`(기준·대상·대조 `EquipmentBlock` 3블록, `autoHeight`+`maxH`) → `tree`(좌, `TreePanel`, `fullHeight`) | `grid`(우, `fullHeight`). 툴바 = 기준 그룹 · 조회/초기화 · 대상 그룹 · 도움말/VOC. 화면을 열면 첫 설비로 한 번 자동 조회, 대상 설비는 고르는 즉시 반영(조회 버튼은 기준만). 판정 라디오 `.eqp-radio-group`, 상이 값 `.eqp-value--diff`, 용어 기준/대상·동일/상이(+기준만/대상만, Comparator 시간축은 변경/삭제됨/추가됨). 성능(`docs/parameter-comparator-perf-architecture.md`): 응답 5~7MB → **쿼리 키는 설비 집합만**, 트리 경로·판정·검색 필터는 `useMemo` 클라이언트 파생(네트워크 0회), 트리 가상화·기본 접힘, `gcTime` 60s + `subscribed`. 배치 키 `parameter-v9`, `comparator-v5`.

### 13.8 FDC 분석 · DCOP

- **FDC** (`/sensor/fdc-analysis`, `.page-fdc-analysis`): 장비 한 대의 1초 파형을 기간·잡·챔버·센서로 조회. 위젯 `conditions`(조건) · `chart`. 조건은 탭 단위(`useFdcTabStore`, localStorage `semi-fdc-analysis`: `tabs[{conditions, applied}]`). API `/api/sensor/fdc` 하위 `chambers`, `sensors`, `graphData`, `waferSpans`, `limits`(+ trace `filter-options`). 장비 사전은 `staleTime` 5분, 받아 둔 값이 있으면 재조회 실패를 오류로 올리지 않음(`listState`).
- **DCOP** (`/sensor/dcop-analysis`, `.page-dcop-analysis`): 웨이퍼당 1행 공정 파라미터 스냅샷을 표 + 점 추이(높이 300)로. `useDcopFilterStore`(기간·라인·설비·유닛·웨이퍼·파라미터 키) → 먼저 `paramKeys`(`/api/sensor/fdc/dcop/params`, jsonb 키 목록) → "조회" 시 조건을 얼려(`AppliedQuery`) `rows` 조회. 빈 결과가 현재의 정상 경로라 오류가 아닌 안내로 표시.

### 13.9 알람 이벤트 (`/sensor/alarm-event`, `AlarmEventPage`, 메뉴 디스패치 전용)

SSE10 전용 SCAMS 이벤트 로그 화면(알람 현황과 다른 데이터). `TenantDataRoute`로 감쌈. 위젯 `summary` + 차트 탭(`ChartTab`, 기본 `module`) + 이벤트/라이프사이클 표. API `/api/sensor/alarm-events/{summary, chart-series, lifecycles, export, lifecycles/export, module-names}`(module-names 키는 기간 무관 사이트 단위). `ReportSourceAction`으로 사이트 소스 저장.

### 13.10 데일리 리포트 (`/sensor/daily-report`, `DailyReportPage`)

탭 `history`(기본, "오늘 리포트가 나왔는지") · `template`(편집) · `siteSources`(모든 사용자) · `dataSources`(`canAccessRoleLevel(user,'ADMIN') && !isAdminContext`일 때만). 편집기는 **TipTap 3**(`ui/editor/ReportEditor.tsx`) + 커스텀 노드 `reportPage`, `reportChart`(차트 11종), `reportDataTable`(동적 표), `reportField`(필드 칩), `reportImage`; 페이지 흐름·밴드·쪽 나눔·목차 유틸(`pageFlow.ts`, `pageBands.ts`, `tableOfContents.ts`). API(`entities/daily-report/api/dailyReport.ts`, `/api/daily-report/...` + `withScope` 쿼리): 카탈로그·데이터 소스(활성 토글)·카테고리 CRUD/순서, 템플릿 CRUD·내보내기/가져오기(파일·텍스트 분석→가져오기), 외부/진단 템플릿, 글로벌 필터 옵션, 대상 옵션, 에셋 업로드(`postMultipart`, JSON part Blob)·조회(`getBlob`→objectURL), 샘플 미리보기·쪽수, 발행 이력·생성 상태(`QUEUED/RUNNING` 폴링). 템플릿·이력 상세는 `PdfPreviewModal`. 사이트 소스 = `/api/report/views`·`/api/report/view-groups`(`entities/report`, 소스 정의 레지스트리 `pages/*/lib/reportSource.ts` 5개: 진단 분석·알람 이벤트·성능 대시보드·챔버 가동현황·알람 현황). 저장 요청·차트 노드는 `contract/` 픽스처로 BE 검증기와 계약 테스트(§3.5).

### 13.11 기준정보 (법인/사이트 · 마스터 데이터 · 공정 · 라인 · 장비 · 사업부)

- **법인/사이트** (`/system/companies`, SU): `companyApi` `list/create/get/update/delete/reorder` + `sites/createSite/updateSite/reorderSites/deleteSite`(`/api/companies`, `/api/companies/order`, `/api/companies/{c}/sites…`). 법인·사이트 목록 2단.
- **공정/라인/사업부**: 공용 `entities/master-data/ui/MasterDataManager`에 어댑터(`label, nameHeader, queryKey, list, create, update, reorder, remove`, 코드 불변 `lockCodeOnEdit`)를 물린 thin 페이지. 목록은 `SortableDataTable` + `useOrderedList`(드래그 → 되돌리기/순서 저장, 저장 대기 중 상세 저장·삭제·신규 잠금 `order-lock-note`), 순서는 `PATCH …/order`에 전체 id 목록. 신규 폼 "삽입 순서"(`resolveInsertOrder`: 비우면 선택 행 자리, 없으면 맨 뒤), 수정 폼 읽기전용 "현재 순서". 라인은 타임존(`Asia/Seoul`·`America/Chicago`·`Asia/Shanghai`)·위경도 필드. 사업부 코드는 대문자 강제, 사용 중 삭제는 409.
- **장비** (`EquipmentManager`): 설비군(equipment-groups) 아래 장비 CRUD, 라인/공정/사업부 지정과 필수 여부, 챔버 목록 `devices[]`(구서버 호환 `pmCount`·`devicePrefix` 동시 전송).
- **마스터 데이터** (`/system/master-data`, nav 숨김): 공정·라인·장비 셋을 `Tabs`로 묶은 통합 화면(같은 매니저 재사용).
- API: `masterDataApi` `processes/lines/equipmentGroups/equipment` × `create/update/reorder/delete` (`/api/companies/{c}/sites/{s}/{processes|lines|equipment-groups}/…`).

### 13.12 운영·시스템 화면

| 화면 | 요점 |
| --- | --- |
| 관리 홈 `/admin` | 위젯 `session`(autoHeight) · `links`(`LISTED_STATIC_NAV_ITEMS`를 묶음별 카드로, fullHeight) · `tools`(**내부 주소에서만**, `isCurrentHostInternal()`: VOC 게시판, `public/docs` ERD·수집 파이프라인 문서 새 탭, 타임존 안내, 가상데이터 시프트 `POST /api/admin/virtual-data/shift-to-today`). 비관리자 컨텍스트면 `useHomeMenu()` 경로로 리다이렉트 |
| 감사 로그 `/system/audit-logs` | `Tabs`(감사 / 로그인), 조건은 헤더 툴바(디바운스 즉시 반영), 서버 페이징 `PAGE_SIZE = 50`, 관리자 컨텍스트는 `/api/admin/{audit,login}-logs`, 사이트 스코프는 `/api/companies/{c}/sites/{s}/{audit,login}-logs`. 원본/변경 데이터·SQL 상세 |
| 에러 로그 `/system/error-logs` (SU) | `GET /api/admin/error-logs`, 같은 지문 반복은 `occurrenceCount`로 접힘, 목록|상세 `fullHeight` |
| 성능 로그·대시보드 | `basePath(scope)` = `/api/admin` 또는 `/api/me` + `/performance-logs`, `/performance-slowest`, `/performance-dashboard`. 대시보드는 KPI(`KpiGrid`) + 시간대 추이 차트 + 엔드포인트/사이트 표, `ReportSourceAction` |
| 디스크 사용량 `/system/disk-usage` (ADMIN) | 경고 80%/위험 90%, 위젯 요약(autoHeight) → 디스크 루트 | 데이터 저장기간(storage-mng) → 모니터링 경로 → 설비별 트리 표. 스캔 `POST /api/admin/disk-usage/scan` → `GET .../scan/{id}` 2초 폴링, 이력 `/history` |
| FTP / S3 등록 `/system/collectors` | `collectorApi` `list/create/update/remove`(`/api/collectors`), 수집 타입 FTP/S3, 자격증명은 값 대신 설정 여부만 응답, 목록|상세 `fullHeight` |
| 계정 사업부 권한 `/system/account-divisions` | 계정별 사업부 체크 → `PUT .../accounts/{id}/business-divisions` |

### 13.13 모델 4화면 (용어: "학습" 금지 → "모델 생성", "데이터 구간", "운영으로 전환")

- **모델 등록** (`ModelRegistryPage`): `modelRegistryApi` `contracts, list, detail, usage, create, update, remove, healthCheck, contractTest, inference`(`/api/model-registry/…`). 외부/내부 모델, 인증 타입(`NONE|BASIC|BEARER|API_KEY`), 점수 스케일, 계약 테스트 본문 초안은 `useContractTestDraftStore`.
- **모델 계약** (`ModelContractPage`, ADMIN): 공통 표준 계약 + 사이트별 매핑형 사용자 계약, `/api/model-contracts/**`, 요청/응답 미리보기.
- **모델 Config** (`ModelCfgPage`): `modelCfgApi` `list, detail, save, remove, steps, stepRecipes, updateSteps, initSteps, initAllSteps, equipmentGroups, internalModels, bootstrap, binFill, history, preflight, sensors`(`/api/model-cfg/**`). 미저장 보호(저장 비활성·이동 확인·`•`).
- **모델 관리** (`ModelManagementPage` → `widgets/model-version`): `ModelStatusBoard`·`ModelVersionPanel`·`TrainRunModal`; `modelVersionApi` `overview, eqpGroups, versions, train, infer, reinfer, reinferTarget, trainRuns, inferRuns, trainHistory, cancel, retry, promote, replay, cfgSnapshot, deleteVersion`(`/api/model-versions/**`). 진행 중 run이 있을 때만 폴링. 헤더엔 공정 선택 + 생성 버튼 둘(선택 생성·전체 모델 생성), 데이터 구간·사업부·자동 운영 전환은 생성 확인 창에서. `MODEL_CD = 'semi_stat'`(현재 모델 종류 1개).

### 13.14 센서 · 레시피

- **센서 관리** (`SensorMasterPage`, 배치 키 `sensor-master-v2`): `sensorMasterApi` `groups/createGroup/updateGroup/deleteGroup, models, sensors/updateSensors, limits/limitDetail/updateLimits`(`/api/sensor/sensor-management/…`), Model 셀렉터의 "전체"는 `__all__`(빈 문자열은 "Model 미지정" 의미), 한계 차트 `SensorLimitChartModal`.
- **레시피 관리**: `recipes/updateRecipes, syncSteps(/recipes/step-sync), durationStats, testToolStatus, shorten/restore RecipeNames·LotCodes`(`/api/sensor/recipe-management/…`).
- **레시피 스텝 관리**: 좌 레시피 목록 | 우 주요 스텝(공통 + 센서별), `recipeStepApi` `recipes, steps, keySteps, updateKeySteps, assignKeySteps, sensors, sensorSteps, saveSensorSteps, clearSensorSteps`. 배치 키 `recipe-step-management-v3`.

### 13.15 VOC (공용 문의 게시판)

모든 화면 헤더 오른쪽 끝의 `VocButton screenKey="<화면키>"`(`entities/voc/ui`). 모달 탭 `new`(작성) · `list`(목록, 답변대기 필터·검색) · `guide`(사용 안내). `vocApi`: `create`(multipart: `request` JSON Blob + `images[]`, 타임아웃 60s) · `list`(`GET /api/voc`, 활성 법인 전체 문의) · `attachmentContent`(`getBlob` → objectURL, `VocAttachmentImage`가 AbortController·revoke) · `answer`(`PATCH /api/voc/{id}/answer`) · `update`(`POST /api/voc/{id}/edit` multipart — Tomcat이 PATCH multipart를 파싱하지 않아 POST) · `remove`(`DELETE /api/voc/{id}` + JSON 본문 비밀번호, soft delete). 첨부 최대 3장(`MAX_ATTACHMENTS`), 클라이언트에서 최대 변 1920px JPEG로 재압축(커지면 원본), 작성자/답변자 이름은 localStorage에 기억해 프리필. 특정 feature에 묶이지 않아 `X-Menu-Id` 미전송. QA 체크리스트 버튼(`QaButton`, `/api/qa/runs`)은 코드·API는 있으나 `SHOW_QA_CHECKLIST = false`로 숨김.

---

## 14. 테스트/품질

### 14.1 단위·컴포넌트 테스트 (vitest)

| 항목 | 값 |
| --- | --- |
| 러너 | vitest ^2.1.9, `environment: jsdom`, `css: false`, `include: src/**/*.test.{ts,tsx}` (`vite.config.ts`의 `test` 블록) |
| 규모 | 기준 커밋 테스트 파일 300개(pages 위주) |
| 셋업 `src/test/setup.ts` | `@testing-library/jest-dom/vitest`, `afterEach(cleanup)`, `ResizeObserver` 목, `HTMLElement.prototype.scrollTo` 목, **Map 기반 `localStorage` 목**, `matchMedia` 목 → 그 뒤에 `await import('@/shared/lib')`로 `localStorage['semi-language']='en'` 고정 후 **`initI18n()`** |
| 언어 고정 이유 | 앱 기본 언어는 ko지만 컴포넌트 테스트 셀렉터가 영어(`findByText('Save')` 등) — 기본 언어가 ko인지는 `i18n.test.ts`가 `getInitialLanguage()`로 별도 검증 |
| 대표 테스트 | `shared/api/http.test.ts`, `shared/lib/i18n/i18n.test.ts`(기본 ko·메뉴명 폴백·en/ko 번역), `shared/config/staticNav.test.ts`(묶음 연속성·순서·단독 항목·menu.json 라벨), `entities/feature/lib/featureManifest.test.ts`, `shared/ui/DataTable*.test.tsx`, `WidgetBoard.test.tsx`, 각 store 테스트, `app/reportSources/{descriptorCoverage,descriptorRules,saveRequestContract}.test.ts`(계약 픽스처 생성), `pages/daily-report/ui/editor/viewChartContract.test.ts` |
| 실행 | `pnpm test` (= `vitest run`). 계약 픽스처 갱신은 `pnpm vitest run -u <파일>` |

### 14.2 브라우저 E2E (Playwright — 리포 의존성 아님)

- Playwright는 `package.json`에 없다. `.claude/skills/verify/` 절차서와 `assets/`(스텁 백엔드 `stub-server.mjs` + 시나리오 `*.test.mjs` 약 30개: 세션 흐름, 리포트 편집기 클릭, Overview, 트레이스 그리드, 그리기 도구, 브랜드별 로고, 데일리 리포트 등)로 **로컬 수동 검증**에 쓴다. 스크래치 디렉터리에 `npm i playwright` 후 `node <script>`.
- 구동 규칙: 스텁 백엔드 포트 18080(`STUB_PORT`), 격리 vite `VITE_API_TARGET=http://localhost:18080 pnpm dev --port 3100 --strictPort`(프로세스 env가 `.env`를 덮음) → `curl :3100/api/me`로 스텁 응답 확인. **실백엔드(8080)·사용자 dev 서버(3000)는 재사용·종료 금지**, 끝나면 자기가 띄운 3100/18080만 종료. 스텁 계정은 `<STUB_USER>`/`<STUB_PASSWORD>`(일반)·비밀번호 변경 필요 계정 1개, 테스트 훅 `POST /api/__test/expire`·`/reset`(실행 간 멱등성 필수).
- 셀렉터 규칙: **페이지 루트 `.page-<slice>`로 스코프**(`.page-trace`, `.page-anomaly-score-dash` 등) — keep-alive로 숨김 탭 DOM이 남아 같은 클래스·문구가 중복 매칭되기 때문(추정, 규칙 자체는 스킬·운영 관례). 모달은 body 포털이라 `.modal` 스코프. 한국어 단언 전에 `addInitScript`로 `semi-language` 고정. 도움말 모달은 여러 쪽이라 "다음"을 끝까지 눌러 수집.

### 14.3 정적 품질 게이트

| 게이트 | 명령 | 기준 |
| --- | --- | --- |
| 타입 | `pnpm type-check` (`tsc --noEmit`) | 에러 0 |
| 린트 | `pnpm lint` (`eslint src`) | **경고 0**(FSD 경계·no-alert·no-explicit-any 포함) |
| 테스트 | `pnpm test` | 전부 통과 |
| i18n | `node scripts/check-i18n.mjs` | 키 패리티·한글 하드코딩 0 (로컬 규칙, CI 미포함) |
| 포맷 | `pnpm format:check` | (CI 미포함) |
| 규칙 스캔(`pre-mr-check` 스킬) | `git diff --name-only` 대상 grep | 색 하드코딩(`#hex`/`rgb(`), 컴포넌트의 `axios.`/`fetch(`, `any`, 딥 import, 인라인 쿼리 키, 이모지/직접 SVG |

### 14.4 CI 게이트 (`.gitlab-ci.yml`)

- MR(대상 `dev`) 파이프라인: `verify` 잡 = `type-check` → `lint` → `test`, `interruptible`(새 커밋 오면 이전 취소). GitLab의 "파이프라인 성공 시에만 머지" 설정과 결합해 **머지 게이트**.
- dev push 파이프라인: `verify_dev`(같은 3단계, `needs: []`로 배포와 병렬, 실패해도 배포를 막지 않는 **알림용**) — MR 파이프라인은 브랜치 끝에서만 돌아 "각자 통과·합치면 깨짐"을 못 잡고, merged results pipeline은 Premium 전용이라 CE에서 대신 쓰는 장치(2026-09-17 실사고 배경).

### 14.5 커밋/브랜치 규칙

- 형식(`.gitmessage`): `<type>(<scope>): <JIRA-KEY> <subject>` — Jira 키 필수(대문자+숫자-숫자, 예 `RSDSEP-35`), subject 50자 내외·마침표 없음·명령형. type: `feat, fix, docs, style, design, test, refactor, build, ci, perf, chore, rename, remove`. body에 이유, footer `Related/Fixes/BREAKING CHANGE`.
- 브랜치: **`dev` 직접 push 금지**. 개인/기능 브랜치(`feature/*`, 예 `feature/<개인 브랜치>`·`feature/RSDSEP-xxx-…`)에서 작업 → `dev` 대상 MR. 요청 없이 commit/push/merge 하지 않음(`AGENTS.md`). `dev` 머지 = 자동 배포(§15).
- 서식 전용 대규모 커밋은 `.git-blame-ignore-revs`에 등록(`git config blame.ignoreRevsFile .git-blame-ignore-revs`).

---

## 15. 빌드/배포

### 15.1 scripts

| 스크립트 | 명령 | 용도 |
| --- | --- | --- |
| `dev` | `vite` | 개발 서버 :3000, `/api` → `VITE_API_TARGET` 프록시 |
| `build` | `tsc && vite build` | 타입 검사 후 `dist/` 생성 |
| `type-check` | `tsc --noEmit` | |
| `test` | `vitest run` | |
| `preview` | `vite preview` | `dist` 미리보기 |
| `lint` | `eslint src` | |
| `format` / `format:check` | `prettier --write src` / `--check src` | |
| (node) | `node scripts/check-i18n.mjs` | i18n 검사 |
| (node) | `LOGIN_ID=<id> LOGIN_PW=<pw> [API_BASE=…] node scripts/create-dashboard-menus.mjs` | 대시보드 4메뉴를 공개 API로 생성(멱등, 1회성) |
| (CI) | `pnpm exec vitest run -c vitest.manifest.config.ts` | `scripts/ci/featureManifest.dump.ts`를 jsdom에서 실행해 `dist/feature-manifest.json`(`{features:[…]}`) 생성 |
| (CI) | `bash scripts/ci/feature-sync.sh <APP_URL> dist/feature-manifest.json` | SU 로그인(CSRF 쿠키→헤더) 후 `POST /api/features/sync` |

### 15.2 빌드 산출물

```
dist/
├─ index.html               # <script type=module src=/assets/index-<hash>.js>
├─ assets/index-<hash>.js   # 단일 엔트리 번들(코드 스플리팅 없음) + CSS·이미지 해시 파일
├─ docs/*.html              # public/docs 복사본(ERD·파이프라인 문서)
└─ feature-manifest.json    # CI 배포 잡에서만 추가 생성
```

### 15.3 서빙 방식

- **정적 파일을 nginx가 서빙**하고 같은 origin의 `/api`를 백엔드로 리버스 프록시한다(같은 origin이므로 `VITE_API_BASE_URL` 비움, 세션·CSRF 쿠키가 그대로 동작). nginx 설정 파일은 이 리포에 없고 인프라 스택에 있다. `main.tsx` 주석에 nginx 프록시 타임아웃이 3,600초(`proxy.conf`)라는 언급이 있다.
- Spring Boot static 리소스로 서빙하지 않는다.
- 클라이언트 라우팅이므로 nginx는 모르는 경로를 `index.html`로 돌려줘야 한다(`try_files … /index.html`) (추정 — 설정 파일 미확인, 딥링크·새로고침이 동작한다는 전제에서).
- 같은 빌드가 여러 도메인에 배포되며, 브랜드 표기(`brand.ts`)와 관리자 도구 노출(`internalHost.ts`)은 **런타임 hostname**으로 갈린다(빌드 분기 없음).

### 15.4 `.gitlab-ci.yml` 단계

```mermaid
flowchart TB
  subgraph MR["MR → dev"]
    V[verify: type-check · lint · test\ninterruptible]
  end
  subgraph PUSH["push → dev"]
    VD[verify_dev: type-check · lint · test\nneeds: [] (병렬·비차단)]
    BD[build_and_deploy\nresource_group 직렬화]
  end
  BD --> B1[pnpm build]
  B1 --> B2[vitest -c vitest.manifest.config.ts\n→ dist/feature-manifest.json, test -s]
  B2 --> B3[웹 루트 백업 tgz(KST 타임스탬프)\n최근 10개 유지]
  B3 --> B4[웹 루트 비우고 dist 복사\nchown 1000:1000]
  B4 --> B5[feature-sync.sh → POST /api/features/sync]
```

| 항목 | 값(일반화) |
| --- | --- |
| 이미지 | `node:20-bookworm` |
| 러너 | 그룹 러너 `<RUNNER_TAG>`(docker executor). 잡 컨테이너에 배포 볼륨 루트(`/deploy`)와 docker.sock 마운트 |
| workflow | MR(target=dev) 또는 dev push일 때만 파이프라인 생성 |
| before_script | `corepack enable && corepack prepare pnpm@9 --activate && pnpm install --frozen-lockfile --store-dir .pnpm-store` |
| 캐시 | key = `pnpm-lock.yaml` 파일, path `.pnpm-store/` |
| 변수 | `WEB_ROOT=<DEPLOY_ROOT>/nginx/html/semi`, `BACKUP_DIR=<DEPLOY_ROOT>/backup/ci/react`, `KEEP_BACKUPS=10`, `APP_URL=<STAGING_URL>`; masked `DSEP_SU_LOGIN`/`DSEP_SU_PASSWORD`=`<PLACEHOLDER>` |
| 배포 대상 | 검증 서버의 nginx 컨테이너 html 볼륨(`semi/`). 폐쇄망 고객사 반입 번들(`react.tar` → nginx html 볼륨)과 같은 경로 구조라 CI에서 검증된 산출물이 그대로 실장비 번들이 된다 |
| 실패 내성 | 첫 배포(백업 0개)에서 `ls`가 실패해도 `|| true`로 잡이 죽지 않게 함 |
| Feature 동기화 | 배포 직후 수행 — 안 하면 새 화면의 API 규칙이 DB에 없어 403 "Menu access is denied.". 서버는 규칙을 60초 경로 캐시 뒤 반영 |

### 15.5 Docker

리포에 `Dockerfile`·nginx 설정은 **없다**. CI 잡이 docker executor 컨테이너에서 돌고, 결과물을 호스트의 인프라 스택(nginx 컨테이너가 마운트한 볼륨)으로 복사하는 구조다. 롤백은 `BACKUP_DIR`의 `semi_<timestamp>.tgz`를 웹 루트에 풀어 되돌리는 방식(추정 — 백업 생성 단계만 CI에 있고 복원 스크립트는 없음).

---

## 16. 재구현 체크리스트

### 16.1 순서

1. **프로젝트 골격**: Vite 5 + React 18 + TS 5(strict) + pnpm 단일 패키지 워크스페이스(`packages: ['.']`, `pnpm add -w`). alias `@/*`. `define: {'process.env': {}}`. dev 3000 + `/api` 프록시.
2. **품질 도구**: ESLint flat config + `eslint-plugin-boundaries`(FSD 6레이어 규칙, resolver v3 절대 경로) + `no-alert`·`no-explicit-any`·`consistent-type-imports`, Prettier, vitest(jsdom, setup에서 모든 브라우저 목 + i18n en 고정).
3. **shared 기반**: `tokens.css`(Primitive→Semantic→Scale, 라이트/다크) + `global.css`(Tailwind v3, 토큰 매핑) + `index.html` 테마 선적용 스크립트; `theme.ts`(`useThemeStore`, `getChartColors`), `resetStores.ts`, `widgetLayoutStore.ts`, `pageActivity.ts`, `queryKeys.ts`, `access.ts`, `unsavedChangesGuard.ts`.
4. **i18n**: 27 ns × en/ko JSON, `resources.ts` 인라인, `initI18n`(기본 ko, `semi-language`), 조사 포매터, `translateMenuName`, `useLanguage`, `CustomTypeOptions`, `check-i18n.mjs`.
5. **HTTP**: axios 인스턴스(withCredentials·XSRF·30s), 응답 인터셉터(401 → 레지스트리 핸들러, `lastServerContactMs`), JSON/Blob/multipart 헬퍼, 에러 판별 유틸, `ApiErrorResponse`/DTO 타입.
6. **shared/ui**: Button·Badge·Field·Modal(포털·ESC 스택·포커스 트랩)·ConfirmModal·Toast·StateBlocks·QueryState·Panel·Page·DataTable(필터 임계 20·가상화)·SortableDataTable·Tabs·Toggle·Pagination·EChart(ResizeObserver resize)·WidgetBoard(RGL legacy, 12×24, auto/fullHeight)·AppTooltip·HelpButton/ManualContent.
7. **auth 엔티티**: `authApi`, `useAuthStore`, `AuthProvider`(부트스트랩·401 핸들러·유휴 감시·BroadcastChannel), `useAuth`(login+lastContext 복원·logout·changePassword·switchContext with `flushSync` → `clear`).
8. **엔트리/라우팅**: `main.tsx` Provider 순서(Query → Toast → Auth → Router), `createBrowserRouter([{path:'*'}])` + 내부 `<Routes>`, `ProtectedRoutes`/`LoginRoute`/`AdminSiteRoute`/`TenantDataRoute`/`BackendMenuView`(FEATURE_CODE_BY_PATH + PAGE_BY_FEATURE_CODE).
9. **메뉴/셸**: `menuApi.myMenus` → `MenuProvider`(path 맵·home) → `AppShell`(정적 nav vs DB 트리, SidebarTree, 사용자 메뉴: 컨텍스트 전환·전체화면·테마·언어·비밀번호·로그아웃), `ROUTES`·`STATIC_NAV_ITEMS`(그룹 연속성 테스트).
10. **워크스페이스 탭**: `tabStore`(sessionStorage·계정 파티션), `useWorkspaceTabs`(자동 등록·컨텍스트 전환·닫기 확인), `WorkspaceTabBar`(dnd-kit 가로 정렬), `WorkspaceTabView`(LRU 5, UNSAFE 컨텍스트 주입, suspended 프레임, 복귀 시 stale refetch `cancelRefetch:false`, resize 이벤트).
11. **features/target-selector** + **widgets/process-scope** (id↔code 해석, `isResolving`).
12. **Feature manifest**: `getFeatureManifest()`(i18n 지연 평가) + Feature 관리 화면 Sync + CI 덤프/동기화 스크립트.
13. **관리자 콘솔 CRUD 화면**: MasterDataManager 패턴(목록|상세, 첫 행 자동 선택, dirty 보호, 순서 저장, 2단계 저장 부분 실패 에러) → 계정·역할(메뉴 권한)·메뉴(트리 DnD)·법인/사이트·로그 화면.
14. **도메인 화면**: 대시보드 4종 → 트레이스 분석(버킷·차트 모드·useChartAxisZoom) → Data Chart → 비교 3화면 → FDC/DCOP → 데일리 리포트(TipTap) → 모델 4화면 → VOC.
15. **CI/배포**: GitLab CI(verify / verify_dev / build_and_deploy), nginx 정적 + `/api` 프록시 + SPA 폴백, 배포 후 Feature Sync.

### 16.2 함정(실제로 겪은 것 위주)

| # | 함정 | 대응 |
| --- | --- | --- |
| 1 | react-grid-layout/react-draggable가 `process.env` 참조·StrictMode 미지원 | `define: {'process.env': {}}`, StrictMode 미사용 |
| 2 | `eslint-import-resolver-typescript` v4는 boundaries 규칙을 조용히 무력화 | v3 고정 + `require.resolve` 절대 경로 |
| 3 | 컨텍스트 전환 후 이전 법인 데이터가 남음 | `invalidate`가 아니라 `queryClient.clear()`, `flushSync`로 user 교체·store 리셋 먼저, keep-alive 캐시 전체 삭제, 활성 화면은 한 프레임 비웠다가 재마운트 |
| 4 | 숨김 탭이 대형 응답을 영구 보유 → OOM | `subscribed: usePageActive()` + `gcTime 60s`, keep-alive 상한 5 (`enabled` 게이팅으로는 GC가 안 풀림) |
| 5 | 탭 빠른 왕복 시 좀비 다운로드 누적 | 복귀 refetch에 `cancelRefetch: false` (queryFn이 AbortSignal 미배선) |
| 6 | 숨김 화면의 `<Navigate>`가 라우터를 낚아챔 | 숨김 탭에 no-op navigator(`UNSAFE_NavigationContext`) 주입 |
| 7 | keep-alive 래퍼 요소 타입이 바뀌면 리마운트 | 활성/숨김 모두 같은 Provider 트리로 감쌈, value 객체 재사용 |
| 8 | 로그인 랜딩이 "컨텍스트 변경 사이클"로 오인돼 첫 탭 미등록 | 컨텍스트 키 ref를 마운트 시점 값으로 초기화 |
| 9 | 타임아웃을 재시도해 서버가 무거운 조회를 두 벌 완주 | `retry`에서 4xx·타임아웃 제외 |
| 10 | 로그아웃 시 만료 세션 401이 "세션 만료" 토스트로 | `setUser(null)`을 logout API 호출보다 먼저, 401 핸들러는 user 있을 때만 |
| 11 | 유휴 감시가 세션을 연장 | 서버 무통신 > 서버 timeout(3h)+1분일 때만 `/api/me` |
| 12 | `alert/confirm`이 전체화면 해제 | `no-alert` + ConfirmModal/Toast |
| 13 | 전체화면 요청을 await 뒤로 미룸 / F11 위에 이중 요청 | submit 동기 구간에서 요청, `isAnyFullscreen()`이면 생략, 실패 시 되돌림 |
| 14 | 모듈 스코프 `t()`로 언어 전환 미반영 | 호출 시점 평가(`getFeatureManifest()` 함수화) |
| 15 | 메뉴 이름을 바꿔도 화면 그대로 | `translateMenuName`은 DB 이름 우선, menu.json은 폴백 |
| 16 | 부분 manifest Sync → 누락 feature MISSING·403 | 항상 전체 manifest, 낡은 번들에서 Sync 금지 |
| 17 | feature rule이 다른 feature에만 있어 403 | `X-Menu-Id`가 가리키는 feature에 필요한 rule을 모두 중복 등록 |
| 18 | 관리자 콘솔 화면에 DB 메뉴 행 없음 | ADMIN/ADMIN 컨텍스트 `tb_co_menu` 행 필수(없으면 "메뉴 없음" 또는 X-Menu-Id 누락) |
| 19 | 테넌트에 콘솔 화면 메뉴 행 생성 | `AdminSiteRoute`는 차단이 아니라 폴백 — COMPANY/FEATURE/ERROR_LOG는 테넌트 메뉴 금지 |
| 20 | refetch마다 편집 draft가 덮임 | draft 동기화는 객체 identity가 아니라 id 키 기준, 순서 dirty 중엔 같은 id 집합 refetch 무시(`useOrderedList`) |
| 21 | 2단계 저장 중 2단계 실패 후 재시도가 중복 생성 | 전용 에러(`RoleAssignError`/`PermissionSaveError`)로 edit 모드 전환 + invalidate |
| 22 | 해석 중을 "미선택"으로 표시 | `target.isResolving` 동안 로딩, 진짜 미선택은 EmptyBlock |
| 23 | echarts 이벤트를 `chart.on`으로 붙여 임시 인스턴스에 걸림 | `onEvents` prop, 인스턴스는 호출 시점 `getInstance()` |
| 24 | 커서를 `style.cursor`로 → zrender가 덮어씀 | `data-cursor` 속성 + CSS `!important` |
| 25 | `resetKey`에 옵션 객체 → 확대가 렌더마다 풀림 / `min`/`max` 키를 빼면 병합으로 남음 | 화면이 정한 안정 키, 조건부 축 범위는 `null` |
| 26 | LTTB `sampling`이 스파이크 삭제 | 화면에서 솎지 않음, 서버 다운샘플(구간 최저·최고) |
| 27 | WidgetBoard 기본 배치를 바꿨는데 기존 사용자에게 안 보임 | pageKey `-vN` 증가 |
| 28 | `.control` 폭을 Tailwind `w-*`로 못 줄임 | 인라인 `style={{width}}` |
| 29 | 세션 필요한 이미지를 `<img src="/api/...">` | `getBlob` → objectURL → revoke |
| 30 | multipart JSON part를 문자열로 → BE `@RequestPart` 파싱 실패 | `new Blob([JSON.stringify(x)], {type:'application/json'})` |
| 31 | 테스트가 ko 기본 언어로 영어 셀렉터 깨짐 | setup에서 en 고정, 기본 언어는 순수 함수 테스트 |
| 32 | MR 파이프라인 통과 후 dev에서 type-check 깨짐 | dev push 후 `verify_dev` 병렬 알림 |

### 16.3 기준 커밋 이후 변경(작성 중 발생, 참고)

이 문서는 `855af1b5`(2026-10-02) 기준이다. 작성 도중(2026-10-06) 같은 브랜치가 `dev`를 받아 `f79a4be9`(2026-10-05)로 전진했다(683 파일 변경). 재구현 시 반영을 검토할 주요 차이:

- 새 화면/Feature: `MODEL_SETUP`(`/sensor/model-setup`, 모델 셋업 콘솔), `VOC_MANAGE`(`/system/voc-management`), `PARQUET_VIEW`(`/system/parquet-viewer`, `entities/parquet-viewer`), `RAW_FILE`(`/system/raw-files`, `/api/collectors/mirror/**`).
- 새 슬라이스: `features/filter-group`, `features/help-resources`(메뉴별 도움말 첨부, `entities/menu/api/helpAttachments.ts`), `features/master-names`, `features/trace-handoff`(TTTM 인계 URL 빌더 이동), `widgets/step-select`(`StepSelectPanel` 이동).
- 기존 화면: 장비 챔버 제거 모달(`ChamberRemovalModal`), 스텝 이상 웨이퍼(`isWaferStepError` — 판정 버킷·비율 분모에서 제외), FDC 그래프 잡(`/api/sensor/fdc/graph-jobs`), DCOP 파라미터 한계, VOC 댓글 스레드·관리·안내, 용어 통일(설비 → 장비, 미추론 → N/A). 테스트 파일 376개.
