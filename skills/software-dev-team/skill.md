---
name: software-dev-team
description: "소프트웨어 개발 팀 오케스트레이터. Gatekeeper-Advisor 패턴 내장 — 개별 에이전트의 교착·오작동을 Advisor(Opus)가 자동으로 교정한다. '개발 팀 구성해줘', '백엔드/프론트엔드 병렬 개발', '풀스택 구현', 'API + UI 동시 개발' 요청 시 반드시 이 스킬을 사용할 것."
---

# Software Dev Team — Gatekeeper-Advisor 오케스트레이터

## 실행 모드

에이전트 팀. 오케스트레이터(이 파일)가 태스크 분배·모니터링·교착 탐지를 담당하고,
각 에이전트는 독립 서브에이전트로 병렬 실행된다.

---

## 에이전트 구성

| 에이전트        | 파일                          | 역할                                    | 모델 권장  |
|-------------|-----------------------------|-----------------------------------------|--------|
| backend-dev  | agents/backend-dev.md       | REST API, DB, 인증, Webhook               | Sonnet |
| frontend-dev | agents/frontend-dev.md      | UI 컴포넌트, 상태 관리, API 훅                  | Sonnet |
| qa-inspector | agents/qa-inspector.md      | 통합 정합성 검증 (경계면·상태전이·라우팅)               | Sonnet |
| advisor      | agents/advisor.md           | 교착 탐지 → ACP 수신 → 교정 지시 (Gatekeeper 역할) | Opus   |

---

## 워크플로우

### Phase 1 — 요구사항 분석 (오케스트레이터 단독)

1. 사용자 요청에서 다음을 추출한다:
   - 구현 목표 (기능 목록)
   - 기술 제약 (런타임, DB, 프레임워크, 외부 API)
   - 완료 기준 (동작 확인 방법)
2. `_workspace/` 디렉토리를 초기화한다.
3. 태스크 목록을 작성한다 (`TaskCreate` 또는 `_workspace/tasks.json`).

### Phase 2 — 태스크 분배 (오케스트레이터 → 에이전트)

1. backend-dev에게 API 엔드포인트 목록과 스키마를 SendMessage로 전달한다.
2. frontend-dev에게 UI 구조와 목표 API 계약을 SendMessage로 전달한다.
3. qa-inspector에게 검증 기준과 워크스페이스 경로를 SendMessage로 전달한다.
4. advisor에게 팀 구성·제약·워크스페이스 경로를 SendMessage로 알린다 (대기 상태).
5. 세 에이전트를 동시에 시작한다.

### Phase 3 — Gatekeeper 모니터링 루프 (핵심)

#### 초기화

```
stagnation      = {"backend-dev": 0, "frontend-dev": 0, "qa-inspector": 0}
advisor_calls   = {"backend-dev": 0, "frontend-dev": 0, "qa-inspector": 0}
last_snapshot   = {}   # {agent: {files: [...mtimes], tasks: [...statuses]}}
loop_count      = 0
```

#### 루프 진입 조건

- 에이전트 idle 알림 수신 시 즉시
- 또는 5분(300초)마다 주기적으로

#### 루프 본체 (매 반복)

```
loop_count += 1

# Step 1: 현재 상태 스냅샷 수집
current_files = Glob("_workspace/**")          # 모든 출력 파일 목록 + mtime
current_tasks = TaskGet()                       # 에이전트별 태스크 상태

# Step 2: 에이전트별 진전 비교
for agent in ["backend-dev", "frontend-dev", "qa-inspector"]:

  # 파일 변화: 해당 에이전트 소유 파일(_workspace/{agent}/ 하위)의 mtime 중 하나라도 변했는가
  agent_files_now  = {f.path: f.mtime for f in current_files if agent in f.path}
  agent_files_prev = last_snapshot.get(agent, {}).get("files", {})
  file_changed = (agent_files_now != agent_files_prev)

  # 태스크 변화: 해당 에이전트 담당 태스크의 status 중 하나라도 변했는가
  tasks_now  = {t.id: t.status for t in current_tasks if t.assignee == agent}
  tasks_prev = last_snapshot.get(agent, {}).get("tasks", {})
  task_advanced = (tasks_now != tasks_prev)

  if file_changed OR task_advanced:
    stagnation[agent] = 0
  else:
    stagnation[agent] += 1

  # 스냅샷 갱신
  last_snapshot[agent] = {"files": agent_files_now, "tasks": tasks_now}

# Step 3: 교착 판정 → Advisor 호출
for agent in ["backend-dev", "frontend-dev", "qa-inspector"]:

  if stagnation[agent] >= 2 AND advisor_calls[agent] < 3:

    # ACP 구성
    acp = f"""
[TRIGGER] stagnation={stagnation[agent]}, loop={loop_count}
[GOAL]    {agent}의 현재 담당 태스크를 완료한다. 구체적 목표: {TaskGet(assignee=agent, status="in_progress")[0].description}
[HARD_CONSTRAINTS] {프로젝트_제약}   # Phase 1에서 추출한 기술 스택·버전
[ATTEMPTS]
  (에이전트 로그 또는 마지막 메시지에서 추출)
[BLOCKER]
  type: (에이전트 마지막 에러 메시지 유형)
  loc:  (파일:라인, 알 수 있는 경우)
  msg:  (에러 첫 줄 100자 이내)
[HYPOTHESIS]
  H1 (??%): (오케스트레이터가 추정하는 원인)
[ASK] 이 교착을 해소하고 {agent}가 다음 단계로 진행할 수 있는 구체적 접근법은?
"""

    SendMessage(to="advisor", message=acp)
    # advisor는 진단 후 해당 에이전트에게 직접 SendMessage로 교정 지시를 보낸다
    # advisor의 교정 지시 발송은 advisor.md 프로토콜에 따른다

    stagnation[agent]    = 0
    advisor_calls[agent] += 1

# Step 4: Advisor 소진 시 사용자 에스컬레이션
for agent in ["backend-dev", "frontend-dev", "qa-inspector"]:

  if advisor_calls[agent] >= 3 AND stagnation[agent] >= 2:

    EscalateToUser(level=3, message=f"""
**[Level 3 에스컬레이션]** {agent} — Advisor 3회 교정 후에도 교착 지속

- 담당 태스크: {TaskGet(assignee=agent, status="in_progress")[0].description}
- 마지막 Advisor 교정: {advisor_last_response[agent]}
- 현재 블로커: {agent_last_error[agent]}

**필요한 결정:**
A. 해당 에이전트 태스크를 스킵하고 다음 Phase로 진행
B. 기술 제약 변경 (예: 다른 라이브러리 허용)
C. 수동 개입 후 재시도

어떻게 진행할까요?
""")
```

