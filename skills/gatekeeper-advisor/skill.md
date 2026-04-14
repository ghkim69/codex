---
name: gatekeeper-advisor
description: "Sonnet(작업 에이전트)이 언제 Advisor를 호출할지 결정하는 Gatekeeper 판단 로직. (1) 작업 에이전트가 루프에 빠지거나 진전이 없을 때, (2) 자가 진단 점수가 임계값 이하일 때, (3) 외부 루프(Outer Loop)에서 상태 변화가 N회 이상 없을 때, (4) 오케스트레이터가 에이전트 교착 상태를 감지할 때 반드시 이 스킬을 사용할 것."
---

# Gatekeeper Advisor — 언제 Advisor를 호출할지 결정하는 판단 로직

작업 에이전트(Sonnet)가 스스로 막혔음을 감지하거나, 외부 루프에서 상태 변화가 없음을 감지할 때 Advisor를 호출하는 판단 체계. 무조건 Advisor를 호출하면 비용이 급증하고, 호출하지 않으면 루프에 갇혀 작업이 실패한다. Gatekeeper는 이 두 실패 모드 사이의 균형점이다.

---

## 두 가지 트리거 경로

Gatekeeper는 두 방향에서 작동한다:

| 경로 | 주체 | 감지 방식 | 결과 |
|------|------|----------|------|
| **Self-Correction** | 작업 에이전트 자신 | 각 이터레이션 후 자가 진단 | 임계값 충족 시 Advisor 호출 |
| **Outer Loop** | 오케스트레이터 | 상태 스냅샷 비교 | N회 무변화 시 Advisor 주입 |

두 경로는 독립적으로 작동한다. Self-Correction은 에이전트 내부에서 빠르게 감지하고, Outer Loop는 에이전트가 자신의 교착을 인식 못할 때 외부에서 잡는 안전망이다.

---

## 경로 1: Self-Correction 프롬프트

### 1-1. 자가 진단 시점

작업 에이전트는 다음 시점에 자가 진단을 실행한다:
- 각 주요 도구 호출 사이클 완료 후 (파일 쓰기, 코드 실행, 검색 등 3회 연속 후)
- 의미 있는 산출물이 나오지 않은 연속 2회 시도 후
- 에러가 발생했지만 원인을 확신할 수 없을 때
- 다음 단계가 2개 이상이고 어느 쪽을 선택할지 불확실할 때

### 1-2. 자가 진단 프롬프트 템플릿

```
## 자가 진단 체크포인트

현재 작업: {현재 수행 중인 태스크 제목}
시도 횟수: {N}번째 시도

다음 질문에 솔직하게 답한다:

[P] 진행률: 목표 대비 현재 완료 비율은? (0 / 25 / 50 / 75 / 100)
[C] 신뢰도: 현재 접근법이 성공할 확신은? (낮음 / 중간 / 높음)
[D] 다양성: 마지막 N번의 시도가 서로 의미 있게 달랐는가? (예 / 아니오)
[B] 장애물: 해결 안 된 핵심 장애물이 지금 있는가? (있음 / 없음)
[R] 자원: 소진한 컨텍스트/토큰 대비 진전이 충분한가? (예 / 아니오)

판단:
```

판단 규칙은 아래 "1-3. 결정 규칙 테이블"에 따른다.

### 1-3. 결정 규칙 테이블

| 조건 | 판단 | 행동 |
|------|------|------|
| `[C]=낮음` AND `[D]=아니오` | Advisor 호출 필수 | 현재 상태를 컨텍스트로 전달 |
| `[D]=아니오` 3회 연속 | Advisor 호출 필수 | 반복 패턴 로그 포함 |
| `[B]=있음` AND `[C]=낮음` | Advisor 호출 권장 | 장애물 설명 포함 |
| `[P]≤25` AND `[R]=아니오` | Advisor 호출 필수 | 자원 소진 경고 포함 |
| `[C]=중간` AND `[D]=예` | 계속 진행 | 다음 시도 진행 |
| `[C]=높음` | 계속 진행 | Advisor 불필요 |

**왜 이 규칙인가:** "낮은 신뢰도 + 반복 패턴"은 에이전트가 같은 실수를 반복하고 있다는 신호다. 이 상태에서 계속 진행하면 토큰만 소모하고 결과는 나오지 않는다. Advisor는 새로운 관점으로 접근법을 재설정할 수 있다.

