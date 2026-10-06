# SCK 프론트엔드 (sck-server-react) 구성 문서

| 항목 | 값 |
|---|---|
| 대상 리포 | `sck-server-react` — `sck-server-spring/src/main/frontend` 에 git 서브모듈로 포함 |
| 기준 브랜치·커밋 | `develop` @ `d83f0e75` (2026-10-02) |
| 작성일 | 2026-10-06 |
| 짝 문서 | [backend.md](./backend.md) (Spring 백엔드), [database.md](./database.md) (DB 스키마), [README.md](./README.md) (프로젝트 개요·전체 흐름) |
| 제품명 | DUTCHBOY OSAT (`index.html` title) — 반도체 OSAT 테스트 장비 실시간 모니터링·규칙/알람·분석 플랫폼 |

> 이 문서의 모든 경로는 프론트 리포 루트(`src/main/frontend/`) 기준이다. 비밀값·사내 호스트는 `<PLACEHOLDER>` 로 바꿨다.

## 목차

1. [개요](#1-개요)
2. [디렉터리 구조](#2-디렉터리-구조)
3. [설정/환경](#3-설정환경)
4. [앱 부트스트랩](#4-앱-부트스트랩)
5. [라우팅](#5-라우팅)
6. [인증/로그인](#6-인증로그인)
7. [사용자 관리 화면](#7-사용자-관리-화면)
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

- Spring Boot 백엔드(`/api/**`)를 호출하는 SPA. 로그인 → 메뉴 트리 수신 → **탭(멀티 문서) 기반 프레임**에 화면을 띄우는 업무 포털 구조.
- 실시간 데이터는 **SSE(EventSource)** 로 받아 React Query 캐시에 직접 끼워 넣는다 (REST 재조회 최소화).
- 화면 종류: 실시간 모니터링(장비 격자/목록·장비 상세·레이아웃 편집·테스트 이력), 장비 관리(장비·모듈 배포·파일·파일 탐색기·규칙·알람), 애널리틱스 스페이스(위젯 대시보드), 시스템 관리 16종(사용자·권한·메뉴·코드·이력 등).

### 1.2 기술 스택 (package.json 기준)

| 분류 | 라이브러리 | 버전 | 비고 |
|---|---|---|---|
| 프레임워크 | react / react-dom | ^18.2.0 | |
| 언어 | typescript | ^5.9.3 | `strict: true` |
| 번들러 | vite | ^7.0.0 | `@vitejs/plugin-react` ^4.7, `vite-tsconfig-paths` ^4.3 |
| 라우터 | react-router-dom | ^6.21.1 | **HashRouter** 사용 |
| 전역 상태 | zustand | ^5.0.8 | `persist` + sessionStorage |
| 서버 상태 | @tanstack/react-query | ^5.8.7 | devtools ^5.17 (주석 처리) |
| HTTP | axios | ^1.7.9 | 단일 인스턴스 `client` |
| 테이블 | @tanstack/react-table ^8.15, @tanstack/react-virtual ^3.13, react-virtuoso ^4.6 | | 레거시 `Table` + 신규 `DataTable` 공존 |
| 스타일 | styled-components ^6.1 (devDependency 로 선언), tailwindcss ^3.3.5, postcss ^8.5, autoprefixer | | 테마 토큰은 styled-components, 유틸 클래스는 Tailwind |
| 차트 | chart.js ^4.5 (+ chartjs-plugin-annotation ^3.1, chartjs-plugin-zoom ^2.2, react-chartjs-2 ^5.3), billboard.js ^3.14, d3 ^7.9, echarts ^6.1 | | CLAUDE.md 의 "Plotly 3" 표기는 package.json 에 없음 |
| 레이아웃 | react-resizable-panels ^3.0, react-grid-layout ^2.2, react-dnd ^16 + html5-backend | | 패널 분할·위젯 그리드·탭 DnD |
| i18n | i18next ^23.7, react-i18next ^14.0, i18next-browser-languagedetector ^7.2 | | ko / en |
| UI 보조 | react-toastify ^10, react-tooltip ^5.29, react-icons ^4.12, react-spinners ^0.17 | | |
| 내보내기 | xlsx-js-style ^1.2 | | 표 → 엑셀 |
| 유틸 | lodash ^4.17, culori ^4 (색 계산), web-vitals | | |
| 테스트 | vitest ^4.0, @testing-library/react ^13.4, @testing-library/jest-dom ^6.9, jsdom ^27, msw ^2.7 | | |
| 문서화 | storybook ^10.2 (`@storybook/react-vite`, addon-a11y / addon-docs / addon-onboarding, msw-storybook-addon) | | |
| 린트 | eslint ^8.57, eslint-plugin-prettier ^5, eslint-config-prettier ^9, eslint-plugin-storybook | | `.eslintrc.json` |

- 패키지 매니저: **npm** (`package-lock.json` 추적). `name: "react-starter"` (CRA 에서 Vite 로 이관된 흔적 — README.md 는 CRA 원본 그대로).
- 지원 브라우저: 별도 `browserslist` 없음. `tsconfig.target ES2020`. 개발 서버 주석 기준 Chrome/Firefox 최신 가정 (`vite.config.ts`).

---

## 2. 디렉터리 구조

```
frontend/
├── index.html                 # Vite 진입점. <div id="root"> + /src/index.tsx
├── vite.config.ts             # dev 서버(3000, TLS 선택, /api 프록시), build.outDir=build, vitest 설정
├── tsconfig.json              # ES2020·bundler·strict·paths "@/*"
├── tailwind.config.js         # 커스텀 색(primary/alert/sub) — 라이트 fallback, GlobalStyle 이 테마로 덮음
├── postcss.config.js
├── .eslintrc.json / .prettierrc
├── .storybook/{main.ts,preview.tsx}
├── .gitlab-ci.yml             # notify 스테이지만 활성(빌드/배포 주석)
├── sonar-project.properties   # SonarQube(lcov)
├── docs/db/*.sql              # 화면 추가 시 메뉴/프로그램/권한 등록 SQL 사본 (ROSAT-765/842/937)
├── e2e/                       # 비어 있음
├── public/                    # favicon.png, MSW worker
└── src/
    ├── index.tsx              # 부트스트랩(QueryClient, Provider 트리, MSW)
    ├── App.tsx                # Toast + GlobalStyle + DndProvider + HashRouter
    ├── GlobalStyle.tsx        # Tailwind 클래스 테마 오버라이드
    ├── ResetStoreContext.ts   # resetAllStores 를 Context 로 제공
    ├── setupTests.ts          # jest-dom, ResizeObserver 폴리필
    ├── styled.d.ts / vite-env.d.ts
    ├── apis/        (107)     # apiUrlConfig.ts(URL 사전), apiKeyConfig.ts(쿼리 키), queries/**, mutations/**
    ├── components/  (119)     # alarm, charts, common, datatable(레거시 Table), equipment, frame, graph, layout, modal, rule, table(DataTable), tree
    ├── context/     (3)       # AuthContext.tsx, ThemeContext.tsx
    ├── features/    (330)     # login/, header/, sideMenu/, tabMenu/, frame/(Frame.tsx + 화면 모듈)
    │   └── frame/
    │       ├── Frame.tsx                   # programId → 화면 컴포넌트 매핑(라우터 역할)
    │       ├── RealtimeDashboard/          # 실시간 모니터링·장비 상세·레이아웃 편집·테스트 이력
    │       ├── EquipmentManagement/        # 장비 목록·모듈 배포·파일·파일 탐색기·규칙·알람
    │       ├── AnalyticsSpace/             # 위젯 대시보드(스페이스/위젯/드릴다운/ECharts)
    │       └── system/                     # 시스템 관리 16 서브프레임
    ├── hooks/       (26)      # SSE 훅, 세션(useIdleSession), 탭 히스토리, 폼, 표 관련
    ├── i18n/                  # index.ts, locales/{en,ko}.json
    ├── mocks/                 # MSW 핸들러(user/menu/system) — VITE_MODE=base
    ├── models/      (27)      # interfaces/, types/, defaults/
    ├── pages/       (6)       # AuthorizationRouter, Login, Home, popup/
    ├── stores/      (26)      # base/(auth, menu, select, layout), system/, 도메인 스토어, resetAllStores
    ├── styles/      (10)      # theme.ts(light/dark 토큰), *Styles.ts(styled 컴포넌트)
    ├── utils/       (30)      # axiosInstance, authSync, clearClientSession, sideMenuHelper, tableHelper …
    └── __tests__/   (111)     # vitest 단위 테스트
```

(괄호 숫자 = 파일 수, 2026-10-02 기준)

### 2.1 아키텍처 패턴·규칙 (CLAUDE.md + 코드)

- **레이어**: `pages`(라우트) → `features`(화면 단위) → `components`(재사용 UI) / `apis`(react-query 훅) / `stores`(zustand) / `models`(타입). 화면 모듈은 `features/frame/<Feature>/{components,hooks,index.ts}` 배럴 구조.
- **import alias**: `@/` → `src/` (`tsconfig.paths` + `vite-tsconfig-paths`). Storybook 은 `components`, `features` … 폴더명 alias 를 별도 등록(`.storybook/main.ts`).
- **네이밍**: 컴포넌트 PascalCase, 훅 `use*`, 스토어 `use<Feature>Store`, 쿼리 훅 `use<Feature>Query`/`use<X>List`, 뮤테이션 훅 `use<X>Mutation`, 타입 `I<Name>`, Props `<Component>Props`, 훅 반환 `Use<Hook>Return`.
- **표 규칙**: 신규 화면은 `components/table/DataTable` 사용. `components/datatable/core/Table`(TanStack Table + 편집 셀 + RowType 추적)은 시스템 관리 화면 등 기존 코드 유지보수용.
- 주석·커밋 메시지는 한국어. 커밋 형식 `type(scope): 한글 설명` (예: `feat(equipment-state): 장비 목록 그리드 뷰 추가`). Jira 키 `ROSAT-###` 를 scope 자리에 쓰는 커밋이 다수.

---

## 3. 설정/환경

### 3.1 환경변수 (`src/vite-env.d.ts`)

| 변수 | 용도 | 비고 |
|---|---|---|
| `VITE_MODE` | `'base'` 이면 **MSW 목 모드**: `mocks/browser.ts` 워커 기동, axios `withCredentials=false`, 메뉴 헤더 미주입, 표 편집 버튼 비활성 | 그 외 값은 실서버 모드 |
| `VITE_GRAFANA_TOKEN` | Grafana 패널 임베드 토큰(`utils/constants.ts GRAFANA_TOKEN`) | `<GRAFANA_TOKEN>` |

- `.env*` 파일은 리포에 추적되지 않음(`.gitignore` 에 `.env.local` 류만 명시, 실제 파일 없음). 필요 시 `.env.local` 에 위 두 값을 둔다.
- 백엔드 주소는 환경변수가 아니라 **동일 출처 상대경로 `/api`** 로 호출한다(`apis/apiUrlConfig.ts prefix='/api'`). 개발은 Vite 프록시, 운영은 nginx 가 같은 출처로 묶는다.

### 3.2 vite.config.ts 핵심

```ts
// vite.config.ts (요지)
server: {
  port: 3000, open: true,
  https: certs/localhost-{key,cert}.pem 이 있을 때만 TLS(HTTP/2)   // SSE 7종이 HTTP/1.1 소켓 6개 한도를 먹어 느려지는 문제 대응
  proxy: {
    '/api': { target: 'http://localhost:8080', changeOrigin: true,
      configure: proxy => proxy.on('proxyReq', (proxyReq, _req, res) => {
        proxyReq.removeHeader('origin');                 // Spring CORS 허용 목록(http://localhost:3000)과 https 출처 충돌 회피
        res.once('close', () => { if (!res.writableEnded && !proxyReq.destroyed) proxyReq.destroy(); }); // 닫힌 SSE 의 백엔드 연결 정리
      }) },
    '/grafana': { target: 'http://localhost', changeOrigin: true },
  },
},
build: { outDir: 'build', sourcemap: true },
define: { 'process.env': {}, global: 'globalThis' },
test: { globals: true, environment: 'jsdom', setupFiles: ['./src/setupTests.ts'], css: true, include: ['src/**/*.{test,spec}.{js,jsx,ts,tsx}'] }
```

- 자체 서명 인증서 생성(선택): `openssl req -x509 -newkey rsa:2048 -nodes -days 3650 -keyout certs/localhost-key.pem -out certs/localhost-cert.pem -subj "/CN=localhost" -addext "subjectAltName=DNS:localhost,IP:127.0.0.1,IP:::1"` — `certs/` 는 gitignore.

### 3.3 tsconfig / 린트

- `tsconfig.json`: `target ES2020`, `lib [ES2020, DOM, DOM.Iterable]`, `moduleResolution bundler`, `jsx react-jsx`, `strict true`, `noUnusedLocals false`, `allowJs true`, `types [vite/client, vitest/globals, node]`, `paths {"@/*": ["./src/*"]}`, `include [src]`.
- ESLint: `eslint:recommended` + `@typescript-eslint/recommended` + `plugin:prettier/recommended` + `plugin:storybook/recommended`. `@typescript-eslint/no-unused-vars: error`, `react/prop-types off`. Prettier `endOfLine: auto`.
- Tailwind `content: ['./index.html','./src/**/*.{js,jsx,ts,tsx}']`, 확장 색 `primary.{base,select,title,point}`, `alert.{base,line}`, `sub.{point,separator,line,word}` (라이트 값, 다크는 `GlobalStyle.tsx`/`index.css` 의 `[data-theme='dark']` 오버라이드).

---

## 4. 앱 부트스트랩

`src/index.tsx` → `src/App.tsx` → `src/pages/AuthorizationRouter.tsx` 순서.

```tsx
// src/index.tsx (요지)
import './i18n';                                   // i18next 초기화(부수효과)
async function enableMocking() { if (import.meta.env.VITE_MODE === 'base') { const { worker } = await import('./mocks/browser'); return worker.start(); } }

const queryClient = new QueryClient({
  defaultOptions: { mutations: { onError: 전역 토스트(401/403 은 로그인 URL 외 무시) } },
  queryCache: new QueryCache({
    onSuccess: (d, q) => q.meta?.success && toast.success(...),
    onError: 전역 토스트(401/403 은 axios 인터셉터가 처리하므로 제외),
  }),
});

<QueryClientProvider client={queryClient}>
  <ResetStoreContext.Provider value={{ resetAllStores }}>
    <ThemeProvider>            {/* context/ThemeContext: styled ThemeProvider + data-theme + Chart.js 기본값 */}
      <AuthProvider>           {/* context/AuthContext: login/logout/remainingMinutes, 탭 간 동기화 */}
        <App />
      </AuthProvider>
    </ThemeProvider>
  </ResetStoreContext.Provider>
</QueryClientProvider>
enableMocking().then(() => root.render(<AppWrapper />));
```

```tsx
// src/App.tsx
<div>
  <Toast />                        {/* react-toastify 컨테이너 */}
  <GlobalStyle />
  <DndProvider backend={HTML5Backend}>
    <HashRouter future={{ v7_startTransition: true }}>
      <AuthorizationRouter />
    </HashRouter>
  </DndProvider>
</div>
```

- 전역 에러 바운더리는 없음. 쿼리/뮤테이션 오류는 QueryCache 콜백에서 토스트(`NewLineContent` 로 줄바꿈 표시).
- 초기 데이터 로드 순서: 쿠키 플래그 확인 → `GET /api/me` → `Home` 마운트 시 `GET /api/menu/main`, `GET /api/menu/my-menu`, 알람 SSE 구독.

---

## 5. 라우팅

HashRouter 이므로 URL 은 `/#/...` 형태. 화면 전환은 라우트가 아니라 **탭 스토어** 로 이루어지며 URL 은 `/#/` 하나에 머문다(`hooks/useTabHistorySync.ts` 가 탭 전환을 브라우저 히스토리에 미러링).

| path | 컴포넌트 | 보호 | 설명 |
|---|---|---|---|
| `/` | `AuthInitializer` → `pages/Home` | O | 헤더 + 사이드바 + 탭 프레임 |
| `/login` | `pages/Login` | X | 로그인. 이미 로그인 상태면 `/` 로 |
| `/popup/bins` | `AuthInitializer` → `pages/popup/BinsPopup` | O | 별도 브라우저 창용 위젯(빈 분포). 동일 출처라 쿠키 인증 공유 |
| `/popup/bin-trend` | `AuthInitializer` → `pages/popup/BinTrendPopup` | O | 빈 추세 팝업 |

404 라우트 없음(미정의 해시는 빈 화면). 코드 스플리팅(lazy) 미사용 — 모든 화면이 `Frame.tsx` 에 정적 import.

### 5.1 라우트 가드 (`pages/AuthorizationRouter.tsx`)

```tsx
function AuthInitializer({ children }) {
  const isLoggedInFlag = document.cookie.split(';').some(c => c.trim() === 'isLoggedIn=true');
  const { isLoading, isError, data } = useMe({ enabled: isLoggedInFlag });   // GET /api/me
  if (!isLoggedInFlag) return <Navigate replace to='/login' />;
  if (isLoading) return null;
  if (isError || !data?.userInfo?.userId) return <Navigate replace to='/login' />;
  return children;
}
```

- 접근 토큰은 **HttpOnly 쿠키 `accessToken`** 이라 JS 가 읽을 수 없고, 서버가 함께 내려주는 **비-HttpOnly 플래그 쿠키 `isLoggedIn=true`** 로 "로그인했을 것" 을 판단한 뒤 `/api/me` 로 확정한다.

---

## 6. 인증/로그인

### 6.1 방식 요약

| 항목 | 값 | 근거 |
|---|---|---|
| 토큰 | JWT, **쿠키 `accessToken`(HttpOnly)** 로만 전달. 응답 본문에는 토큰이 없다(프론트 `LoginResponse` 타입에 `accessToken` 필드가 남아 있으나 값은 오지 않음) | `apis/mutations/useLoginMutation.ts`, 백엔드 `LoginController` |
| 로그인 플래그 | 쿠키 `isLoggedIn=true` (JS 가 읽음) | `pages/AuthorizationRouter.tsx` |
| 전송 | `axios withCredentials: true` (base 모드 제외) — 헤더 주입 없음 | `utils/axiosInstance.ts` |
| 토큰 수명 | 로그인 응답 `expTime`(초) → `authStore.expTime = now + expTime*1000`, `tokenLifetimeMs` | 서버 기본 `jwt.access-token-validity-in-seconds` |
| 세션 저장 | `authStore` (zustand persist → **sessionStorage `auth-storage`**): `userInfo`, `expTime`, `tokenLifetimeMs` | `stores/base/authStore.ts` |
| 갱신 | `GET /api/refresh` — 수동(연장 버튼) + 자동(활동 후 10분 경과 시) | `apis/queries/useRefresh.ts`, `hooks/useIdleSession.ts` |
| 로그아웃 | `GET /api/logout` → 서버가 쿠키 삭제 → 클라이언트 정리 | `apis/mutations/useLogoutMutation.ts` |
| 서버 사용자 대조 | 응답 헤더 `x-user-id` 와 현재 `userInfo.userId` 비교 | `utils/authSync.ts RESPONSE_USER_HEADER` |

### 6.2 API 계약

| API | 요청 | 응답 | 프론트 처리 |
|---|---|---|---|
| `POST /api/login` | `{ userId, password, releaseLockSnos?: number[] }` (`IUser` + 이 브라우저가 쥔 룰 편집 잠금 번호) | `{ userInfo: IUserInfo, expTime: number(유효 초) }` + Set-Cookie `accessToken`(HttpOnly), `isLoggedIn=true` | 아래 6.3 |
| `GET /api/me` | – (쿠키) | `{ userInfo: IUserInfo, expTime: number(만료 epoch ms — 로그인 응답과 단위가 다름), authorityIds?: string[] }` | `authorityIds` 를 `userInfo` 에 병합해 저장, `expTime` 저장, 다른 탭에 `session` 알림 |
| `GET /api/refresh` | – | `{ expTime(유효 초) }` + 두 쿠키 재설정 | `expTime`·`tokenLifetimeMs` 갱신, 활동 기록, 다른 탭 알림, 토스트 `toast.loginSessionExtended` |
| `GET /api/logout` | – | 200 (쿠키 만료) | 성공·실패 모두 `clearClientSession()` |
| `POST /api/user-info/change` | `{ oldPassword, newPassword }` (`IChangePassword`) | `{ message }` | 토스트 성공 메시지 (헤더 프로필 메뉴의 비밀번호 변경) |
| `PUT /api/user-info/reset` | `IUserCode` (관리자 화면의 사용자 행) | 200 | `alert(userInfo.passwordResetSuccess)` — 사용자 비밀번호 초기화 |

```ts
// models/interfaces/loginInterface.ts
export interface IUser { userId: string; password: string; [key: string]: string }
export interface IUserInfo {
  department: string; userId: string; userName: string;
  authorityIds?: string[];      // /me 에서 병합
  deptName?: string; positionName?: string;
  // employeeNo 는 2026-09-17 제거 — 사번이 초기 비밀번호라 로그인 응답/JWT/sessionStorage 로 내보내지 않는다
}
```

실패 응답은 `{ status, message }` (`AxiosErrorResponse`, 메시지는 `Accept-Language` 에 따라 ko/en). 로그인 실패는 사용자 없음 404, 비밀번호 불일치 400, 퇴직·차단 403 이며, 로그인 URL 이므로 인터셉터의 세션 정리 대상이 아니고 전역 토스트가 `message` 를 그대로 보여준다.

### 6.3 로그인 성공 처리 (`apis/mutations/useLoginMutation.ts`)

```ts
onSuccess: (data: LoginResponse) => {
  queryClient.clear(); sessionStorage.clear();            // 이전 사용자 캐시/persist 잔존 방지(HashRouter 라 리로드가 없다)
  if (data.userInfo?.userId) forgetOtherAccountRuleLocks(data.userInfo.userId);
  const expTime = Date.now() + data.expTime * 1000;
  setExpTime(expTime); setUserInfo(data.userInfo); setTokenLifetime(data.expTime * 1000);
  markActivity(userInfo?.userId);                          // 비활동 타이머 시작점
  announceAuth({ type: 'login', userId, expTime });        // 다른 탭에 알림
  if (!document.fullscreenElement) document.documentElement.requestFullscreen().catch(() => undefined);  // 로그인 직후 전체화면 진입(공장 단말 UX)
  window.location.href = '/#/';                            // 해시 이동(리로드 없음 → 전체화면 유지)
}
```

### 6.4 HTTP 클라이언트 인터셉터 (`utils/axiosInstance.ts`)

```ts
const REQUEST_TIMEOUT_MS = 5_000;            // 기본. "영원히 대기" 방지(SSE 가 소켓을 점유한 사고 대응)
export const LONG_READ_TIMEOUT_MS = 60_000;  // 대시보드 목록·미니맵·장비 상세 통계
export const FILE_TRANSFER_TIMEOUT_MS = 300_000; // 패치 업로드·파일 다운로드
export const client = axios.create({ withCredentials: !isBase, timeout: REQUEST_TIMEOUT_MS });

client.interceptors.request.use(cfg => { cfg.headers['Accept-Language'] = i18n.language || 'en'; return cfg; });
if (!isBase) {
  client.interceptors.request.use(cfg => {              // 서버 로깅 AOP(SystemLoggingAspectJoinPoint)가 읽는 메뉴 헤더
    const sel = useSelectStore.getState().selectSideMenu;
    const menuId = sel?.baseMenuId ?? (parseEquipmentDetailMenuId(sel?.menuId) ? sel.programId : sel?.menuId);
    if (menuId != null) cfg.headers.menuId = String(menuId);
    if (sel?.programId != null) cfg.headers.programId = String(sel.programId);
    return cfg;
  });
  client.interceptors.response.use(res => {             // 응답 사용자 대조(/me 제외)
    const served = res.headers?.['x-user-id']; const me = useAuthStore.getState().userInfo?.userId;
    if (served && !res.config.url?.endsWith('/api/me') && me && decodeURIComponent(served) !== me)
      clearClientSession({ broadcast: false, clearCookies: false, notice: i18n.t('toast.otherTabLogin', { user: served }) });
    return res;
  });
  client.interceptors.response.use(res => res, err => {
    const isTimeout = err.code === 'ECONNABORTED' || err.code === 'ETIMEDOUT';
    err.message = isTimeout ? i18n.t('common.requestTimeout') : err.response?.data?.message || 'An error occurred';
    const status = err.response?.status;
    if ((status === 401 || status === 403) && !location.hash.includes('/login') && !err.config.url?.includes('/api/login'))
      clearClientSession({ notice: err.message });      // 로그인 화면으로 + 사유 표시
    return Promise.reject(err);
  });
}
```

- 토큰 자동 재발급(401 → refresh → 재시도)은 **없다**. 401/403 은 곧바로 로그아웃 처리. 만료 전 갱신은 아래 6.5 의 비활동 타이머가 담당.

### 6.5 세션 타이머·자동 로그아웃 (`hooks/useIdleSession.ts`)

| 상수 | 값 | 의미 |
|---|---|---|
| `REFRESH_INTERVAL_MS` | 10분 | 활동이 있고 지난 발급 후 10분 지나면 `GET /api/refresh` |
| `DEFAULT_TOKEN_LIFETIME_MS` | 250분 | 서버 응답이 없을 때 기본 토큰 수명 |
| `idleMsFor(lifetime)` | `max(60s, lifetime − 10분)` = 240분 | **비활동 자동 로그아웃** 시간 |
| `ACTIVITY_EVENTS` | `pointerdown, keydown, wheel, touchstart` | 사람의 조작만 활동으로 인정(마우스 이동·SSE·폴링 제외) |
| `sck-last-activity:<userId>` | localStorage | 탭 공용 마지막 활동 시각(5초에 1회 기록) |

- 1초마다 `deadline = min(lastActivity + idle, expTime)` 계산 → `remainingMinutes` (사이드바 하단 "N분 남음"). 5분 남으면 토스트, 0이 되면 `GET /api/logout` 후 `clearClientSession({ notice: toast.sessionExpired })`.
- 사이드바 "연장" 버튼(`features/sideMenu/SideAuth.tsx`)은 `useRefresh(count)` 쿼리를 카운터 증가로 트리거.

### 6.6 다중 탭 동기화 (`utils/authSync.ts`, `context/AuthContext.tsx`)

- 채널: `BroadcastChannel('sck-auth-sync')`, 없으면 localStorage 키 `sck-auth-sync` 의 storage 이벤트.
- 메시지: `{type:'login', userId, expTime}`, `{type:'session', userId, expTime, tokenLifetimeMs?}`, `{type:'logout', userId?}`.
- 수신 처리(로그인된 탭만): `logout` → 이 탭도 정리(쿠키는 안 지움, 재방송 안 함); 다른 `userId` 의 login → 이 탭 물러남(쿠키는 새 계정 것이므로 유지) + 안내 문구; 같은 계정 → 만료 시각만 갱신.

### 6.7 클라이언트 세션 정리 (`utils/clearClientSession.ts`)

```ts
clearClientSession({ redirectToLogin = true, clearCookies = true, broadcast = true, notice }) {
  resetAllStores(); sessionStorage.clear();
  if (clearCookies) { document.cookie = 'accessToken=; Max-Age=0; path=/;'; document.cookie = 'isLoggedIn=; Max-Age=0; path=/;'; } // HttpOnly accessToken 은 서버 /logout 이 지운다
  toast.dismiss();
  if (broadcast) announceAuth({ type: 'logout', userId });
  if (notice) sessionStorage.setItem('sck-login-notice', notice);   // 로그인 화면이 ✕ 누를 때까지 표시
  if (redirectToLogin) window.location.replace('/#/login');
}
```

### 6.8 로그인 화면 (`pages/Login.tsx`, `features/login/*`)

- 좌: 로고 박스(`LoginLogoBox`, `assets/login_logo.png`, 문구 `login.mainText/subText`), 우: `LoginForm`(제목 `login.title` = "DUTCHBOY", 입력 `userId`·`password`, 버튼 `login.loginButton`). 배경 `assets/login_bg.png`. 로그인 화면은 항상 **라이트 테마 고정**(`styles/loginStyles.ts`).
- 폼 상태: `hooks/useForm.ts` (`initialValues`/`validate`/`onSubmit`, `touched`, `errors`) + `LoginFormContext`. 검증: 빈 아이디/비밀번호 → 한글 메시지 하드코딩(`loginValidate`, i18n 미적용).
- 진입 시 `clearAllToasts()`, `sessionStorage['sck-login-notice']` 가 있으면 제목 아래 경고 박스(세션 만료·다른 탭 로그인 등 사유).
- `userInfo.userId && isLoggedIn 쿠키` 이면 `nav('/')`.

### 6.9 로그인 시퀀스

```mermaid
sequenceDiagram
  participant U as 사용자
  participant L as Login.tsx
  participant M as useLoginMutation
  participant S as Spring /api
  participant A as authStore(sessionStorage)
  participant H as Home(AuthInitializer)
  U->>L: ID/PW 입력, 제출
  L->>M: login({userId,password,releaseLockSnos?})
  M->>S: POST /api/login (withCredentials)
  S-->>M: 200 {userInfo, expTime} + Set-Cookie accessToken(HttpOnly), isLoggedIn=true
  M->>A: queryClient.clear(); sessionStorage.clear(); set userInfo/expTime/tokenLifetime
  M->>M: markActivity, announceAuth('login'), requestFullscreen
  M->>H: location.href = '/#/'
  H->>H: cookie isLoggedIn=true 확인
  H->>S: GET /api/me
  S-->>H: {userInfo, expTime, authorityIds}
  H->>A: userInfo(+authorityIds) 갱신
  H->>S: GET /api/menu/main, GET /api/menu/my-menu
  H->>S: EventSource /api/alarms/unconfirmed/sse
  Note over H: 이후 모든 요청: Cookie + Accept-Language + menuId/programId 헤더
```

### 6.10 권한 기반 UI 분기

- `userInfo.authorityIds` (권한 ID 배열) 로만 판정. `utils/adminAuthority.ts` 에 `ADMIN_AUTH_IDS`(레이아웃 편집·삭제 관리자, 서버 설정 `layout.admin-authority-ids` 와 동일해야 함)와 `ALARM_VISIBLE_AUTH_IDS`(알람바·아이콘 노출) 집합을 둔다. `isLayoutManageable(layout, userId, authorityIds)` = 소유자 또는 관리자.
- **최종 방어는 서버(403)**. 프론트 판정은 버튼 활성/표시용. 메뉴 접근 권한은 서버가 `/api/menu/main` 에서 사용자 권한에 맞는 메뉴만 내려주는 방식(권한별 메뉴 매핑 화면, §8).

---

## 7. 사용자 관리 화면

시스템 관리 하위 "User Management"(`programId: userInfo`, 메뉴 `8000200`). 파일: `features/frame/system/userInfo/{UserInfo,UserInfoSearch,UserInfoTable}.tsx`.

### 7.1 구성

- 상단 검색 바(`FrameInfo`): 사용자유형(콤보 `CMB0022`), 아이디, 이름, 부서명, 재직구분(콤보 `CMB0023`). **입력 즉시 300ms 디바운스 검색** + 검색 버튼. 상태는 `systemConfigurationStore.searchUserMap['userInfo']`.
- 본문: 레거시 편집 그리드 `Table`(`editable`, `draggable`, 무한 스크롤). 상단 버튼 박스(`TableButtonBox`): 엑셀 내보내기 / 추가 / 삭제 / 저장(저장 전 `confirm`).
- 부서 셀 클릭 → 부서 검색 모달(`Modal param=DEPARTMENT_PARAM`) → `deptCode/deptName` 반영 + `rowType=UPDATE`.

### 7.2 컬럼 (`UserInfoTable.tsx`)

| 컬럼(필드) | 셀 | 규칙 |
|---|---|---|
| status | `StatusCell` | 행 상태 아이콘(ADD/UPDATE/DELETE) |
| `userId` | `TextCell` | 저장 후 수정 불가(`unmodifiable`), 필수 |
| `employeeNo` | `TextCell` | 사번(초기 비밀번호로 쓰임), 수정 불가, 필수 |
| `userName` | `TextCell` | 필수 |
| `deptName` (`deptCode`) | `ModalCell` | 부서 모달 선택, 필수 |
| `positionCode` → `positionName` | `SelectCell` | 콤보 `CMB0024` |
| `responsibilityCode` → `responsibilityName` | `SelectCell` | 콤보 `CMB0025` |
| `holdOfficeDttCode` → `holdOfficeDttName` | `SelectCell` | 콤보 `CMB0023` (재직구분) |
| `blockYn` | `CheckInputCell` | 차단 여부 |
| RESET | `DisplayButton` | `confirm` 후 `PUT /api/user-info/reset` (비밀번호 초기화) |

- 새 행 기본값: `defaultUser` + 검색 중인 `userTypeCode` + 각 콤보 첫 값.
- 저장: `saveTableData(initialData, ['userId','employeeNo','userName','deptCode'])` 가 `rowType ≠ NORMAL` 행만 추려 필수키 검사 → `POST /api/user-info` (`IUserCode[]`, 각 행에 `rowType` 1/2/3) → `refetchQueries(['system','user-info'])`.

### 7.3 API 매핑

| Method | Path | 파라미터/본문 | 응답 | 훅 |
|---|---|---|---|---|
| GET | `/api/user-info` | query: `ISearchUserCode` + `page`(0부터), `fetchSize`(100, `utils/constants.ts FETCHSIZE`) | `UserType[]` (한 페이지) | `useUserList` (useInfiniteQuery, 마지막 페이지 < fetchSize 면 종료) |
| GET | `/api/user-info` | `fetchSize: 0` → 전량 | `UserType[]` | `useAllUserList` (드롭다운용) |
| POST | `/api/user-info` | `IUserCode[]` (rowType 포함) | 200 | `useUserMutation` |
| PUT | `/api/user-info/reset` | `IUserCode` | 200 | `usePasswordMutation` |
| POST | `/api/user-info/change` | `IChangePassword` | `{message}` | `useChangePasswordMutation` |
| GET | `/api/department` | `{deptName, useYn}` | `IDepartmentCode[]` | `useDepartmentCodeList` |
| GET | `/api/search/combo` | `{comboCode}` | 옵션 목록 | `useSearchComboList` 훅(코드 → `{value,label}`) |

```ts
// models/interfaces/system/systemManagementInterface.ts
export interface ISearchUserCode { userId; userName; deptCode; deptName?; parentDeptCode?; holdOfficeDttCode; userTypeCode; responsibilityCode }
export interface IUserCode extends ISearchUserCode { relationCompanyCode?; positionCode?; positionName?; responsibilityName?; employeeNo?; blockYn?; holdOfficeDttName?; userDttCode?; rowType? }
// models/types/commonType.ts
export const enum RowType { NORMAL = 0, ADD = 1, UPDATE = 2, DELETE = 3 }
```

### 7.4 편집 그리드 공통 프로토콜 (시스템 화면 전부 동일)

1. 조회 결과를 `initialData`(편집본)와 `backupData`(원본) 로 보관.
2. 셀 편집 → `editCellValue` 가 값 변경 + `rowType` 을 `UPDATE`(새 행이면 `ADD` 유지) 로 표시.
3. 삭제 버튼 → 기존 행이면 `rowType=DELETE` 표시(새 행이면 제거).
4. 저장 버튼 → `saveTableData` 가 `rowType≠0` 인 행만 모아 필수키 검증(`toast.requiredFieldsEmpty`) → `POST <자원>` 배열 전송. 서버는 `rowType` 별로 INSERT/UPDATE/DELETE 분기.
5. 성공 시 쿼리 무효화/재조회.

---

## 8. 메뉴/권한

### 8.1 메뉴 데이터 모델 (`models/interfaces/menuInterface.ts`)

```ts
export interface ICommonMenu { parentMenuId: string; level: number; menuId: string; menuName: string; menuUsage: string; path: string[]; }
export interface ISideMenu extends ICommonMenu { programId: string; programName: string; programUsage: string; sortOrder: number; }
```

- `level`: 0 = HOME(사이드바 미표시), **1 = 상위메뉴(스페이스 그룹)**, 2 이상 = 하위 항목. 서버 뷰가 계산해 내려준다. 상수 `HEADER_LEVEL=1`, `TITLE_LEVEL=2`(축소 레일의 정렬 기준).
- `programId` 가 있는 행은 레벨과 무관하게 **리프(클릭 → 탭)**, 없는 level ≥ 2 행은 중간 그룹 타이틀(예: 시스템 스페이스 아래 History 그룹 `8300000`).
- `path`: 루트부터의 경로 배열(정렬키와 menuId 가 번갈아 든 형태 — 코드 주석). 헤더 소속 판정은 `path.includes(header.menuId)`.
- `menuUsage === 'Y'` 인 행만 사이드바에 표시.

실제 메뉴 ID 체계(i18n `menu.*` 키, 7자리; DB `tb_co_mnu_m.mnu_id`):

| menuId | 이름 | level/programId |
|---|---|---|
| 0 | Home | 0 |
| 1000000 | Monitoring Space | 1 |
| 1000100 / 1000200 / 1000300 | Realtime Monitoring / Equipment Detail / Edit Layout | `stateList` / `stateDetail` / `editLayout` |
| 1500000 | Test History | `testHistory` |
| 2000000 | Equipment Management | 1 |
| 2000100 / 2000101 / 2000102 / 2000105 | Management List / Module Deploy / File List / File Explorer | `managementList` / `moduleDeploy` / `fileList` / `fileExplorer` |
| 3000000 | Control Space | 1 |
| 3000103 / 3000104 | Rule / Alarm | `ruleList` / `alarm` |
| 4000000 | Analytics Space | 1 |
| 4000100 / 4000200 | Analytics Space / Analytics Space 2 | `analyticsSpace` / `analyticsSpace2` |
| 8000000 | System Space | 1 |
| 8000100~8000600 | Department / User / Permission / Permission-Menu Mapping / User Permission / User Permission Mapping | `department` / `userInfo` / `authority` / `menuAuthority` / `userAuthority` / `userAuthMapping` |
| 8200100~8201000 | Program / Menu / Authority / Code / Search 그룹, Common Code Type / Common Code | `program` / `menu` / … / `commonCodeType` / `commonCode` |
| 6000300 / 6000400 / 6000700 | Search Popup / Search Combo / System Message Management | `searchPopup` / `searchCombo` / `systemMessage` |
| 8300000, 8300100~8300300 | History 그룹, Login / System Usage / Error History | `loginHistory` / `systemHistory` / `errorHistory` |

(그룹 ID 는 백만 단위, 리프는 그룹 + 100 단위. 근거: `docs/db/ROSAT-842_analytics_space_menu.sql`)

### 8.2 메뉴 로드 → 사이드바 흐름

```mermaid
flowchart LR
  A[Home 마운트] --> B[GET /api/menu/main<br/>useMenuList]
  B --> C[menuStore.separateMenus<br/>headerMenu=level1<br/>sideMenu=level>1 && usage Y]
  C --> D[탭 복원: sessionStorage 의 tabMenu 를<br/>최신 메뉴로 치환, 최대 20개]
  A --> E[GET /api/menu/my-menu<br/>useMyMenu → menuStore.myMenu]
  C --> F[Side.tsx: headerGroups 계산<br/>접힘 상태·검색어 반영]
  F --> G[SideHeaderItem / SideTitleItem / SideChildItem]
  G -->|클릭| H[selectTargetMenu type=button]
  H --> I[menuStore.tabMenu 삽입 + selectStore.selectSideMenu]
  I --> J[Frame.tsx: tabFrame[programId]]
```

- `GET /api/menu/main` → 사용자 권한에 맞는 평면 메뉴 목록(`ISideMenu[]`). `retry:false`, 포커스/마운트 재조회 없음.
- `GET /api/menu/my-menu` → 즐겨찾기(내 메뉴). `POST /api/menu/my-menu { userId, menuId }` 로 토글(`useMyMenuMutation`, 성공 시 `my` 키 무효화).
- 사이드바 상단 `[메뉴 | 내 메뉴]` 카테고리(`SideMenuCategory`, `selectStore.selectMenu = 'system' | 'my'`).
- 접힘 상태는 "닫힌 것" 목록으로 저장(`collapsedHeaderMenus`, `collapsedGroupMenus`, sessionStorage persist) → 기본값 전부 펼침. 탭 선택 시 `expandAncestors` 가 조상만 펼친다.
- 검색(`SideSearch`): 후보 = 번역명 + 영문 원본명 + 약자(대문자만, `getSideMenuInitials`). 걸린 항목·조상은 보이고 자손은 원래 접힘 상태.
- 축소(레일 60px) 상태는 같은 계층을 **약자**로 그린다(`SIDE_RAIL_WIDTH=60`, `getSideMenuRailAlign`). 펼침 폭 `TARGET_WIDTH=220`.
- 라벨: `menu.<menuId>` 번역이 있으면 번역, 없으면 `menuName`. 한국어 모드에선 영문 원본을 부제로 함께 표시(`resolveSideMenuLabel`).
- `programId === 'stateDetail'`(장비 상세) 는 목록·검색에서 제외(대시보드에서 장비 클릭으로만 열림).

### 8.3 탭 시스템

- 스토어: `menuStore.tabMenu: ISideMenu[]`(최대 `MAX_TAB_MENU_COUNT=20`), `selectStore.selectSideMenu`(활성 탭), `selectHeaderMenu`(활성 탭의 상위메뉴, 기본 `'100000'`).
- `selectTargetMenu(target, getMenuStore, getSelectStore)` (`stores/base/selectStore.ts`): `type: 'search'`(항상 새 탭 삽입) / `'button'`(없으면 활성 탭 뒤에 삽입) / `'tab'`(선택만). 상한 초과 시 `alert(sideMenu.maxMenusAlert)`.
- `features/tabMenu/Tab.tsx`: react-dnd 로 순서 변경, 닫기(미저장 변경 있으면 `confirm`), "모두 닫기"(활성 탭만 남김). 탭 라벨은 `menu.<menuId>` 번역 → 없으면 `menuName`.
- `Frame.tsx`: `tabFrame: Record<programId, JSX>` 표에서 활성 탭의 `programId` 로 화면을 고른다. `key={selectSideMenu.menuId}` 로 탭마다 재마운트. **탭이 비면 로고만 표시.** `FLUID_PROGRAM_IDS`(`testHistory`) 만 반응형, 나머지는 `FRAME_WIDTH=1440` 최소 폭.
- 장비별 상세 탭: `menuId = 'stateDetail::<eqptPk>'` 합성(`utils/equipmentDetailTab.ts`), `programId='stateDetail'`, `baseMenuId` 에 원래 메뉴 ID 를 둬 헤더 전송 시 사용. 탭 라벨은 장비명(`eqptNm`).
- 브라우저 뒤로/앞으로 ↔ 탭 전환: `hooks/useTabHistorySync.ts` 가 `navigate('.', {state:{sckTab:{menu, seq, nonce}}})` 로 히스토리 엔트리를 쌓고 POP 시 그 탭으로 전환(닫힌 탭은 재오픈). `sessionStorage sck-nav-nonce / sck-nav-seq`.
- 메뉴·프로그램 식별자는 모든 API 요청 헤더(`menuId`, `programId`) 로 실려 서버 접근 로그에 남는다(§6.4).

### 8.4 메뉴 관리 화면 (`programId: menu`, `features/frame/system/menu/`)

- 좌: 메뉴 트리(`components/tree/Tree`, `GET /api/menu?domainCode&useYn&menuName` → `convertDataToTree(parentMenuId→menuId)`), 검색 조건: 도메인(콤보 `CMB0027`, 기본 `'AIBIZ'`), 사용여부, 메뉴명.
- 우: 선택 노드의 **하위 메뉴 편집 그리드**(`GET /api/menu?domainCode&parentMenuId&useYn`).

| 컬럼 | 셀 | 규칙 |
|---|---|---|
| `menuId` | `TextCell` | 필수, **정확히 7자**, 저장 후 수정 불가 |
| `parentMenuName` | 읽기 전용 | 선택 트리 노드 |
| `menuName` | `TextCell` | 필수 |
| `programName`(`programId`) | `ModalCell` | 프로그램 검색 모달(`PROGRAM_PARAM`) |
| `connectionDttCode` | `SelectCell` | 옵션 `WEB` 만 |
| `sortSequence` | `NumberCell` | 필수 |
| `menuIndicateYn` / `privateInfoIndicateYn` / `useYn` | `CheckInputCell` | 표시/개인정보표시/사용 |

- 저장 → `POST /api/menu` (`IMenu[]`) → `['system','menu']` 전체와 `my` 무효화(사이드바 즉시 반영).

```ts
export interface IMenu { menuId; parentMenuId; parentMenuName; menuName; programId; programName; menuParamValue; connectionDttCode; sortSequence; menuIndicateYn; privateInfoIndicateYn; nonLoginPermissionYn; domainCode; useYn; rowType }
```

### 8.5 권한 관리 (`programId: authority`)

- `GET /api/authority?authId&authName` → `IAuthority[]`, 편집 후 `POST /api/authority` (`IAuthority[]`).

```ts
export interface IAuthority { authId; authName; authDescription; useYn; sortSequence; wholeRelatedCoUseYn; unitaryDeptAuthYn; unitaryUseDeptCode?; rowType }
```

### 8.6 권한별 메뉴 매핑 (`programId: menuAuthority`)

- 좌 `MenuAuthorityTable`: 권한 목록(`GET /api/authority`), 행 선택 → `searchAuthorityMenu.authId`.
- 우 `MenuAuthorityMenuTable`: `GET /api/menu-authority?authId&systemDttMenu` → **전체 메뉴 트리 + 권한 플래그** (`IAuthorityMenu[]`, `level`, `parentMenuId`, `TreeCell` 로 들여쓰기).

| 컬럼 | 의미 |
|---|---|
| `inquiryAuthYn` / `updateAuthYn` / `printAuthYn` | 조회/수정/출력 권한 체크박스. **부모를 토글하면 모든 자손에 전파**(`getDescendantMenuIds`) |
| `inquiryRangeDttCode` | 조회 범위(콤보 `CMB0018`), 자식이 있는 행은 비활성 |

- 저장 → `POST /api/menu-authority` (`IAuthorityMenu[]`, `inquiryRangeDttCode` 빈 값은 `null`) → `menu-authority`·`menu main`·`my` 무효화 + 활성 쿼리 즉시 재조회 → 사이드바가 바로 갱신된다.
- 권한 ID 는 서버 설정 및 프론트 상수(§6.10)에서도 참조되므로 운영 중 변경 시 함께 맞춘다.

### 8.7 사용자 권한 매핑 (`programId: userAuthMapping`)

- 좌: 부서 트리(`GET /api/department`), 상단 검색(사용자유형·재직구분·아이디·이름·부서명).
- 중앙 위: 사용자 목록(`GET /api/user-info`, 무한 스크롤). 선택 → `ISearchAuth {userId, deptCode?}`.
- 중앙 아래: **부여 권한**(`GET /api/user-authority/assigned?userId`) ◀▶ **부여 가능 권한**(`GET /api/user-authority/possible?userId`). 화살표 버튼(`UserAuthMappingSwitch`)이 `POST /api/user-authority/assigned` 에 **전체 부여 목록**(`IAssignedAuthorityMenu[]`, 추가 행은 `rowType: ADD`)을 보낸다. 최소 1개 권한 유지(`messages.requireAtLeastOneAuthority`).

### 8.8 사용자 권한 관리 (`programId: userAuthority`)

- `GET /api/user-authority?<ISearchUserCode>` → `IUserAuthority[]` (사용자 행 + 권한 ID 별 플래그 컬럼 `ATCO010…ATCO090`). 저장 → `POST /api/user-authority` (`IUserAuthorityMenuWithFlag[] = {userId, authId, flag, rowType}`).

### 8.9 새 화면을 추가해 메뉴에 노출시키는 절차

1. **화면 구현**: `features/frame/<Feature>/` 에 컴포넌트·훅·배럴 작성(§2.1 패턴). API 는 `apis/apiUrlConfig.ts` 에 URL 키, `apis/apiKeyConfig.ts` 에 쿼리 키 추가.
2. **Frame 매핑**: `features/frame/Frame.tsx` 의 `tabFrame` 에 `'<programId>': <Component />` 추가. (이 매핑이 없으면 탭은 열리지만 프레임이 빈 화면 — 코드 주석)
3. **i18n**: `src/i18n/locales/{en,ko}.json` 의 `menu.<menuId>` 에 라벨 추가(없으면 DB `mnu_nm` 폴백).
4. **DB 등록**(백엔드/DBA 또는 시스템 화면): ① 프로그램 `tb_co_pgm_m(pgm_id=<programId>, pgm_nm, sys_dtt_cd, use_yn)` ② 메뉴 `tb_co_mnu_m(mnu_id 7자리, hrk_mnu_id=그룹, mnu_nm, pgm_id, srt_sqn, mnu_idct_yn, use_yn, dmn_cd='AIBIZ', cnn_dtt_cd='WEB')` ③ 권한-메뉴 `tb_co_mnu_ath_r` 에 노출할 권한마다 행 추가. 예시 SQL: `docs/db/ROSAT-842_analytics_space_menu.sql`(ON CONFLICT DO NOTHING 로 재실행 안전). 화면으로는 **프로그램 관리 → 메뉴 관리 → 권한별 메뉴 매핑** 순서.
5. 재로그인(또는 메뉴 쿼리 무효화) 하면 `GET /api/menu/main` 에 새 리프가 포함되어 사이드바에 나타난다.

---

## 9. 레이아웃/공통 컴포넌트

### 9.1 App shell (`pages/Home.tsx`)

```
┌ Header (42px, TARGET_HEIGHT) ──────────────────────────────────────────┐
│ [로고+사이드 접기] [탭 바(TabMenuBox, 가로 스크롤·DnD)] [⛶][사용자][로그아웃] │
├ Side (220px / 레일 60px) ┬ Frame ───────────────────────────────────────┤
│ [메뉴|내 메뉴]            │ FrameContainer > FrameWrapper(key=menuId)   │
│ 검색                      │   tabFrame[programId]                        │
│ 헤더그룹 ▸ 타이틀 ▸ 리프   │                                              │
│ "N분 남음" [연장]          │                                              │
└──────────────────────────┴──────────────────────────────────────────────┘
+ AlarmOverlay(장비 알람 창) + GlobalRuleWindows(룰 상세/폼 전역 창) — 탭 전환에도 유지
```

- 헤더 우측 `HeaderUserInfo`: 프로필 드롭다운(이름/부서/직급, 비밀번호 변경 폼 `HeaderUserPasswordForm`: 현재/새/확인, 불일치·동일 검증, 최소 4자 안내), 언어 선택(`LanguageSelector`), 테마 토글.
- `layoutStore.sideOpen`(사이드바 축소), `infoPanelOpen`(대시보드 우측 정보 패널).

### 9.2 공통 컴포넌트 카탈로그

| 컴포넌트 | 용도 | 주요 props | 파일 |
|---|---|---|---|
| `DataTable` | 신규 표준 표(가상 스크롤, 정렬/필터/컬럼 폭·순서 개인화, 체크박스, 그룹 헤더) | `columns: Column<T>[]`, `data`, `keyExtractor`, `onRowClick`, `isRowHighlighted`, `sortState/onSort`, `filterValues/filterOptions/filterSelections`, `onLayoutChange`, `tableHeight`, `rowHeight` | `components/table/DataTable.tsx` |
| `Table` (레거시) | 편집 그리드(TanStack Table, RowType 추적, 셀 컴포넌트, 헤더 필터, 무한 스크롤) | `title`, `initialData`, `backupData`, `setData`, `columns`, `fetchScroll`, `editable`, `draggable`, `headless`, `customHeader` + `TableContext`(handleAdd/Remove/Save/Export, currentData) | `components/datatable/core/Table.tsx` |
| 셀 | `TextCell, NumberCell, SelectCell, CheckInputCell, ModalCell, DateCell, StatusCell, TreeCell, DisplayButton, ReadOnly*Cell, UploadCell, ImageCell, IconCell, ExpressionCell` | 컬럼 `meta`(`unmodifiable`, `required`, `options`, `renderKey`, `onClick`, `justify`, `min/maxLength`) | `components/datatable/cells/` |
| `Tree` | 좌측 계층 트리 선택 | `data`, `currentData/setCurrentData`, `nodeKey`, `parentKey`, `renderKey`, `isLoading` | `components/tree/` |
| `Modal`, `ModalSearch`, `ModalTable`, `ModalButton` | 검색 팝업(부서/프로그램 등, `param=DEPARTMENT_PARAM/PROGRAM_PARAM`) | `param`, `handleSelect`, `onClose` | `components/modal/` |
| `SimpleModal`, `DraggableOverlay` | 단순 모달 / 드래그 가능한 오버레이 창 | | `components/common/` |
| `FrameInfo`, `FrameInfoInput`, `FrameInfoSelect`, `FrameInfoRadio` | 화면 상단 검색 바 | `handleSearch`, `title`, `options`, `onChange` | `components/frame/` |
| `FrameEdit*`, `FrameUserForm` | 상세 편집 폼 필드 | | `components/frame/` |
| `ResizablePanelLayout`, `PanelContainer`, `PanelContent`, `DropdownPanelLayout` | 좌우 분할·패널 | `panels: [{id, defaultSize, content}]`, `minWidth` | `components/layout/` |
| `PageHeader`, `PageTitle`, `StateDisplay`(Loading/Error/Empty), `LoadingSpinner` | 페이지 상태 표시 | | `components/common/` |
| `CommonButton`, `ActionButton`, `ToolButton`, `AutocompleteSelect`, `TruncatedText`, `CustomTooltip`, `ContentTab`, `FileUpload(Modal)`, `LanguageSelector`, `Toast`, `NewLineContent`, `WheelHint`, `Grafana` | 기본 UI | | `components/common/` |
| `EquipmentStatus`, `AlarmRealtime`, `EquipmentAlarmBar`, `ModuleList`, `ModuleStatus`, `ConnectionStatus` | 장비 상태 표시 | | `components/equipment/` |
| 차트 | `BinsDistributionChart`, `BinTrendChart`, `SiteToSiteYieldChart`, `TestTimeAndIndexChart`, `YieldInfoChart`, `LotInfoChart`, `ChartExternalTooltip` … | | `components/charts/` |
| `AlarmOverlay`, `UserCanPicker`, `GlobalRuleWindows` | 알람/룰 전역 창 | | `components/alarm/`, `components/rule/` |

- 확인창: 네이티브 `window.confirm` / `window.alert` 사용(저장 확인, 탭 닫기, 비밀번호 초기화, 탭 상한). 토스트: `react-toastify` + `hooks/useCustomToast.ts`(`TOAST_IDS` 로 중복 방지).
- 표 헤더 필터: 레거시 `Table` 은 `HeaderFilter`(체크 목록, `getFilterCheckList(backupData)`), `DataTable` 은 `filterOptions`(원본 기준 값 목록) + `useColumnFilterSort`.

---

## 10. API 레이어/서버 상태

### 10.1 URL 사전 (`apis/apiUrlConfig.ts`)

`callApiUrl(key)` = `'/api' + urlObject[key]`. 전체 키:

| 영역 | 키 → 경로 |
|---|---|
| 인증 | `login /login`, `me /me`, `logout /logout`, `refreshToken /refresh` |
| 시스템 | `department /department`, `userInfo /user-info`, `program /program`, `menu /menu`, `authority /authority`, `menuAuthority /menu-authority`, `userAuthority /user-authority`, `commonCodeType /common/code-type`, `commonCode /common/code`, `searchPopup /search/popup`, `searchCombo /search/combo`, `systemMessage /system/message`, `loginHistory /login-history`, `systemHistory /system-history`, `errorLogHistory /error-history` |
| 사용자 설정 | `userBinSettings /user-bin-settings`, `userGridSettings /user-grid-settings`, `widgetPosition /widget-position`, `testTimeChartSetting /test-time-chart-setting`, `gridColumnSettings /grid-column-settings` |
| 규칙/알람 | `rules /rules`, `ruleCatalog /rules/catalog`, `alarms /alarms`, `alarmSse /alarms/unconfirmed/sse` |
| 애널리틱스 | `analyticsSpaces /analytics/spaces` (+`/{spcSno}/widgets`), `analyticsDataSources /analytics/data-sources`, `analyticsParameters /analytics/parameters`, `analyticsFilterOptions /analytics/filter-options`, `analyticsQuery /analytics/query`, `analyticsQueryExport /analytics/query/export` |
| 장비 | `equipment /manager/equipment`, `equipmentList /manager/equipment/list`, `equipmentListPage /manager/equipment/list/page`, `equipmentSearchOptions /manager/equipment/search-options`, `equipmentMonitoringList /manager/equipment/real-time`, `equipmentMonitoringSse /manager/equipment/real-time/sse`, `managementSse /manager/equipment/management/sse`, `equipmentGridDelta /manager/equipment/grid/delta`, `equipmentGridHeatmapPreview /manager/equipment/grid/heatmap-preview`, `equipmentFileExplorer /manager/equipment/file-explorer`, `fileExplorerBrowse /manager/file-explorer`, `fileHistory /manager/file-history`, `equipmentFiles /manager/mngr-file/equipment`, `patchEquipmentList /manager/mngr-patch/equipment/list`, `patchModuleList /manager/mngr-patch/module/list`, `patchModuleVersion /manager/mngr-patch/module/version`, `patchExecute /manager/mngr-patch/module/patch`, `layout /manager/layout`, `parmHist /manager/parm-hist` |

### 10.2 쿼리 키 (`apis/apiKeyConfig.ts`)

- 시스템 도메인은 `['system', '<자원>', ...조건객체]` 형태(`userInfoApiKeys.list(search)`, `menuApiKeys.main()/my()/parent()/child()`, `authorityApiKeys`, `menuAuthorityApiKeys`, `userAuthorityApiKeys.assigned/possible`, `historyApiKeys.login/system/error`, `comboApiKeys`, `popupApiKeys`, `userGridSettingsApiKeys` 등).
- 도메인: `['event-log', ...]`, `['alarm', 'list', mgEqptSno, page, scope]`, `['rule', 'detail'|'timeline'|'equipments'|'by-equipment', ...]`, 대시보드 `MONITORING_QUERY_KEYS = ['equipmentMonitoring','equipmentMonitoringList','equipmentMonitoringGrid']`.
- 조건 객체를 키에 그대로 넣으므로 같은 조건이면 캐시 재사용(검색 바 디바운스와 결합).

### 10.3 훅 패턴

```ts
// 조회: <X>Base(options) + 기본 옵션 래퍼. 시스템 화면은 공통적으로
useQuery({ queryKey, queryFn: () => client.get(url, { params }).then(r => r.data), retry: false, refetchOnMount: false, refetchInterval: false, refetchOnWindowFocus: false, enabled: 조건 });
// 목록 무한 스크롤: useInfiniteQuery({ initialPageParam: 0, getNextPageParam: (last, pages) => last.length < FETCHSIZE ? undefined : pages.length }) + hooks/useInfiniteScroll(query, param, 300)
// 변경: useMutation({ mutationFn: data => client.post(url, data) }) + onSuccess: invalidate/refetch 키
```

- 응답은 래퍼 없이 **DTO/배열 그대로**(`response.data`). 메시지성 응답은 `{ message }`. 오류는 `{ status, message }`.
- 사용자별 화면 설정(격자 뷰, 선택 레이아웃, 대시보드 개인화 JSON, 룰 검색 축 등)은 `GET/PUT /api/user-grid-settings` (`UserGridSettings` 타입, 서버는 JSON 원문을 해석하지 않음).

### 10.4 실시간(SSE)

`new EventSource(url)` (쿠키 자동 전송) + 지수 백오프 재연결(1s → 최대 30s) + `queryClient.setQueryData` 로 캐시 직접 갱신. 연결 수가 많아 HTTP/1.1 에서는 소켓 한도(6) 문제가 있어 운영/개발 모두 **HTTP/2(TLS)** 를 전제한다.

| 훅 | 엔드포인트 | 이벤트 | 용도 |
|---|---|---|---|
| `hooks/useDashboardSSE` | `/api/manager/equipment/real-time/sse` | `CONNECTED, CONNECTION_STATUS, YIELD_UPDATE, FILE_METADATA, DETAIL_METADATA, ALARM_FIRED, ALARM_CONFIRMED, ALARM_DELETED` | 실시간 모니터링 목록/격자 |
| `hooks/useManagementSSE` | `/api/manager/equipment/management/sse` | `CONNECTED, MODULE_STATUS, EQUIPMENT_CONTROL, EVENT_LOG_ALERT` | 장비/모듈 관리, 파일 탐색기(`useControllerSSE` 가 MODULE_STATUS 필터) |
| `hooks/useAlarmSSE` | `/api/alarms/unconfirmed/sse` | `CONNECTED`, 알람 변경 페이로드 `{kind, count, lastAlrmSno, reload, changed[], openByEqpt[]}` | **모듈 싱글턴**(참조 카운트, Home 이 1회 구독). 서버가 0.5초 창으로 바뀐 행을 밀어주고 캐시에 끼워 넣음. 재접속 시 `seenAlrmSno/seenEvtSno` 이후 증분 조회 |
| `RealtimeDashboard/hooks/useEquipmentSSE`, `useEquipmentTrackingSSE`, `useProductYieldComparison` | 장비 상세·추적·제품 수율 | | 장비 상세 탭 |
| `apis/queries/equipmentstate/useEventLogSSE` | 이벤트 로그 | | 장비 상세 이벤트 로그 |

---

## 11. 상태 관리

| 스토어 | 상태 | 주요 액션 | persist |
|---|---|---|---|
| `base/authStore` | `userInfo`, `expTime`, `tokenLifetimeMs` | `setUserInfo/setExpTime/setTokenLifetime/clearAuth` | sessionStorage `auth-storage` |
| `base/menuStore` | `appMenu, headerMenu, sideMenu, myMenu, tabMenu` | `separateMenus, setTabMenu, addTabMenu, removeTabMenu, resetMenuStore` | sessionStorage `menu-storage` |
| `base/selectStore` | `selectMenu('system'|'my')`, `selectSideMenu`, `selectHeaderMenu`, `collapsedGroupMenus`, `collapsedHeaderMenus` | `setSelect*`, `toggle*Collapse`, `resetSelectStore`; 헬퍼 `selectTargetMenu`, `expandAncestors` | sessionStorage `select-storage` |
| `base/layoutStore` | `sideOpen`, `infoPanelOpen` | 토글 | 없음 |
| `alarmStore`, `equipmentStateStore`, `equipmentManagementStore`, `sharedEquipmentStore`, `ruleStore`, `ruleWindowStore`, `patchStore`, `moduleVersionStore`, `overlayWindowStore`, `analyticsSpaceStore`, `detailPanelStore`, `equipmentDetailLayoutStore`, `binsDistributionViewStore`, `waferBinColorStore` | 도메인 화면 상태(선택 장비, 검색 조건, 창 위치, 뷰 설정) | 각 `reset<X>Store` | 일부 sessionStorage/localStorage(`overlayWindowStore` viewState 는 localStorage, `binsDistributionViewStore` 는 로그아웃에도 유지) |
| `system/formManagementStore`, `historyManagementStore`, `standardInformationStore`, `systemConfigurationStore` | 시스템 화면 검색 조건·목록·현재 행(`Map<화면키, …>` 로 화면별 분리) | getter/setter | 없음 |

- `stores/resetAllStores.ts`: 로그아웃/세션 정리 시 모든 스토어 리셋 + `clearPersistedViewState()`.
- 폼 상태: 로그인/비밀번호는 `hooks/useForm`, 시스템 그리드는 `Table` 내부 + `TableContext`, 복잡 폼(룰 작성)은 각 feature 훅.

---

## 12. 스타일/테마/i18n

- **테마**(`styles/theme.ts`): `lightTheme`/`darkTheme` (`mode`, `radii`, `colors.{appBg, surface*, text*, border*, brand*, info/success/warning/danger*, chart*, tooltipBg, overlay*}`), `getTheme(mode)`, 사이드바 전용 팔레트 `getSideMenuTheme` (라이트에서도 남색 배경).
- `context/ThemeContext.tsx`: localStorage `sck-theme-mode`, `<html data-theme>` 속성, styled `ThemeProvider`, Chart.js 전역 기본색 동기화 및 기존 차트 인스턴스 갱신, 팝업 창과 storage 이벤트로 동기화.
- styled-components 가 주 스타일 수단(`styles/*Styles.ts`). Tailwind 는 레이아웃 유틸(`flex`, `h-full` …)과 소수 색 클래스. 반응형은 사실상 없음(프레임 최소 폭 1440px, 일부 화면만 fluid).
- **i18n**(`src/i18n/index.ts`): i18next + `LanguageDetector(order: localStorage → navigator, caches: localStorage 'i18nextLng')`, `fallbackLng: 'en'`, 단일 네임스페이스 `translation`, 리소스 `locales/en.json`, `locales/ko.json`. 섹션: `common, login, header, sideMenu, tabMenu, equipmentState, equipmentManagement, fileStatus, equipmentStatus, equipmentConnectionStatus, moduleStatus, layout, system, searchPopup, userInfo, department, authority, authorization, menuAuthority, menuManagement, program, searchCombo, commonCode, loginHistory, systemUsageHistory, errorHistory, systemMessageManagement, validation, messages, toast, alert, confirm, table, menu, charts, equipmentPatch, alarm, equipmentRule, fileExplorer, analyticsSpace, ruleTimeline, ruleDetail, ruleType, ruleForm, userCan, ruleHistory, rulePending`. 메뉴 라벨은 `menu.<menuId>`. `Accept-Language` 헤더로 서버 메시지도 같은 언어로 받는다. 팝업 창과는 storage 이벤트로 언어 동기화.
- 서버 메시지 코드(`INFO_00140` 등)는 `utils/getAlertMessage` 로 매핑.

---

## 13. 그 외 도메인 화면

| 화면(programId) | 폴더 | 목적 / 주요 구성 | 주요 API·실시간 |
|---|---|---|---|
| 실시간 모니터링 `stateList` | `RealtimeDashboard/RealtimeDashboard` | 장비 목록/격자(가상화 `VirtualizedEquipmentGrid`), 미니 히트맵·웨이퍼맵, 상태/수율 필터, 레이아웃 선택, 핀 고정, 개인화(`useDashboardPref` → `user-grid-settings.dashboardPref`) | `GET /manager/equipment/real-time`(60s 타임아웃), `grid/delta`, `grid/heatmap-preview`, `useDashboardSSE` |
| 장비 상세 `stateDetail` | `RealtimeDashboard/EquipmentDetail` | 장비별 탭(`stateDetail::PK`), 위젯 창(`DashboardWindow`, react-grid-layout), 히트맵/웨이퍼맵, 토글 탭(온도, 테스트시간·인덱스, 알람 이력, 활성 룰), 이벤트 로그 | `GET /manager/equipment/{pk}/...`, `parm-hist`, `widget-position`, `test-time-chart-setting`, `useEquipmentSSE`, `useEventLogSSE` |
| 레이아웃 편집 `editLayout` | `RealtimeDashboard/EditLayout` | 장비 배치 격자 편집(DnD, 프리셋, 자동 스크롤), 소유자/관리자만 편집·삭제 | `GET/POST/PUT/DELETE /manager/layout` |
| 테스트 이력 `testHistory`(구 `config`) | `RealtimeDashboard/FileHistory` | STDF 파일 이력 표 + 상세 패널 | `GET /manager/file-history` |
| 장비 관리 `managementList` | `EquipmentManagement/ManagementList` | 장비 CRUD(모달 폼), 메모, 모듈 상태 | `GET /manager/equipment/list(/page)`, `useManagementSSE` |
| 모듈 배포 `moduleDeploy` | `EquipmentManagement/ModuleDeploy` | 에이전트 모듈 버전 업로드/패치 실행, 단계 진행바 | `mngr-patch/*` (multipart, 300s 타임아웃) |
| 파일 목록 `fileList` / 파일 탐색기 `fileExplorer` | `EquipmentManagement/FileList`, `FileExplorer` | 장비 서버 파일 브라우징·다운로드(blob) | `mngr-file/equipment`, `file-explorer`, `useControllerSSE` |
| 규칙 `ruleList` | `EquipmentManagement/Rule` | 룰 목록/상세/폼/이력/타임라인, 모집단 편집, 편집 잠금(다른 사용자 수정 중 표시), 초안 | `/rules`, `/rules/catalog`, `useRuleEvalMutation`, `ruleLiveSync` |
| 알람 `alarm` | `EquipmentManagement/Alarm` | 알람 목록(쪽 단위 200건), 상세 카드, 확인/정지/응답자 | `/alarms`, `useAlarmSSE` |
| 애널리틱스 스페이스 `analyticsSpace`/`analyticsSpace2` | `AnalyticsSpace` | 스페이스(명명된 페이지) 단위 위젯 그리드, 데이터소스/필드 선택 마법사, BAR/LINE/SCATTER/PIE/TABLE/CONTROL/WAFERMAP 렌더러(Chart.js + ECharts v2), 드릴다운/롤업, xlsx 내보내기 | `/analytics/*` |
| 시스템 관리 16종 | `system/*` | 부서·사용자·권한·권한메뉴·사용자권한·사용자권한매핑·프로그램·메뉴·코드유형·코드·검색팝업·검색콤보·시스템메시지·로그인이력·시스템이력·오류이력 | 각 `/api/<자원>` GET/POST, 이력 3종은 무한 스크롤 |

---

## 14. 테스트/품질

- **단위**: `vitest` (jsdom, globals, `src/setupTests.ts` 가 jest-dom + `ResizeObserver` 폴리필). 테스트 111개 파일 `src/__tests__/{apis,components,features,hooks,models,stores,utils,fixtures}`. 실행 `npm test` / `npm test -- --run`.
- **MSW**: `src/mocks/{userHandlers,menuHandlers,systemHandlers}.ts` — `VITE_MODE=base` 로 브라우저 목 모드, Storybook 은 `msw-storybook-addon`.
- **Storybook 10**: `*.stories.tsx` 가 feature 폴더에 공존(장비 표, 히트맵, 차트 등). `npm run storybook` (6006).
- **e2e**: `e2e/` 폴더만 있고 비어 있음(Playwright 미도입).
- **CI**(`.gitlab-ci.yml`): MR(develop/master 대상)·push(develop/master) 시 `notify` 스테이지만 실행(사내 CI 템플릿 `sck-ci-templates` 의 MR/머지/스테일 알림). `build/deploy` 스테이지는 주석 처리. SonarQube 설정 파일 존재(`sonar-project.properties`, lcov).

---

## 15. 빌드/배포

| 스크립트 | 동작 |
|---|---|
| `npm run dev` / `npm start` | Vite 개발 서버 `https?://localhost:3000` (인증서 있으면 HTTP/2) |
| `npm run build` | `tsc && vite build` → `build/` (sourcemap 포함) |
| `npm run preview` | 빌드 미리보기 |
| `npm test` | vitest |
| `npm run storybook` / `build-storybook` | Storybook |

- **백엔드 통합(단일 jar)**: 부모 리포 `sck-server-spring` 의 CI 가 프론트를 함께 빌드한다.
  1. `build_frontend_job`(`node:20-alpine`, develop 브랜치): `cd src/main/frontend && npm ci && npm run build` → artifacts `src/main/frontend/build/` (`.gitlab/build/frontend.yml`).
  2. `build_backend_job`(JDK 17): `cp -r src/main/frontend/build/* src/main/resources/static/` 후 `./gradlew clean build -x test` → jar 의 `BOOT-INF/classes/static/` 에 포함 (`.gitlab/build/backend.yml`).
  3. 서브모듈은 `GIT_SUBMODULE_STRATEGY: recursive` 로 **부모가 박제한 커밋**을 그대로 쓴다(`--remote` 금지 — develop 배포에 다른 브랜치 프론트가 섞이는 것 방지). 프론트 변경을 배포하려면 부모 리포의 서브모듈 포인터를 갱신해 커밋해야 한다.
- 로컬에서 jar 에 넣어 보려면 `build.gradle` 하단의 주석 처리된 `installReact → buildReact → cleanStaticFolder → copyReactBuildFiles` 태스크를 잠시 풀어 쓴다(커밋 금지). `src/main/resources/static/` 은 gitignore.
- **운영 서빙**: Spring 기본 정적 리소스 핸들러가 `/index.html`, `/assets/*` 를 제공하고, 앞단 nginx 가 TLS 종료 + HTTP/2 를 맡는다(vite.config.ts 주석). 브라우저 입장에서 화면과 `/api` 가 **같은 출처**라 CORS 가 개입하지 않는다. HashRouter 라 딥링크 새로고침도 항상 `/` 를 받으므로 SPA fallback 이 필요 없다. 개발 환경은 Vite 프록시가 같은 역할을 하며 `Origin` 헤더를 제거해 Spring `cors.allowed-origins`(`http://localhost:3000`) 충돌을 피한다.
- `index.html` 이 `/assets/...` 절대 경로를 참조하므로 컨텍스트 패스는 `/` 여야 한다(Vite `base` 기본값).
- 배포 환경 구성 요건: HTTP/2(SSE 다중 연결), `/api/**` 프록시 시 **버퍼링 끄기·긴 read timeout**(SSE), 업로드 본문 크기 상향(패치 모듈 업로드, 서버 `multipart max 1000MB`).
- 프론트 리포 자체 CI(`.gitlab-ci.yml`)는 알림(notify) 스테이지만 돈다. 실제 빌드·배포는 부모 리포 파이프라인(develop push → 빌드 → Sonar → 수동 dev 배포)에서 일어난다. 자세한 단계는 [backend.md §9](./backend.md#9-빌드배포).
- 브랜치: `develop`(통합) → `release/<yyyymmdd>vN` → `master`, 릴리즈 태그 CalVer `YYYYMMDDvN`. Jira `ROSAT-###`.

---

## 16. 재구현 체크리스트

1. Vite 7 + React 18 + TS strict 프로젝트 생성, alias `@/`, Tailwind 3 + styled-components 6 설정, `theme.ts` 토큰 정의.
2. `apis/apiUrlConfig.ts`(URL 사전) / `apiKeyConfig.ts`(쿼리 키) 작성 → axios 인스턴스(5s 기본 타임아웃, `withCredentials`, `Accept-Language`·`menuId`·`programId` 헤더, 401/403 → 세션 정리, `x-user-id` 대조).
3. 인증: 쿠키 기반(`accessToken` HttpOnly + `isLoggedIn` 플래그). `authStore`(sessionStorage) + `useMe` + `AuthInitializer` 가드 + HashRouter. 로그인 성공 시 캐시/sessionStorage 초기화 후 `/#/` 해시 이동.
4. 세션: `useIdleSession`(활동 이벤트 → 240분 비활동 로그아웃, 10분 간격 `/refresh`), 탭 간 `BroadcastChannel` 동기화, `clearClientSession` 단일 진입점.
5. 메뉴: `GET /menu/main` 평면 목록 → `separateMenus`(level 1 헤더 / 하위 리프) → 사이드바(접힘·검색·레일) → `selectTargetMenu` 로 탭 삽입 → `Frame.tsx` `programId` 매핑. `MAX_TAB=20`, 장비 상세 합성 ID `stateDetail::PK`, 히스토리 미러링.
6. 시스템 관리 화면: 레거시 편집 그리드 프로토콜(`RowType` 0/1/2/3, 필수키 검사, 배열 POST). 사용자(`/user-info`)·메뉴(`/menu`, 7자리 ID)·권한(`/authority`)·권한메뉴(`/menu-authority`, 자손 전파)·사용자권한(`/user-authority`, assigned/possible).
7. 실시간: SSE 훅 3+5종, 백오프 재연결, `setQueryData` 캐시 패치, 알람 SSE 는 싱글턴 + 증분 재동기화.
8. i18n: i18next(ko/en, localStorage `i18nextLng`), `menu.<menuId>` 키, 서버에 `Accept-Language`.
9. 테마: `ThemeContext`(localStorage `sck-theme-mode`, `data-theme`), Chart.js 기본색 동기화.
10. 테스트/품질: vitest + RTL + MSW(목 모드 `VITE_MODE=base`), Storybook, ESLint/Prettier.
11. 배포: `npm run build` → `build/` 를 백엔드 `src/main/resources/static/` 에 복사해 단일 jar 로 묶는다(부모 CI). 앞단 nginx 는 TLS·HTTP/2, `/api` SSE 버퍼링 off.

**함정**
- `Frame.tsx` 매핑 누락 → 탭은 열리고 화면이 빈다. 메뉴 ID 는 7자리·i18n `menu.<id>` 필요.
- HashRouter 라 로그인/로그아웃 이동이 리로드가 아니다 → 이전 사용자 캐시/sessionStorage 를 명시적으로 비워야 한다.
- `accessToken` 쿠키는 HttpOnly 라 JS 로 못 지운다 → 로그아웃은 반드시 서버 `/logout` 호출.
- 기본 타임아웃 5초를 올리지 말 것(무응답 요청 방지 안전망). 긴 조회는 호출부에서 `LONG_READ_TIMEOUT_MS`.
- SSE 가 많아 HTTP/1.1 에서는 요청이 줄을 선다 → dev 도 TLS(HTTP/2) 권장, 프록시는 닫힌 SSE 의 상류 연결을 끊어야 한다.
- `VITE_MODE=base` 에서는 표 편집 버튼이 비활성화되고 쿠키 인증이 꺼진다.
- `Login.tsx` 의 입력 검증 메시지는 한글 하드코딩(i18n 미적용) — 재구현 시 i18n 키로.
