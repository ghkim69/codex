# 오케스트레이터 통합 전체 예시

Gatekeeper-Advisor 패턴을 실제 오케스트레이터에 통합한 완전한 예시.
에이전트 팀 모드와 서브 에이전트 모드 두 가지를 제공한다.

---

## 목차

1. 에이전트 팀 모드 통합 (코드 리뷰 팀 예시)
2. 서브 에이전트 모드 통합 (리서치 파이프라인 예시)
3. 두 모드 선택 기준

---

## 1. 에이전트 팀 모드 — 코드 리뷰 팀 + Gatekeeper

코드 리뷰 팀이 보안/성능/테스트를 병렬 검토하는 상황에서,
리뷰어 중 한 명이 교착에 빠졌을 때 Gatekeeper가 자동으로 Advisor를 호출하는 예시.

```markdown
---
name: code-review-orchestrator
description: "코드 리뷰 팀을 조율하는 오케스트레이터. Gatekeeper-Advisor 패턴 내장."
---

# Code Review Orchestrator

보안/성능/테스트 리뷰어를 병렬 실행하고, 교착 리뷰어는 Gatekeeper가 자동 개입한다.

## 실행 모드: 에이전트 팀

## 에이전트 구성

| 팀원 | 타입 | 역할 | 출력 |
|------|------|------|------|
| security-reviewer | 커스텀 | 보안 취약점 검토 | `_workspace/01_security.md` |
| perf-reviewer | 커스텀 | 성능 및 복잡도 검토 | `_workspace/01_perf.md` |
| test-reviewer | 커스텀 | 테스트 커버리지 검토 | `_workspace/01_test.md` |
| advisor | 커스텀 | 교착 리뷰어 지원 (Gatekeeper 호출 시) | 없음 (즉시 반환) |

## 워크플로우

### Phase 1: 준비

1. 리뷰 대상 파일 목록 파악 (Read/Glob으로 변경된 파일 확인)
2. `_workspace/` 생성, 리뷰 범위를 `_workspace/00_scope.md`에 저장

### Phase 2: 팀 구성

```
TeamCreate(
  team_name: "code-review-team",
  members: [
    { name: "security-reviewer", agent_type: "security-reviewer",
      model: "opus",
      prompt: "보안 리뷰어 역할. _workspace/00_scope.md에 나열된 파일을
               OWASP Top 10 기준으로 검토하고 _workspace/01_security.md에 결과 저장.
               완료 시 리더에게 idle 알림." },
    { name: "perf-reviewer", agent_type: "perf-reviewer",
      model: "opus",
      prompt: "성능 리뷰어 역할. _workspace/00_scope.md 파일을
               시간/공간 복잡도, N+1 쿼리, 캐싱 누락 관점으로 검토.
               _workspace/01_perf.md에 결과 저장. 완료 시 idle 알림." },
    { name: "test-reviewer", agent_type: "test-reviewer",
      model: "opus",
      prompt: "테스트 리뷰어 역할. _workspace/00_scope.md 파일의
               테스트 커버리지, 엣지 케이스 누락, 모킹 적절성 검토.
               _workspace/01_test.md에 결과 저장. 완료 시 idle 알림." },
    { name: "advisor", agent_type: "advisor",
      model: "opus",
      prompt: "교착 해소 Advisor. Gatekeeper가 ACP를 전달할 때만 응답.
               진단 결과와 권장 접근법을 교착 팀원에게 SendMessage로 전달." }
  ]
)

TaskCreate(tasks: [
  { title: "보안 검토", assignee: "security-reviewer" },
  { title: "성능 검토", assignee: "perf-reviewer" },
  { title: "테스트 검토", assignee: "test-reviewer" }
])
```

### Phase 3: 병렬 리뷰 + Gatekeeper 모니터링

**팀원들이 자체 조율하며 병렬 실행한다.**

**Gatekeeper 모니터링 (리더가 병행):**

```
# 내부 상태 변수 초기화
stagnation_count = {reviewer: 0 for reviewer in ["security", "perf", "test"]}
advisor_count = 0
prev_snapshot = null

# 주기적 실행 (팀원 idle 알림 수신 또는 5분마다)
loop:
  current_snapshot = {
    files: Glob("_workspace/01_*.md") → 파일별 크기+mtime,
    tasks: TaskGet() → 상태별 분류
  }

  for reviewer in ["security", "perf", "test"]:
    if task_{reviewer}.status == "done":
      continue  # 완료된 리뷰어는 건너뜀

    reviewer_file = "_workspace/01_{reviewer}.md"
    file_changed = (reviewer_file의 mtime이 prev와 다름)
    task_advanced = (reviewer task가 전환됨)

    if file_changed OR task_advanced:
      stagnation_count[reviewer] = 0
    else:
      stagnation_count[reviewer] += 1

    if stagnation_count[reviewer] >= 2 AND advisor_count < 3:
      # ACP 구성 (context-compression.md 참조)
      acp = """
        ## ACP
        [TRIGGER] OL:stagnation×{stagnation_count[reviewer]}+task_frozen:{reviewer}-reviewer
        [GOAL]    {reviewer} 관점 코드 리뷰 완료 (_workspace/01_{reviewer}.md 생성)
        [HARD_CONSTRAINTS] 리뷰 범위: _workspace/00_scope.md 파일 목록만
        [ATTEMPTS]
          ×{stagnation_count[reviewer]}: workspace 파일 변화 없음, 태스크 전환 없음
        [BLOCKER]
          type: 출력 부재
          loc:  {reviewer}-reviewer / "{reviewer} 검토" 태스크
          msg:  리뷰 파일 미생성, 마지막 출력: {마지막_idle_메시지}
        [HYPOTHESIS]
          H1 (60%): 리뷰 범위가 너무 넓거나 파일 접근에 문제
          H2 (30%): {reviewer} 검토 기준이 불명확하여 시작 못함
        [ASK] {reviewer}-reviewer가 지금 당장 시작할 수 있는 첫 번째 리뷰 파일과 체크 항목은?
      """

      SendMessage(to: "advisor", message: acp)
      # advisor가 {reviewer}-reviewer에게 직접 지시 전달
      stagnation_count[reviewer] = 0
      advisor_count += 1

    if advisor_count >= 3 AND any(stagnation_count[r] >= 2):
      → 사용자에게 보고 및 해당 리뷰어 결과 없이 진행 여부 확인

  prev_snapshot = current_snapshot
```

### Phase 4: 통합

1. 모든 `_workspace/01_*.md` 파일 Read
2. 발견 사항을 심각도별로 통합
3. 최종 리뷰 보고서 생성

### Phase 5: 정리

1. TeamDelete
2. `_workspace/` 보존
3. 최종 보고서 사용자 전달

## 데이터 흐름

```
입력 → [Phase 1] → _workspace/00_scope.md
                         │
                    [Phase 2] TeamCreate
                    ├── security-reviewer ──→ _workspace/01_security.md ─┐
                    ├── perf-reviewer ───────→ _workspace/01_perf.md ────┼→ [Phase 4] 통합
                    ├── test-reviewer ───────→ _workspace/01_test.md ────┘
                    └── advisor (대기)
                         │
                    [Phase 3] Gatekeeper 모니터링
                    stagnation 감지 시:
                    리더 → advisor(ACP) → 교착_팀원
```

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| 리뷰어 1명 교착 | Gatekeeper → Advisor 호출 → 리뷰어에게 지시 |
| Advisor 3회 실패 | 해당 리뷰어 결과 없이 통합 (보고서에 누락 명시) |
| advisor 에이전트 응답 없음 | 1회 재시도 후 실패하면 오케스트레이터가 직접 지시 |
| 과반 리뷰어 교착 | 사용자에게 알리고 진행 여부 확인 |
```

---

## 2. 서브 에이전트 모드 — 리서치 파이프라인 + 래퍼

단일 리서치 에이전트를 Gatekeeper 래퍼로 감싸서 교착 시 자동 재시도하는 예시.

```markdown
---
name: research-orchestrator
description: "리서치 파이프라인 오케스트레이터. Gatekeeper-Advisor 래퍼로 교착 자동 처리."
---

# Research Orchestrator

단일 리서치 에이전트를 Gatekeeper 래퍼 안에서 실행하고, 교착 시 자동으로 Advisor를 개입시킨다.

## 실행 모드: 서브 에이전트

## 워크플로우

### Phase 1: 준비

1. 리서치 주제 파악, `_workspace/00_topic.md`에 저장
2. 성공 기준 정의: "어떤 파일이 생성되면 완료인가"

### Phase 2: Gatekeeper 래퍼 실행

```
# 초기화
attempt = 0
stagnation_count = 0
advisor_count = 0
snapshots = []
current_prompt = Read("_workspace/00_topic.md")

while attempt < 6:  # 최대 6회 시도

  result = Agent(
    subagent_type: "researcher",
    model: "opus",
    prompt: current_prompt,
    run_in_background: false
  )

  # 스냅샷 수집
  snapshot = {
    files: Glob("_workspace/02_*.md"),  # 리서치 산출물 경로
    output: result의 핵심 요약 (100자)
  }
  snapshots.append(snapshot)

  # 완료 판단
  if "_workspace/02_research.md" 파일 존재 AND 내용이 충분:
    break  # 성공

  # 진행 여부 판단
  if len(snapshots) >= 2:
    prev = snapshots[-2]
    curr = snapshots[-1]
    file_delta = (curr.files != prev.files)  # 파일 수/크기 변화
    output_delta = (curr.output와 prev.output의 유사도 < 90%)

    if file_delta OR output_delta:
      stagnation_count = 0
    else:
      stagnation_count += 1

  # 교착 감지
  if stagnation_count >= 2:
    if advisor_count >= 3:
      → 사용자 에스컬레이션 (outer-loop-stagnation.md Level 3 포맷)
      break

    # ACP 구성
    acp = build_acp_from_snapshots(
      topic = Read("_workspace/00_topic.md"),
      snapshots = snapshots[-stagnation_count:],
      trigger = "OL:stagnation×{stagnation_count}"
    )

    advice = Agent(
      subagent_type: "advisor",
      model: "opus",
      prompt: acp
    )

    # current_prompt에 Advisor 지시 통합
    current_prompt = f"""
      원래 리서치 주제:
      {Read("_workspace/00_topic.md")}

      Advisor 지시:
      {advice.권장접근법}

      위 Advisor 지시에 따라 리서치를 수행하고 결과를 _workspace/02_research.md에 저장.
    """

    stagnation_count = 0
    advisor_count += 1

  attempt += 1

# 결과 처리
if "_workspace/02_research.md" 존재:
  → Phase 3: 결과 정제
else:
  → 부분 결과로 계속 (보고서에 미완료 명시)
```

### Phase 3: 결과 정제

1. `_workspace/02_research.md` Read
2. 핵심 인사이트 추출
3. 최종 보고서 생성

### Phase 4: 정리

`_workspace/` 보존, 결과 보고

## 에러 핸들링

| 상황 | 전략 |
|------|------|
| 리서치 에이전트 실패 | 1회 재시도. 재실패 시 stagnation_count 증가로 처리 |
| Advisor 3회 후 교착 지속 | 사용자 에스컬레이션 |
| 타임아웃 | 현재까지의 부분 결과 사용, 누락 명시 |
```

---

## 3. 두 모드 선택 기준

| 상황 | 권장 | 이유 |
|------|------|------|
| 여러 에이전트가 병렬로 다른 역할 수행 | **에이전트 팀** | 팀원 간 교착 개별 감지, SendMessage로 advisor 연결 |
| 단일 에이전트 반복 실행 | **서브 에이전트 래퍼** | 단순 루프 구조, 팀 구성 오버헤드 없음 |
| 교착이 팀원 간 의존성에서 발생할 가능성 | **에이전트 팀** | 순환 의존 감지를 Gatekeeper가 처리 |
| 교착이 단일 작업의 기술적 막힘 | **서브 에이전트 래퍼** | Advisor에게 ACP만 전달하면 충분 |
| Gatekeeper를 여러 오케스트레이터에서 재사용 | **독립 Gatekeeper 에이전트** | agent-definitions.md의 방식 A 사용 |

### 통합 시 공통 규칙

- **ACP는 항상 680 토큰 이내**: `references/context-compression.md` 규칙 적용
- **advisor는 항상 `model: "opus"`**: 진단 품질이 핵심
- **stagnation_count 리셋 잊지 않기**: Advisor 호출 후 반드시 0으로 리셋
- **_workspace/00_gatekeeper_log.md에 기록**: 교착 감지 이력 보존 (사후 분석용)