### Phase 4 — 통합 및 QA

1. backend-dev와 frontend-dev가 모두 완료 신호를 보내면 qa-inspector를 활성화한다.
2. qa-inspector가 `_workspace/` 전체를 검증한다.
3. qa-inspector의 발견 사항을 backend-dev, frontend-dev에게 각각 라우팅한다.
4. 수정 완료 후 qa-inspector 재검증을 실행한다.

### Phase 5 — 완료 보고

1. 모든 에이전트의 태스크 상태가 `done`으로 전환되면 루프를 종료한다.
2. 최종 결과물 요약을 사용자에게 전달한다.

---

## 에러 핸들링

| 상황                        | 조치                                               |
|---------------------------|--------------------------------------------------|
| 에이전트 응답 없음 (10분)          | idle 처리 → stagnation 카운터 즉시 2로 설정 → Advisor 호출 |
| Advisor 응답 없음 (5분)         | 직접 해당 에이전트에게 "현재 상태 보고" SendMessage            |
| advisor_calls >= 3 교착 지속   | Level 3 에스컬레이션 (Phase 3 Step 4)                 |
| qa-inspector 검증 실패 3회 반복   | 해당 버그를 직접 오케스트레이터가 backend/frontend에 분배       |
| _workspace 파일 접근 오류        | 경로 재확인 후 에이전트에게 올바른 경로 재전달                    |

---

## 테스트 시나리오

### 시나리오 1 — Self-Correction 경로 (에이전트 자율 회복)

**설정:** backend-dev가 환경변수 미설정으로 DB 연결 실패 반복

**예상 흐름:**
1. backend-dev가 3회 도구 호출 후 Self-Correction 자가 진단 실행
2. [C]=낮음, [D]=아니오 → ACP를 advisor에게 직접 SendMessage
3. advisor가 환경변수 주입 방법 교정 지시 발송
4. backend-dev가 교정 지시 따라 `.env.local` 확인 후 연결 성공
5. 오케스트레이터 모니터링 루프가 파일 변화 감지 → stagnation[backend-dev] = 0

**결과:** Gatekeeper 루프 개입 없이 자율 회복

### 시나리오 2 — Outer Loop 경로 (Gatekeeper 개입)

**설정:** frontend-dev가 API shape 불일치로 타입 에러 반복. Self-Correction 2회 시도했으나 advisor 교정도 효과 없음

**예상 흐름:**
1. frontend-dev Self-Correction → ACP → advisor (1회차)
2. advisor 교정 지시 → frontend-dev 시도 → 여전히 교착
3. Gatekeeper 루프: stagnation[frontend-dev] = 2 → advisor 호출 (2회차)
4. advisor 교정 지시 (다른 접근법) → frontend-dev 시도 → 여전히 교착
5. stagnation[frontend-dev] = 2, advisor_calls[frontend-dev] = 3 → Level 3 에스컬레이션
6. 사용자: "B. backend-dev에게 API shape 수정 요청"
7. 오케스트레이터: backend-dev에게 타입 수정 지시 → frontend-dev 재시도 성공

---

## 데이터 흐름도

```
사용자 요청
    │
    ▼
오케스트레이터 (skill.md)
    ├─── SendMessage ──► backend-dev ──► _workspace/backend/
    ├─── SendMessage ──► frontend-dev ──► _workspace/frontend/
    └─── SendMessage ──► qa-inspector ──► _workspace/qa-report/
                              ▲
          Gatekeeper 루프 (5분 주기)
          │ Glob + TaskGet → 진전 없음 감지
          │ stagnation[agent] >= 2
          ▼
        advisor ◄── ACP SendMessage (오케스트레이터 또는 에이전트)
          │
          └─── 교정 지시 SendMessage ──► 해당 에이전트
          └─── 요약 SendMessage ──► 오케스트레이터
```