### 1-4. Advisor 호출 컨텍스트 — ACP 포맷

Advisor(Opus)는 고성능이지만 토큰 비용도 높다. 전체 히스토리를 그대로 전달하면 Advisor가 노이즈를 걸러내는 데 추론을 낭비하고, 실제 진단 품질은 오히려 떨어진다. **ACP(Advisor Context Package)**는 Advisor가 진단에 필요한 정보만 추출한 압축 포맷이다.

```
## ACP (Advisor Context Package)

[TRIGGER] {결정 규칙 코드 — 예: C=낮음+D=아니오, D×3회, P≤25+R=아니오}
[GOAL]    {달성하려는 목표 — 2문장 이내, 구현 세부사항 제외}
[HARD_CONSTRAINTS] {변경 불가능한 기술 제약 — 예: Node 18, Postgres 14, 외부 API 스펙 고정}
[ATTEMPTS]
  A1: [{접근법 유형}] {시도 내용 한 줄} → {결과 5단어}
  A2: [{접근법 유형}] {시도 내용 한 줄} → {결과 5단어}
  ×N: [반복] {동일 접근 N회 반복} → {동일 에러}
[BLOCKER]
  type: {에러 유형 — TypeError/NetworkError/LogicError/등}
  loc:  {파일:라인 또는 "없음(출력 부재)"}
  msg:  {에러 메시지 첫 줄, 100자 이내}
[HYPOTHESIS]
  H1 ({신뢰도}%): {가설 한 문장}
  H2 ({신뢰도}%): {가설 한 문장}
[ASK] {Advisor에게 원하는 것 — 한 문장, 가능하면 Yes/No 또는 선택지 형태}
```

**토큰 버짓 목표:** 전체 ACP ≤ 500 토큰. 각 섹션의 세부 한도와 압축 규칙은 `references/context-compression.md` 참조.

> 도메인별 ACP 작성 예시(압축 전/후 비교 포함): `references/self-correction-prompts.md` 참조.

---

## 경로 2: Outer Loop 상태 변화 감지

### 2-1. 왜 Outer Loop가 필요한가

Self-Correction은 에이전트가 자신의 교착 상태를 인식할 때만 작동한다. 하지만 에이전트가 "진행 중"이라고 착각하면서 실제로는 무의미한 작업을 반복하는 경우가 있다. Outer Loop는 오케스트레이터가 외부에서 실제 상태 변화를 측정하여 이를 잡는다.

### 2-2. 상태 스냅샷 구조

오케스트레이터는 매 이터레이션(또는 Phase 완료 시) 상태 스냅샷을 기록한다:

```
StateSnapshot = {
  iteration:    int,           # 이터레이션 번호
  timestamp:    string,        # ISO 8601
  workspace_files: [           # _workspace/ 파일 목록
    { path: string, size: int, mtime: string }
  ],
  task_states: [               # 작업 목록 상태
    { id: string, title: string, status: "pending|in_progress|done|failed" }
  ],
  output_summary: string       # 에이전트 최근 출력의 핵심 요약 (100자 이내)
}
```

### 2-3. 상태 변화 판단 기준

두 스냅샷 사이에 다음 중 하나라도 변하면 "상태 변화 있음"으로 판단:
- `workspace_files` 내 파일 수 변화, 또는 기존 파일 `size`/`mtime` 변화
- `task_states` 내 어떤 작업의 `status` 변화
- `output_summary`가 이전과 의미 있게 다름 (단순 재시도 메시지 제외)

**의미 없는 변화 (상태 변화로 인정 안 함):**
- 임시 파일 생성/삭제 (`*.tmp`, `*.lock`)
- 동일 내용으로 덮어쓰기
- "재시도 중...", "다시 시도합니다" 같은 반복 로그

### 2-4. 교착 임계값과 에스컬레이션

```
교착 카운터 로직:

이터레이션마다:
  if has_meaningful_state_change(snapshot[N], snapshot[N-1]):
    stagnation_count = 0
  else:
    stagnation_count += 1

  if stagnation_count >= STAGNATION_THRESHOLD:
    → Advisor 호출 (경로 2)
    stagnation_count = 0  # 리셋 후 재모니터링

기본값:
  STAGNATION_THRESHOLD = 2  # 연속 2회 무변화 → Advisor 호출
  MAX_ADVISOR_CALLS = 3     # Advisor를 3회 이상 호출해도 해결 안 되면 → 사용자 에스컬레이션
```

