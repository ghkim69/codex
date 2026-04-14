---
title: Recovery Playbook — 소프트웨어 개발 교착 패턴 & Advisor 교정 사례
---

# Recovery Playbook

## 1. 백엔드 개발 교착 패턴

### 1-1. 환경변수/DB 연결 실패

**교착 신호:** [P]=0, [C]=낮음, [B]=있음 → 필수 트리거

**ACP 예시:**
```
[TRIGGER] C=낮음+D=아니오
[GOAL]    PostgreSQL 연결 후 users 테이블 초기 마이그레이션 실행. Node 18, pg@8.x.
[HARD_CONSTRAINTS] runtime: Node 18, db: PostgreSQL 15, host: localhost:5432
[ATTEMPTS]
  A1: [env] DATABASE_URL에 연결 문자열 설정 → "ECONNREFUSED 127.0.0.1:5432"
  ×3: [반복] 포트 변경(5433, 5434) → 동일 ECONNREFUSED
[BLOCKER]
  type: ConnectionError
  loc:  db/connect.ts:12
  msg:  connect ECONNREFUSED 127.0.0.1:5432
[HYPOTHESIS]
  H1 (60%): PostgreSQL 서비스 미실행
  H2 (40%): 소켓 경로 문제 (Docker 내부 네트워크)
[ASK] 로컬 PostgreSQL vs Docker 컨테이너 중 어떤 환경인지 확인 방법?
```

**Advisor 교정 방향:** `pg_isready -h localhost -p 5432` 실행 후 미응답이면 `brew services start postgresql` 또는 `docker-compose up -d db`. 응답하면 `.env` 파일 vs 실제 실행 환경의 DATABASE_URL 비교.

---

### 1-2. 외부 API 인증 실패 (401/403 반복)

**교착 신호:** [C]=낮음, [D]=아니오 → 필수 트리거

**ACP 예시:**
```
[TRIGGER] C=낮음+D=아니오
[GOAL]    Stripe API로 결제 인텐트 생성. Node 18, stripe@14.x.
[HARD_CONSTRAINTS] runtime: Node 18, ext_api: Stripe v1, libs: stripe@14
[ATTEMPTS]
  A1: [auth] Authorization: Bearer {sk_test_...} 헤더 설정 → 401
  A2: [auth] stripe.paymentIntents.create() SDK 사용 → 401
  ×2: [반복] API 키 재입력 → 동일 401
[BLOCKER]
  type: AuthError, loc: payments/stripe.ts:8
  msg:  No such payment_intent: 'pi_xxx'; a similar object exists in live mode
[HYPOTHESIS]
  H1 (70%): test/live 키 환경 불일치
  H2 (30%): Stripe 계정 활성화 미완료
[ASK] .env의 STRIPE_SECRET_KEY가 sk_test_ 로 시작하는지 확인 방법?
```

**Advisor 교정 방향:** `process.env.STRIPE_SECRET_KEY.startsWith('sk_test_')` 로그 출력. 라이브 키가 섞여 있으면 테스트 키로 교체. Dashboard > Developers > API keys에서 Restricted vs Secret 키 구분 확인.

---

### 1-3. TypeScript 타입 에러 반복

**교착 신호:** [D]=아니오 3회 연속 → 필수 트리거

**ACP 예시:**
```
[TRIGGER] D=아니오×3
[GOAL]    Express 라우터에서 req.user 타입 확장. TypeScript strict.
[HARD_CONSTRAINTS] runtime: Node 18, libs: express@4, @types/express@4
[ATTEMPTS]
  A1: [type] req.user에 as any 캐스팅 → 에러 숨김, 런타임 정상
  A2: [type] interface User 직접 정의 → TS2339: Property 'user' does not exist
  A3: [type] Request 재선언 → TS2300: Duplicate identifier
[BLOCKER]
  type: TypeError, loc: types/express.d.ts:3
  msg:  Duplicate identifier 'Request'
[HYPOTHESIS]
  H1 (75%): module augmentation 구문 누락
  H2 (25%): @types/express 버전 불일치
[ASK] express.d.ts에서 기존 Request를 확장하는 올바른 module augmentation 구문?
```

**Advisor 교정 방향:** `declare global { namespace Express { interface Request { user?: User } } }` 구문을 `express.d.ts`에 작성. `import` 없는 파일로 분리하거나 `/// <reference types="express" />` 추가.

---

## 2. 프론트엔드 개발 교착 패턴

### 2-1. API Shape 불일치

**교착 신호:** [B]=있음 (APIShapeMismatch), [C]=낮음 → 권장/필수 트리거

