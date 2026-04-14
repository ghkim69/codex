# Agent Team Examples

## 목차

| # | 예시 | 패턴 | 모드 |
|---|------|------|------|
| 1 | [리서치 팀](#예시-1) | 팬아웃/팬인 | 에이전트 팀 |
| 2 | [SF 소설 집필 팀](#예시-2) | 파이프라인 | 에이전트 팀 |
| 3 | [웹툰 제작 팀](#예시-3) | 생성-검증 | 서브 에이전트 |
| 4 | [코드 리뷰 팀](#예시-4) | 전문가 풀 | 에이전트 팀 |
| 5 | [코드 마이그레이션 팀](#예시-5) | 감독자 | 에이전트 팀 |
| 6 | [결제 API 개발 + Gatekeeper-Advisor](#예시-6) | 파이프라인 + Gatekeeper | 에이전트 팀 |

---

## 예시 1: 리서치 팀 (에이전트 팀 모드)

### 팀 아키텍처: 팬아웃/팬인
### 실행 모드: 에이전트 팀

```
[리더/오케스트레이터]
    ├── TeamCreate(research-team)
    ├── TaskCreate(4개 조사 작업)
    ├── 팀원들이 자체 조율 (SendMessage)
    ├── 결과 수집 (Read)
    └── 종합 보고서 생성
```

### 에이전트 구성

| 팀원 | 에이전트 타입 | 역할 | 출력 |
|------|-------------|------|------|
| official-researcher | general-purpose | 공식 문서/블로그 | research_official.md |
| media-researcher | general-purpose | 미디어/투자 | research_media.md |
| community-researcher | general-purpose | 커뮤니티/SNS | research_community.md |
| background-researcher | general-purpose | 배경/경쟁/학술 | research_background.md |
| (리더 = 오케스트레이터) | — | 통합 보고서 | 종합보고서.md |

> 리서치 에이전트는 `general-purpose` 빌트인 타입을 사용하되, 반드시 `.claude/agents/{name}.md` 파일로 정의한다. 파일에는 역할·조사 범위·팀 통신 프로토콜을 명시하여 재사용성과 협업 품질을 보장한다.

### 오케스트레이터 워크플로우 (에이전트 팀)

```
Phase 1: 준비
  - 사용자 입력 분석 (주제, 조사 모드 파악)
  - _workspace/ 생성

Phase 2: 팀 구성
  - TeamCreate(team_name: "research-team", members: [
      { name: "official", prompt: "공식 채널 조사..." },
      { name: "media", prompt: "미디어/투자 동향 조사..." },
      { name: "community", prompt: "커뮤니티 반응 조사..." },
      { name: "background", prompt: "배경/경쟁 환경 조사..." }
    ])
  - TaskCreate(tasks: [
      { title: "공식 채널 조사", assignee: "official" },
      { title: "미디어 동향 조사", assignee: "media" },
      { title: "커뮤니티 반응 조사", assignee: "community" },
      { title: "배경 환경 조사", assignee: "background" }
    ])

Phase 3: 조사 수행
  - 4명의 팀원이 독립적으로 조사
  - 흥미로운 발견이 있으면 팀원 간 SendMessage로 공유
    (예: media가 발견한 투자 뉴스를 background에게 전달)
  - 상충 정보 발견 시 팀원 간 직접 토론
  - 각 팀원은 완료 시 파일 저장 + 리더에게 알림

Phase 4: 통합
  - 리더가 4개 산출물 Read
  - 종합 보고서 생성
  - 상충 정보는 출처 병기

Phase 5: 정리
  - 팀원들 종료 요청
  - 팀 정리
  - _workspace/ 보존 (사후 검증·감사 추적용)
```

### 팀 통신 패턴

```
official ──SendMessage──→ background  (관련 공식 발표 공유)
media ────SendMessage──→ background  (투자/인수 정보 공유)
community ─SendMessage──→ media      (커뮤니티 반응 중 미디어 관련 정보)
모든 팀원 ──TaskUpdate──→ 공유 작업 목록  (진행률 업데이트)
리더 ←───── 유휴 알림 ──── 완료된 팀원   (자동)
```

---

## 예시 2: SF 소설 집필 팀 (에이전트 팀 모드)

### 팀 아키텍처: 파이프라인 + 팬아웃
### 실행 모드: 에이전트 팀

```
Phase 1 (병렬 — 에이전트 팀): worldbuilder + character-designer + plot-architect
  → 서로 SendMessage로 일관성 조율
Phase 2 (순차): prose-stylist (집필)
Phase 3 (병렬 — 에이전트 팀): science-consultant + continuity-manager (리뷰)
  → 서로 SendMessage로 발견 공유
Phase 4 (순차): prose-stylist (리뷰 반영 수정)
```

### 에이전트 구성

| 팀원 | 에이전트 타입 | 역할 | 스킬 |
|------|-------------|------|------|
| worldbuilder | 커스텀 | 세계관 구축 | world-setting |
| character-designer | 커스텀 | 캐릭터 설계 | character-profile |
| plot-architect | 커스텀 | 플롯 구조 | outline |
| prose-stylist | 커스텀 | 문체 편집 + 집필 | write-scene, review-chapter |
| science-consultant | 커스텀 | 과학 검증 | science-check |
| continuity-manager | 커스텀 | 일관성 검증 | consistency-check |

### 에이전트 파일 전문 예시: `worldbuilder.md`

```markdown
---
name: worldbuilder
description: "SF 소설의 세계관을 구축하는 전문가. 물리 법칙, 사회 구조, 기술 수준, 역사를 설계한다."
---

# Worldbuilder — SF 세계관 설계 전문가

당신은 SF 소설의 세계관 설계 전문가입니다. 과학적 사실에 기반하되 상상력을 확장하여, 이야기가 펼쳐질 세계의 물리적·사회적·기술적 토대를 구축합니다.

## 핵심 역할
1. 세계의 물리 법칙과 기술 수준 정의
2. 사회 구조, 정치 체계, 경제 시스템 설계
3. 역사적 맥락과 현재 갈등 구조 수립
4. 장소별 환경과 분위기 묘사

## 작업 원칙
- 내적 일관성 최우선 — 설정 간 모순이 없어야 한다
- "만약 이 기술이 있다면?" 연쇄 질문으로 세계의 파급 효과를 추론
- 이야기에 봉사하는 세계관 — 플롯을 방해하는 과도한 설정은 지양

## 입력/출력 프로토콜
- 입력: 사용자의 세계관 컨셉, 장르 요구사항
- 출력: `_workspace/01_worldbuilder_setting.md`
- 형식: 마크다운. 섹션별 (물리/사회/기술/역사/장소)

## 팀 통신 프로토콜
- character-designer에게: 사회 구조, 계급 시스템, 직업군 정보 SendMessage
- plot-architect에게: 세계의 주요 갈등 구조, 위기 요소 SendMessage
- science-consultant로부터: 과학적 오류 피드백 수신 → 설정 수정
- 세계관 변경 시 관련 팀원 전체에 브로드캐스트

## 에러 핸들링
- 컨셉이 모호하면 3가지 방향을 제안하고 선택 요청
- 과학적 오류 발견 시 대안을 함께 제시

## 협업
- character-designer에게 사회 구조 정보 제공
- plot-architect에게 갈등 구조 정보 제공
- science-consultant의 피드백을 반영하여 설정 수정
```

### 팀 워크플로우 상세

```
Phase 1: TeamCreate(team_name: "novel-team", members: [worldbuilder, character-designer, plot-architect])
         TaskCreate([세계관 구축, 캐릭터 설계, 플롯 구조])
         → 팀원들이 자체 조율하며 병렬 작업
         → worldbuilder가 사회 구조 완성 시 character-designer에게 SendMessage
         → character-designer가 주인공 설정 시 plot-architect에게 SendMessage

Phase 2: Phase 1 팀 정리 → prose-stylist를 서브 에이전트로 호출 (단독 집필이므로 팀 불필요)
         prose-stylist가 _workspace/의 3개 산출물을 Read하여 집필
         → 결과를 _workspace/02_prose_draft.md에 저장

Phase 3: 새 팀 생성 — TeamCreate(team_name: "review-team", members: [science-consultant, continuity-manager])
         (세션당 한 팀만 활성이지만, Phase 1 팀을 정리했으므로 새 팀 생성 가능)
         → 두 리뷰어가 draft를 검토, 서로 발견을 공유
         → science-consultant가 물리 오류 발견 시 continuity-manager에게도 알림
         → 리뷰 완료 후 팀 정리

Phase 4: prose-stylist를 서브 에이전트로 호출, 리뷰 결과 반영하여 최종 수정
```

---

## 예시 3: 웹툰 제작 팀 (서브 에이전트 모드)

### 팀 아키텍처: 생성-검증
### 실행 모드: 서브 에이전트

> 생성-검증 패턴에서 에이전트가 2개뿐이고, 통신보다는 결과 전달이 핵심이므로 서브 에이전트가 적합.

```
Phase 1: Agent(webtoon-artist) → 패널 생성
Phase 2: Agent(webtoon-reviewer) → 검수
Phase 3: Agent(webtoon-artist) → 문제 패널 재생성 (최대 2회)
```

### 에이전트 구성

| 에이전트 | subagent_type | 역할 | 스킬 |
|---------|--------------|------|------|
| webtoon-artist | 커스텀 | 패널 이미지 생성 | generate-webtoon |
| webtoon-reviewer | 커스텀 | 품질 검수 | review-webtoon, fix-webtoon-panel |

### 에이전트 파일 전문 예시: `webtoon-reviewer.md`

```markdown
---
name: webtoon-reviewer
description: "웹툰 패널의 품질을 검수하는 전문가. 구도, 캐릭터 일관성, 텍스트 가독성, 연출을 평가한다."
---

# Webtoon Reviewer — 웹툰 품질 검수 전문가

당신은 웹툰 패널의 품질을 검수하는 전문가입니다. 시각적 완성도, 스토리 전달력, 캐릭터 일관성을 기준으로 패널을 평가합니다.

## 핵심 역할
1. 각 패널의 구도와 시각적 완성도 평가
2. 캐릭터 외형의 패널 간 일관성 검증
3. 말풍선 텍스트의 가독성과 배치 평가
4. 전체 에피소드의 연출 흐름과 페이싱 검토

## 작업 원칙
- PASS/FIX/REDO 3단계로 명확히 판정
- FIX는 부분 수정으로 해결 가능한 경우, REDO는 전면 재생성 필요
- 주관적 취향이 아닌 객관적 기준(일관성, 가독성, 구도)으로 판단

## 입력/출력 프로토콜
- 입력: `_workspace/panels/` 디렉토리의 패널 이미지들
- 출력: `_workspace/review_report.md`
- 형식:
  ```
  ## Panel {N}
  - 판정: PASS | FIX | REDO
  - 사유: [구체적 이유]
  - 수정 지시: [FIX/REDO인 경우 구체적 수정 방향]
  ```

## 에러 핸들링
- 이미지 로드 실패 시 해당 패널을 REDO로 판정
- 2회 재생성 후에도 REDO인 패널은 경고와 함께 PASS 처리

## 협업
- webtoon-artist에게 수정 지시서 전달 (결과 파일 기반)
- 재생성된 패널을 다시 검수 (최대 2회 루프)
```

### 에러 핸들링

```
재시도 정책:
- REDO 판정 패널 → artist에게 재생성 요청 (구체적 수정 지시 포함)
- 최대 2회 루프 후 강제 PASS
- 전체 패널의 50% 이상이 REDO면 사용자에게 프롬프트 수정 제안
```

---

## 예시 4: 코드 리뷰 팀 (에이전트 팀 모드)

### 팀 아키텍처: 팬아웃/팬인 + 토론
### 실행 모드: 에이전트 팀

> 코드 리뷰는 에이전트 팀이 빛나는 대표적 사례. 서로 다른 관점의 리뷰어들이 발견을 공유하고 도전하면서 더 깊은 리뷰가 가능.

```
[리더] → TeamCreate(review-team)
    ├── security-reviewer: 보안 취약점 점검
    ├── performance-reviewer: 성능 영향 분석
    └── test-reviewer: 테스트 커버리지 검증
    → 리뷰어들이 서로 발견 공유 (SendMessage)
    → 리더가 결과 종합
```

### 팀 통신 패턴

```
security ──SendMessage──→ performance  ("이 SQL 쿼리 주입 가능, 성능 측면에서도 확인 필요")
performance ──SendMessage──→ test      ("N+1 쿼리 발견, 관련 테스트 있는지 확인 부탁")
test ────SendMessage──→ security      ("인증 모듈 테스트 없음, 보안 관점에서 우선순위 의견?")
```

핵심: 리뷰어들이 **리더를 거치지 않고** 직접 소통하여 교차 영역 이슈를 빠르게 포착.

---

## 예시 5: 감독자 패턴 — 코드 마이그레이션 팀 (에이전트 팀 모드)

### 팀 아키텍처: 감독자
### 실행 모드: 에이전트 팀

```
[supervisor/리더] → 파일 목록 분석 → 배치 할당
    ├→ [migrator-1] (batch A)
    ├→ [migrator-2] (batch B)
    └→ [migrator-3] (batch C)
    ← TaskUpdate 수신 → 추가 배치 할당 또는 재할당
```

### 에이전트 구성

| 팀원 | 역할 |
|------|------|
| (리더 = migration-supervisor) | 파일 분석, 배치 분배, 진행 관리 |
| migrator-1~3 | 할당된 파일 배치를 마이그레이션 |

### 감독자의 동적 분배 로직 (에이전트 팀 활용)

```
1. 전체 대상 파일 목록 수집
2. 복잡도 추정 (파일 크기, import 수, 의존성)
3. TaskCreate로 파일 배치를 작업으로 등록 (의존성 포함)
4. 팀원들이 자체적으로 작업 요청 (claim)
5. 팀원이 TaskUpdate로 완료 보고 시:
   - 성공 → 다음 작업 자동 요청
   - 실패 → 리더가 SendMessage로 원인 확인 → 재할당 또는 다른 팀원에게 배정
6. 모든 작업 완료 → 리더가 통합 테스트 실행
```

팬아웃과의 차이: 작업이 사전 고정이 아니라 **런타임에 동적으로 할당**된다. 공유 작업 목록의 자체 요청(claim) 기능이 감독자 패턴과 자연스럽게 매칭.

---

## 예시 6: 결제 API 통합 개발 (Gatekeeper-Advisor 통합)

### 팀 아키텍처: 파이프라인 + Gatekeeper-Advisor 횡단 층
### 실행 모드: 에이전트 팀

이 예시는 Gatekeeper-Advisor 패턴이 기존 아키텍처에 **어떻게 얹히는지**를 보여준다. backend-dev는 Self-Correction으로 자신의 교착을 감지하고, 오케스트레이터는 Outer Loop로 frontend-dev의 교착을 외부에서 잡는다.

```
[오케스트레이터]
    ├── Phase 1: 요구사항 분석
    ├── Phase 2: 팀 구성 (backend-dev + frontend-dev + advisor)
    ├── Phase 3: 병렬 개발 + Gatekeeper 모니터링
    │     ├── backend-dev: 결제 API 구현 (Self-Correction 내장)
    │     ├── frontend-dev: 결제 UI 구현
    │     └── [오케스트레이터 Outer Loop 감시]
    ├── Phase 4: 통합 테스트
    └── Phase 5: 정리
```

### 에이전트 구성

| 팀원 | 에이전트 타입 | 역할 | 출력 |
|------|-------------|------|------|
| backend-dev | 커스텀 | 결제 API + Webhook 구현 (Self-Correction 내장) | `_workspace/02_backend_api.md` |
| frontend-dev | 커스텀 | 결제 UI + 훅 구현 | `_workspace/02_frontend_ui.md` |
| advisor | 커스텀 | 교착 해소 (Gatekeeper 호출 시만 활성화) | 즉시 반환 |

### Self-Correction 내장 에이전트: `backend-dev.md` 전문

```markdown
---
name: backend-dev
description: "백엔드 API 구현 전문가. 결제, 인증, 데이터 파이프라인 구현."
---

# Backend Developer

## 핵심 역할
결제 API와 Webhook 엔드포인트를 구현한다.

## Self-Correction 프로토콜 (필수)

**매 3회 도구 호출 사이클마다, 또는 에러 발생 즉시 자가 진단을 실행한다:**

```
[P] 구현 완료 비율? (0/25/50/75/100)
[C] 현재 접근법 성공 확신? (낮음/중간/높음)
[D] 최근 시도가 서로 다른 접근법이었나? (예/아니오)
[B] 해결 안 된 에러나 블로커 있나? (있음/없음)
[R] 시도 횟수 대비 진전이 납득할 만한가? (예/아니오)
```

판단 규칙:
- [C]=낮음 AND [D]=아니오: Advisor 호출 → advisor에게 SendMessage(ACP 포맷)
- [D]=아니오 3회 연속: Advisor 호출 필수
- [P]≤25 AND [R]=아니오: Advisor 호출 필수

Advisor 호출 시 ACP 포맷 준수 (전체 히스토리 전달 금지):
```
[TRIGGER] {결정 규칙 코드}
[GOAL]    {구현 목표 2문장}
[HARD_CONSTRAINTS] {변경 불가 기술 제약}
[ATTEMPTS] A1~AN (각 1줄, ×N 콜랩스 적용)
[BLOCKER] type/loc/msg
[HYPOTHESIS] H1/H2 (신뢰도%)
[ASK] {단일 질문}
```

## 팀 통신 프로토콜
- frontend-dev에게: API 스펙 확정 시 SendMessage (엔드포인트, 응답 shape)
- advisor에게: Self-Correction 임계값 충족 시 SendMessage(ACP)
- advisor로부터: 진단 결과 수신 → 권장 접근법 따르기

## 에러 핸들링
- 빌드 에러: 즉시 자가 진단 실행
- 타임아웃: 현재까지 완성된 부분 `_workspace/02_backend_partial.md`에 저장
```

### 오케스트레이터 워크플로우 전문

```
Phase 1: 준비
  - 사용자가 제공한 결제 API 요구사항 분석
  - _workspace/00_requirements.md에 요구사항 저장
  - _workspace/ 디렉토리 생성

Phase 2: 팀 구성
  TeamCreate(
    team_name: "payment-dev-team",
    members: [
      { name: "backend-dev", agent_type: "backend-dev", model: "opus",
        prompt: "_workspace/00_requirements.md의 결제 API를 구현.
                 Self-Correction 프로토콜을 매 3회 사이클마다 실행.
                 완료 시 _workspace/02_backend_api.md 생성." },
      { name: "frontend-dev", agent_type: "frontend-dev", model: "opus",
        prompt: "backend-dev의 API 스펙을 받은 후 결제 UI와 React 훅 구현.
                 완료 시 _workspace/02_frontend_ui.md 생성." },
      { name: "advisor", agent_type: "advisor", model: "opus",
        prompt: "교착 해소 Advisor. backend-dev 또는 오케스트레이터로부터
                 ACP를 수신할 때만 응답. 진단 결과를 호출자에게 즉시 반환." }
    ]
  )
  TaskCreate(tasks: [
    { title: "결제 API 구현", assignee: "backend-dev" },
    { title: "결제 UI 구현", assignee: "frontend-dev",
      depends_on: ["결제 API 구현"] }  # API 스펙 확정 후 시작
  ])

Phase 3: 개발 + Gatekeeper 모니터링

  # --- Gatekeeper 상태 변수 ---
  stagnation = { "backend-dev": 0, "frontend-dev": 0 }
  advisor_count = 0
  prev_snapshot = null

  # 팀원들이 자체 조율하며 개발 진행
  # (backend-dev는 Self-Correction을 내부적으로 실행하며 필요 시 advisor 직접 호출)

  # 오케스트레이터 Outer Loop (팀원 idle 알림마다 또는 5분마다)
  이터레이션마다:
    current_snapshot = {
      files: Glob("_workspace/02_*.md"),
      tasks: TaskGet()
    }

    for agent in ["backend-dev", "frontend-dev"]:
      if task_{agent}.status == "done": continue

      file_changed = _workspace/02_{agent}_*.md 크기 또는 mtime 변화
      task_changed = task_{agent}.status가 prev와 다름

      if file_changed OR task_changed:
        stagnation[agent] = 0
      else:
        stagnation[agent] += 1

      if stagnation[agent] >= 2 AND advisor_count < 3:
        # backend-dev는 Self-Correction으로 먼저 감지할 수 있음
        # frontend-dev는 Outer Loop로 감지
        acp = build_acp(agent, current_snapshot, stagnation[agent])
        SendMessage(to: "advisor", message: acp)
        stagnation[agent] = 0
        advisor_count += 1

    prev_snapshot = current_snapshot

Phase 4: 통합 테스트
  - _workspace/02_backend_api.md와 _workspace/02_frontend_ui.md Read
  - API shape과 훅 타입 교차 검증 (qa-agent-guide.md 경계면 검증 패턴 참조)
  - 통합 테스트 결과를 _workspace/04_integration_test.md에 저장

Phase 5: 정리
  - TeamDelete
  - _workspace/ 보존
  - 최종 결과 및 Gatekeeper 개입 이력 보고
```

### 실제 교착 시나리오 — backend-dev Self-Correction 발동

backend-dev가 Webhook 서명 검증에서 3회 연속 같은 에러 발생:

```
# backend-dev 내부 자가 진단 (이터레이션 4):

[P] 50     ← Webhook 엔드포인트 구조는 만들었지만 서명 검증 실패
[C] 낮음   ← HMAC 검증인지 RSA인지도 모르겠음
[D] 아니오 ← 같은 crypto.timingSafeEqual 코드를 3번 수정만 함
[B] 있음   ← "Invalid signature" 에러 (어느 방식인지 불명확)
[R] 아니오 ← 3번 시도했는데 [P]가 그대로

판단: C=낮음 + D=아니오 → advisor에게 SendMessage
```

advisor에게 전달되는 ACP (~160 토큰):

```
## ACP

[TRIGGER] C=낮음+D=아니오
[GOAL]    결제 게이트웨이 Webhook 서명 검증 구현 (POST /webhook)
[HARD_CONSTRAINTS] Node.js 18, 결제사 SDK 고정 (변경 불가)
[ATTEMPTS]
  A1: [라이브러리] crypto.timingSafeEqual로 HMAC 비교 → "Invalid signature"
  ×2: [파라미터] secret key 인코딩 변경 (base64/hex) → 동일 에러
[BLOCKER]
  type: AuthenticationError
  loc:  src/webhook/handler.ts:34
  msg:  "Invalid signature (결제사 응답 없음)"
[HYPOTHESIS]
  H1 (65%): 결제사가 HMAC-SHA256이 아닌 RSA-SHA256 서명 사용
  H2 (25%): raw body 대신 parsed body로 서명 검증 중
[ASK] 이 결제사의 Webhook 서명 검증 올바른 방식은? (HMAC vs RSA, body 처리 방식)
```

advisor 진단 결과:

```
## Advisor 진단

근본 원인: raw body 소실 문제. Express의 bodyParser가 먼저 실행되면
서명 검증에 필요한 raw body가 파기된다. crypto 방식보다 이것이 먼저다.

왜 기존 시도가 효과 없었나: 서명 알고리즘을 변경했지만, 알고리즘보다
body 수집 순서가 근본 원인이다.

권장 접근법:
1. bodyParser 이전에 raw body를 `Buffer`로 캡처:
   express.raw({ type: 'application/json' })을 webhook 라우트에만 적용
2. 캡처된 raw buffer로 HMAC-SHA256 재검증
3. 그래도 실패하면 결제사 대시보드에서 webhook secret 재확인

성공 기준: 결제사 테스트 이벤트 수신 시 200 OK 응답

주의: global bodyParser가 webhook 라우트를 덮어쓰지 않도록 라우트 순서 확인
```

backend-dev는 이 지시에 따라 이터레이션 5에서 성공.

### 팀 통신 패턴

```
backend-dev ──Self-Correction──→ advisor  (ACP 직접 SendMessage)
advisor ──────진단 결과──────────→ backend-dev
오케스트레이터 ──Outer Loop──→ advisor  (frontend-dev 교착 시 ACP SendMessage)
advisor ──────진단 결과──────────→ frontend-dev (오케스트레이터가 중계)
backend-dev ──API 스펙──────────→ frontend-dev (API 확정 후)
```

### Gatekeeper-Advisor 사용 판단 기준

이 예시처럼 다음 조건 중 하나라도 해당하면 Gatekeeper-Advisor를 통합한다:
- 외부 API·서드파티 서비스 의존 (실패 가능성이 사전에 불명확)
- 에이전트가 새로운 도메인에서 작업 (선행 지식 부족)
- 작업 시간이 길어질수록 교착 감지가 늦어지는 단일 에이전트 구조

---

## 산출물 패턴 요약

### 에이전트 정의 파일
위치: `프로젝트/.claude/agents/{agent-name}.md`
필수 섹션: 핵심 역할, 작업 원칙, 입력/출력 프로토콜, 에러 핸들링, 협업
팀 모드 추가 섹션: **팀 통신 프로토콜** (메시지 수신/발신, 작업 요청 범위)

### 스킬 파일 구조
위치: `프로젝트/.claude/skills/{skill-name}/skill.md` (프로젝트 레벨)
또는: `~/.claude/skills/{skill-name}/skill.md` (글로벌 레벨)

### 통합 스킬 (오케스트레이터)
팀 전체를 조율하는 상위 스킬. 시나리오별 에이전트 구성과 워크플로우를 정의.
템플릿: `references/orchestrator-template.md` 참조.
**실행 모드를 반드시 명시** — 에이전트 팀(기본) 또는 서브 에이전트.