**왜 임계값이 2인가:** 1회 무변화는 일시적 지연일 수 있다. 2회 연속이면 패턴이다. 3회 이상으로 설정하면 낭비가 커진다.

### 2-5. 오케스트레이터 통합 코드 패턴

에이전트 팀 모드에서 Outer Loop를 통합하는 방법:

```markdown
### Phase 3: 실행 + Gatekeeper 모니터링

각 이터레이션마다:
1. 팀원들이 작업 수행 (자체 조율)
2. 오케스트레이터가 TaskGet으로 상태 조회
3. StateSnapshot 기록
4. 상태 변화 판단:
   - 변화 있음 → stagnation_count 리셋, 계속 진행
   - 변화 없음 → stagnation_count 증가
5. stagnation_count >= 2 → Advisor 에이전트 주입:
   Agent(
     subagent_type: "advisor",
     model: "opus",
     prompt: "[ACP 포맷으로 압축한 컨텍스트 — TRIGGER=Outer Loop, ATTEMPTS는 스냅샷 델타로 구성]"
   )
   # ACP 구성 상세: references/context-compression.md의 "Outer Loop ACP" 섹션 참조
6. Advisor 출력을 막힌 팀원에게 SendMessage로 전달
7. 재모니터링 시작
```

서브 에이전트 모드에서:

```markdown
각 Agent 호출을 래핑:

attempt = 0
while attempt < MAX_ATTEMPTS:
  result = Agent(subagent_type: "worker", prompt: current_task)
  snapshot = capture_snapshot(result)
  
  if has_meaningful_progress(snapshot):
    break
  
  attempt += 1
  if attempt >= STAGNATION_THRESHOLD:
    advisor_output = Agent(
      subagent_type: "advisor",
      model: "opus",
      prompt: build_acp(current_task, snapshot_history)
      # build_acp: 전체 히스토리 대신 ACP 포맷으로 압축
      # 구현 상세: references/context-compression.md 참조
    )
    current_task = incorporate_advice(current_task, advisor_output)
    attempt = 0  # 리셋

if attempt >= MAX_ATTEMPTS:
  → 사용자 에스컬레이션
```

> 더 복잡한 상태 추적 패턴과 다중 에이전트 교착 감지는 `references/outer-loop-stagnation.md` 참조.

---

## Advisor 에이전트 연결

Gatekeeper가 Advisor를 호출할 때, Advisor는 다음을 수행한다:
1. 호출 컨텍스트를 읽고 교착 원인 분석
2. 현재 접근법의 근본 문제 진단
3. 구체적인 대안 접근법 또는 다음 단계 제시
4. 작업 에이전트에게 actionable한 지시로 전달

Advisor는 항상 `model: "opus"` (고성능 모델)를 사용하고, 작업 에이전트는 Advisor의 지시를 따른다.

> Advisor 에이전트 정의 템플릿: `references/self-correction-prompts.md`의 "Advisor 에이전트 정의" 섹션 참조.

---

## 통합 흐름도

```
[작업 에이전트 이터레이션]
        │
        ▼
  [Self-Correction 자가 진단]
        │
   임계값 충족?
   ├── Yes → [Advisor 호출] ──────────────────┐
   └── No  → 계속 진행                        │
                                              │
[오케스트레이터 Outer Loop 모니터링]           │
        │                                    │
  상태 변화 있음?                             │
  ├── Yes → stagnation_count = 0, 계속       │
  └── No  → stagnation_count++               │
               │                             │
          count >= 2?                        │
          ├── Yes → [Advisor 호출] ──────────┤
          └── No  → 다음 이터레이션           │
                                              │
                                    [Advisor 분석 및 제시]
                                              │
                                    [작업 에이전트에 지시 전달]
                                              │
                                    [재이터레이션 시작]
                                              │
                                    MAX_ADVISOR_CALLS 초과?
                                    ├── Yes → 사용자 에스컬레이션
                                    └── No  → 계속
```

---

## 참고

- 도메인별 자가 진단 프롬프트 변형 및 Advisor 에이전트 정의: `references/self-correction-prompts.md`
- Outer Loop 상태 추적 상세 구현 및 다중 에이전트 교착 감지: `references/outer-loop-stagnation.md`
- **ACP 압축 알고리즘, 토큰 버짓, 섹션별 압축 규칙**: `references/context-compression.md`