**ACP 예시:**
```
[TRIGGER] B=있음+C=낮음
[GOAL]    /api/users 응답을 UserList 컴포넌트에 렌더링. Next.js 14, TypeScript strict.
[HARD_CONSTRAINTS] framework: Next.js 14, api_spec: _workspace/backend/api_spec.md
[ATTEMPTS]
  A1: [shape] users.map(u => u.name) → TypeError: Cannot read 'map' of undefined
  A2: [shape] (users as any[]).map → 에러 숨김, 빈 화면
[BLOCKER]
  type: APIShapeMismatch, loc: hooks/useUsers.ts:18
  msg:  Cannot read properties of undefined (reading 'map')
[HYPOTHESIS]
  H1 (65%): API 응답이 배열이 아닌 {data: User[]} 래핑 형태
  H2 (35%): 빈 배열 vs undefined 처리 누락
[ASK] /api/users 실제 응답이 User[] 직접인지 {data: User[]} 래핑인지?
```

**Advisor 교정 방향:** `_workspace/backend/api_spec.md`의 `/api/users` 응답 스키마 재확인. `console.log(JSON.stringify(response))` 로 원형 확인. `response?.data ?? response ?? []` 패턴으로 방어 처리 후 타입 정의 수정.

---

### 2-2. 렌더링 무한 루프

**교착 신호:** [D]=아니오 3회 → 필수 트리거

**ACP 예시:**
```
[TRIGGER] D=아니오×3
[GOAL]    useEffect에서 API 호출 후 상태 업데이트. React 18.
[HARD_CONSTRAINTS] framework: React 18, state_mgr: useState
[ATTEMPTS]
  A1: [render] useEffect(() => fetchData(), [data]) → 무한 루프
  A2: [render] useEffect(() => fetchData(), [data.id]) → 무한 루프
  A3: [render] useEffect(() => fetchData(), [JSON.stringify(data)]) → 루프 감소, 여전히 반복
[BLOCKER]
  type: RenderLoop, loc: components/UserList.tsx:22
  msg:  Warning: Maximum update depth exceeded
[HYPOTHESIS]
  H1 (80%): fetchData 내부에서 data 상태 변경 → 의존성 재트리거
  H2 (20%): 객체 참조 동일성 문제
[ASK] fetchData 함수가 setState를 호출하는지, 의존성에서 제거 가능한지?
```

**Advisor 교정 방향:** 의존성 배열에서 `data` 제거하고 `[]`로 마운트 시 1회 실행. 재조회가 필요하면 `refetch` 함수를 명시적으로 호출하는 패턴으로 변경. `useCallback`으로 fetchData 참조 안정화.

---

## 3. QA 검증 교착 패턴

### 3-1. TypeScript 제네릭 캐스팅으로 추적 불가

**교착 신호:** [B]=있음 (GenericCasting), [C]=낮음 → 필수 트리거

**ACP 예시:**
```
[TRIGGER] B=있음+C=낮음
[GOAL]    useUserData<User>() 훅의 반환 타입이 API 실제 응답과 일치하는지 검증.
[HARD_CONSTRAINTS] ts_cfg: strict:true, spec: _workspace/backend/api_spec.md
[ATTEMPTS]
  A1: [type] 훅 타입 정의 읽기 → T extends Record<string,any> 제네릭만 확인
  A2: [type] 호출 측 as User 캐스팅 발견 → 런타임 불일치 여부 정적 확인 불가
  ×2: [반복] 관련 파일 재읽기 → 여전히 불확실
[BLOCKER]
  type: GenericCasting, loc: hooks/useUserData.ts:5
  msg:  return data as T — 런타임 실제 shape 정적 추론 불가
[HYPOTHESIS]
  H1 (55%): 런타임에 shape 불일치 존재 (id 대신 _id 반환 가능성)
  H2 (45%): 실제로 일치하지만 캐스팅으로 숨겨짐
[ASK] API 응답 필드가 id인지 _id인지 확인할 수 있는 로그/테스트 방법?
```

**Advisor 교정 방향:** `zod` 스키마로 런타임 검증 추가. 또는 API 응답에 `console.log` 임시 추가 후 실제 키 목록 확인. 정적 분석으로 불가하면 통합 테스트(Playwright/Cypress)로 실제 응답 캡처.

---

## 4. Advisor 교정 성공 사례

### 사례 1 — 환경변수 DB 연결 교착 (backend-dev)

**ACP 수신 (요약):** `ECONNREFUSED 127.0.0.1:5432`, 포트 변경 3회 동일 에러, Docker 환경 여부 불명

**Advisor 응답:**

```markdown
## Advisor 교정 진단

**교착 원인:** 포트를 변경해도 ECONNREFUSED가 유지되는 패턴은 PostgreSQL
서비스 자체 미실행을 가리킨다. Docker 내부라면 `localhost`가 컨테이너를
벗어나지 못하는 네트워크 격리 문제일 수 있다.

**기존 시도의 맹점:** 포트 번호만 변경하는 시도는 서비스 상태나
네트워크 경로를 확인하지 않은 동일 방향 반복이다.

**교정 접근법:**
1. `pg_isready -h localhost -p 5432` 실행 → "accepting connections" 이면
   연결 문자열 재확인, "no response" 이면 서비스 미실행
2. 서비스 미실행이면 `docker-compose ps` 또는 `brew services list` 로
   PostgreSQL 상태 확인 후 기동
3. Docker 환경이면 DATABASE_URL의 host를 `localhost` → `db` (서비스명)으로 변경

**성공 기준:** `pg_isready` 가 "accepting connections" 반환,
이후 `npm run migrate` 에러 없이 완료

**주의:** Docker Compose 내부에서 localhost는 컨테이너 자신을 가리킨다.
다른 컨테이너의 DB에 접근하려면 서비스명을 호스트로 사용해야 한다.
```

**결과:** backend-dev가 `pg_isready` 실행 → "no response" 확인 → `docker-compose up -d db` → 연결 성공. Gatekeeper 루프 미개입으로 자율 회복.

---

### 사례 2 — API Shape 불일치 교착 (frontend-dev)

**ACP 수신 (요약):** `users.map is not a function`, `{data: User[]}` 래핑 가설 65%, backend-dev 확인 불가 상태

**Advisor 응답:**

```markdown
## Advisor 교정 진단

**교착 원인:** ATTEMPTS에서 `as any[]`로 에러를 숨겼지만 빈 화면이 나오는
것은 응답이 배열이 아님을 확인한다. 백엔드 확인 전에 실제 응답 구조를
직접 로깅하면 추측을 제거할 수 있다.

**기존 시도의 맹점:** 타입 캐스팅으로 에러를 숨기는 시도는 원인 진단이 아닌
증상 억제다. 실제 응답 shape을 한 번도 출력하지 않았다.

**교정 접근법:**
1. `useUsers` 훅 내부에 `console.log('raw response:', JSON.stringify(res))`
   추가 → 브라우저 콘솔에서 실제 shape 확인
2. `{data: User[]}` 래핑이면 `return res.data` 로 훅 수정, 타입 정의도
   `ApiResponse<User[]>` 제네릭으로 교체
3. `_workspace/backend/api_spec.md` 의 응답 스키마 섹션을 재확인해서
   타입 정의 파일과 동기화

**성공 기준:** `users.map(u => u.name)` 가 에러 없이 실행되고
UserList 컴포넌트가 실제 데이터를 렌더링함

**주의:** `as any` 캐스팅을 제거하지 않으면 수정 후에도 잘못된 타입이
숨겨져 다른 컴포넌트에서 동일 문제가 재발할 수 있다.
```

**결과:** frontend-dev가 로그 추가 → `{data: User[], total: number}` 래핑 확인 → 훅 수정 → 렌더링 성공.

---

### 사례 3 — QA 제네릭 캐스팅 검증 교착 (qa-inspector)

**ACP 수신 (요약):** `return data as T` 패턴으로 런타임 shape 정적 확인 불가, `id` vs `_id` 필드 불확실

**Advisor 응답:**

```markdown
## Advisor 교정 진단

**교착 원인:** `as T` 캐스팅은 TypeScript에게 신뢰를 강제하는 구문으로,
정적 분석으로는 런타임 shape을 검증할 수 없다. 코드 읽기를 반복해도
결론이 나지 않는 이유는 증거가 코드 레이어에 없기 때문이다.

**기존 시도의 맹점:** 동일 파일을 반복 읽는 것은 새로운 정보를 생성하지 않는다.
런타임 레이어 또는 테스트 레이어에서 실제 응답을 캡처해야 한다.

**교정 접근법:**
1. `_workspace/backend/` 의 실제 응답 fixture 파일이나
   테스트 스냅샷에서 `id` vs `_id` 확인 (파일: tests/fixtures/)
2. 없으면 `_workspace/backend/api_spec.md` 에서 users 스키마 필드명 직접 확인
3. 불확실하면 "이 훅의 런타임 타입 안전성 확인 불가 — 통합 테스트 필요"로
   `_workspace/qa-report/findings.md` 에 Medium 심각도로 기록하고 다음 항목 진행

**성공 기준:** findings.md에 해당 항목 확정 상태(버그있음/없음/테스트필요) 기록

**주의:** 모든 항목을 100% 정적으로 확인하려다 검증 전체가 멈추는 것이
더 큰 위험이다. "확인 불가"도 유효한 QA 결과다.
```

**결과:** qa-inspector가 `tests/fixtures/` 확인 → `id` 필드 사용 확인 → 훅의 `_id` 참조 발견 → `BUG_FRONTEND` SendMessage. 교착 해소.

---

## 5. 교착 예방 패턴

| 패턴 | 예방 방법 |
|------|----------|
| DB 연결 교착 | Phase 1에서 연결 테스트를 첫 번째 태스크로 설정 |
| API shape 불일치 | backend-dev가 openapi.yaml 먼저 작성, frontend-dev가 코드 생성 |
| 타입 에러 누적 | `as any` 금지 규칙을 tsconfig `strict: true`와 eslint로 강제 |
| 렌더링 루프 | useEffect 의존성 배열 리뷰를 컴포넌트 완성 체크리스트에 포함 |
| QA 정적 분석 한계 | fixtures 및 통합 테스트를 QA 단계 전에 준비 |
