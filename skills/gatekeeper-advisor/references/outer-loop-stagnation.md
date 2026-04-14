# Outer Loop 상태 변화 감지 — 상세 구현

오케스트레이터가 외부에서 작업 에이전트의 교착 상태를 감지하는 상세 구현 패턴. Self-Correction(경로 1)이 에이전트 내부 감지라면, Outer Loop(경로 2)는 에이전트의 자가 인식 외부의 안전망이다.

---

## 목차

1. 상태 스냅샷 상세 설계
2. 교착 감지 알고리즘
3. 에이전트 팀 모드 통합 패턴
4. 서브 에이전트 모드 통합 패턴
5. 다중 에이전트 교착 (팀 전체 교착)
6. 에스컬레이션 단계 설계
7. 파인 튜닝 가이드

---

## 1. 상태 스냅샷 상세 설계

### 1-1. 스냅샷 항목 정의

스냅샷은 "이 이터레이션에서 실질적인 진전이 있었는가"를 판단하기 위한 관찰 가능한 지표만 포함한다. 에이전트의 주관적 보고("진행 중"이라는 메시지)는 포함하지 않는다 — 교착된 에이전트도 "진행 중"이라고 보고하기 때문이다.

```markdown
## 스냅샷 항목별 의의

| 항목 | 측정하는 것 | 한계 |
|------|-----------|------|
| workspace_files | 파일이 생성/수정됐는가 | 내용 없는 빈 파일 반복 생성은 변화로 오인 가능 |
| task_states | 작업이 실제로 전환됐는가 | 에이전트가 임의로 TaskUpdate할 수 있음 |
| output_summary | 에이전트가 새로운 내용을 말했는가 | 의미 없는 재시도 메시지 필터링 필요 |
```

### 1-2. 스냅샷 수집 타이밍

| 모드 | 스냅샷 수집 시점 |
|------|----------------|
| 에이전트 팀 | 각 팀원의 TaskUpdate(status: "done") 후 |
| 서브 에이전트 | 각 Agent 호출 완료 후 |
| 시간 기반 | 팀원이 done을 보고하지 않을 때 주기적 폴링 (예: 5분마다) |

### 1-3. 의미 없는 변화 필터

다음은 "상태 변화 있음"으로 인정하지 않는다:

```markdown
파일 변화 필터:
- 크기 0바이트 파일의 생성
- 동일 내용의 덮어쓰기 (mtime만 바뀌고 내용 동일)
- 임시 파일: *.tmp, *.lock, *.log, __pycache__/

태스크 상태 필터:
- pending → pending (변화 없음)
- done → done (재확인으로 인한 중복 TaskUpdate)

출력 요약 필터:
- "재시도 중입니다", "다시 시도합니다"
- 이전 이터레이션과 90% 이상 중복되는 텍스트
- 에러 메시지만 반복 (에러 내용이 동일하고 해결 시도가 없음)
```

---

## 2. 교착 감지 알고리즘

### 2-1. 단일 에이전트 교착 감지

```
입력: 스냅샷 시퀀스 [S_0, S_1, S_2, ..., S_N]
출력: 교착 여부 + 교착 시작 이터레이션

알고리즘:
  stagnation_streak = 0
  last_meaningful_snapshot = S_0

  for i in range(1, N+1):
    delta = compare_snapshots(S_{i-1}, S_i)
    
    if is_meaningful_change(delta):
      stagnation_streak = 0
      last_meaningful_snapshot = S_i
    else:
      stagnation_streak += 1
    
    if stagnation_streak >= STAGNATION_THRESHOLD:
      return STAGNANT(
        since_iteration = i - stagnation_streak,
        evidence = [S_{i-stagnation_streak}, ..., S_i]
      )
  
  return OK
```

### 2-2. compare_snapshots 정의

두 스냅샷을 비교하여 의미 있는 변화가 있는지 판단한다:

```markdown
compare_snapshots(S_prev, S_cur) → delta:

  file_delta:
    new_files = cur.workspace_files - prev.workspace_files
    modified_files = {f | f in prev AND f in cur AND f.size != prev_f.size}
    deleted_files = prev.workspace_files - cur.workspace_files
    
    meaningful_file_change = (
      new_files이 존재하고 비어있지 않음 OR
      modified_files이 존재하고 내용이 실질적으로 다름
    )
  
  task_delta:
    transitioned_tasks = {
      t | t.status in cur != t.status in prev
        AND NOT (t.status_prev == "done" AND t.status_cur == "done")
    }
    meaningful_task_change = len(transitioned_tasks) > 0
  
  output_delta:
    similarity = cosine_similarity(prev.output_summary, cur.output_summary)
    meaningful_output_change = similarity < 0.9  # 90% 미만 유사도일 때만 변화
    AND NOT is_retry_message(cur.output_summary)  # 재시도 메시지 필터
  
  return {
    is_meaningful: meaningful_file_change OR meaningful_task_change OR meaningful_output_change,
    evidence: { file_delta, task_delta, output_delta }
  }
```

### 2-3. STAGNATION_THRESHOLD 기본값과 조정 기준

| 작업 유형 | 권장 임계값 | 이유 |
|----------|-----------|------|
| 코드 구현 | 2 | 빠른 피드백 루프 필요, 2회면 패턴 충분히 확인됨 |
| 리서치/조사 | 3 | 검색 결과 없는 이터레이션이 자연스럽게 발생 가능 |
| 설계/분석 | 2 | 설계는 빠른 수렴이 어려우므로 빠른 Advisor 개입이 효과적 |
| 데이터 처리 | 2 | 변환 파이프라인 교착은 빠르게 감지해야 손실 최소화 |

---

## 3. 에이전트 팀 모드 통합 패턴

### 3-1. 리더 에이전트의 Gatekeeper 역할

에이전트 팀에서 리더(오케스트레이터)가 Outer Loop Gatekeeper를 담당한다. 팀원들이 태스크를 수행하는 동안 리더는 상태를 모니터링한다.

```markdown
### Phase 3 수정: Gatekeeper 내장 실행 루프

리더 에이전트 프롬프트에 추가할 섹션:

---
## Gatekeeper 모니터링 프로토콜

Phase 3 실행 중 다음을 병행한다:

**이터레이션 루프:**
각 이터레이션마다 (팀원의 idle 알림 또는 5분 타임아웃):

1. TaskGet으로 모든 작업 상태 조회
2. _workspace/ 파일 목록 확인 (Glob으로)
3. StateSnapshot 기록 (아래 포맷으로 _workspace/00_gatekeeper_log.md에 추가)
4. compare_snapshots(이전, 현재) 실행
5. 결과에 따라:
   - 의미 있는 변화 있음 → stagnation_count = 0, 계속 모니터링
   - 의미 있는 변화 없음 → stagnation_count++
6. stagnation_count >= 2:
   → 교착 팀원 식별 (마지막으로 TaskUpdate한 팀원 기준)
   → Advisor 에이전트 호출:
     Agent(
       subagent_type: "advisor",
       model: "opus",
       prompt: "[Advisor 호출 컨텍스트 포맷 + 교착 팀원 정보]"
     )
   → Advisor 출력을 교착 팀원에게 SendMessage

**StateSnapshot 기록 포맷:**
_workspace/00_gatekeeper_log.md에 추가:
```
### 이터레이션 {N} — {timestamp}

workspace_files:
- {파일경로} ({크기}, {mtime})

task_states:
- [{status}] {작업 제목} (담당: {팀원})

stagnation_count: {N}
판단: {의미있는변화 | 무변화}
```
---
```

### 3-2. 교착 팀원 식별 로직

팀 전체가 교착된 게 아니라 특정 팀원만 교착됐을 때, 그 팀원을 정확히 식별해야 Advisor를 효율적으로 활용할 수 있다.

```markdown
교착 팀원 식별:
1. 각 팀원별로 마지막 TaskUpdate 시간 기록
2. 이터레이션 간 TaskUpdate가 없는 팀원 → 교착 후보
3. 해당 팀원의 할당 작업(pending 또는 in_progress)이 있는지 확인
4. 있으면 → 해당 팀원 교착으로 판단
```

---

## 4. 서브 에이전트 모드 통합 패턴

### 4-1. 단일 작업 래퍼

하나의 Agent 호출을 Gatekeeper가 감싸는 패턴:

```markdown
단일 Agent 호출 Gatekeeper 래퍼:

function gatekeeper_run(task, max_attempts=5):
  attempt_count = 0
  stagnation_count = 0
  advisor_count = 0
  snapshots = []
  current_prompt = build_initial_prompt(task)
  
  while attempt_count < max_attempts:
    # 작업 에이전트 실행
    result = Agent(
      subagent_type: "worker",
      model: "opus",
      prompt: current_prompt
    )
    
    # 스냅샷 수집 및 비교
    snapshot = capture_snapshot(result, _workspace/)
    snapshots.append(snapshot)
    
    if is_task_complete(result):
      return SUCCESS(result)
    
    if len(snapshots) >= 2:
      delta = compare_snapshots(snapshots[-2], snapshots[-1])
      if delta.is_meaningful:
        stagnation_count = 0
      else:
        stagnation_count += 1
    
    # 교착 감지 → Advisor 호출
    if stagnation_count >= STAGNATION_THRESHOLD:
      if advisor_count >= MAX_ADVISOR_CALLS:
        return ESCALATE_TO_USER(task, snapshots, "반복 교착 — 사용자 개입 필요")
      
      advice = Agent(
        subagent_type: "advisor",
        model: "opus",
        prompt: build_advisor_context(task, snapshots)
      )
      
      current_prompt = incorporate_advice(current_prompt, advice)
      stagnation_count = 0
      advisor_count += 1
    
    attempt_count += 1
  
  return PARTIAL_RESULT(result, "최대 시도 횟수 도달")
```

### 4-2. 병렬 서브 에이전트에서 선택적 Gatekeeper

여러 서브 에이전트를 병렬 실행할 때, 완료된 에이전트는 건드리지 않고 교착된 에이전트만 재처리:

```markdown
병렬 실행 + 선택적 Gatekeeper:

Phase 2: 병렬 Agent 호출
  agents = [
    Agent(subagent_type: "researcher-1", run_in_background: true),
    Agent(subagent_type: "researcher-2", run_in_background: true),
    Agent(subagent_type: "researcher-3", run_in_background: true),
  ]

Phase 2.5: Gatekeeper 체크 (모든 에이전트 완료 후 또는 타임아웃 후)
  완료된 에이전트: 결과 수집, skip
  완료 안 된 에이전트:
    → 해당 에이전트의 partial output 확인
    → stagnation 여부 판단
    → 교착 확인 시 Advisor 호출 후 재실행
    → 또는: 해당 에이전트 결과 없이 진행하고 보고서에 누락 명시
```

---

## 5. 다중 에이전트 교착 (팀 전체 교착)

팀 전체가 교착되는 상황은 단일 에이전트 교착보다 드물지만, 발생 시 더 심각하다.

### 5-1. 팀 전체 교착 신호

다음 조건이 모두 충족되면 팀 전체 교착으로 판단한다:
- 모든 팀원의 작업이 in_progress 상태로 멈춤 (done으로 전환 없음)
- _workspace/ 파일 변화 없음 (2회 연속 이터레이션)
- 팀원 간 SendMessage 트래픽은 있지만 내용이 반복적

### 5-2. 팀 전체 교착 대응

```markdown
팀 전체 교착 대응:

1. 리더가 전체 팀에게 SendMessage({to: "all"}):
   "현재 작업 상태와 막힌 지점을 즉시 보고하라"

2. 각 팀원의 보고를 수집 (30초 내 응답 없으면 해당 팀원 교착 확인)

3. 교착 원인 분류:
   a. 팀원 A가 B의 결과를 기다리는데 B가 A의 결과를 기다리는 경우 (순환 의존)
      → 오케스트레이터가 직접 중재: 한쪽에 임시 placeholder 결과 제공
   
   b. 외부 데이터/API 접근 불가로 모두 멈춘 경우
      → Advisor에게 우회 전략 요청
   
   c. 요구사항 자체가 불명확하여 모두 진행 못 하는 경우
      → 사용자 에스컬레이션 (Advisor로 해결 불가)

4. 원인 분류 후 해당 전략 실행
   순환 의존: 리더가 직접 개입하여 의존성 끊기
   외부 접근 불가: Advisor 호출
   요구사항 불명확: 사용자에게 명확화 요청
```

---

## 6. 에스컬레이션 단계 설계

Gatekeeper는 단계적으로 에스컬레이션한다. 각 단계는 이전 단계가 효과 없을 때만 진행한다.

```
Level 0 (정상): 작업 에이전트가 자체 진행
    ↓ (stagnation_count >= 2 OR Self-Correction 임계값 충족)
Level 1 (Advisor 개입): Advisor가 새 접근법 제시, 작업 에이전트 재시도
    ↓ (advisor_count >= MAX_ADVISOR_CALLS AND 여전히 교착)
Level 2 (전략 변경): 오케스트레이터가 작업 분해 방식 자체를 변경
    ↓ (전략 변경 후에도 교착)
Level 3 (사용자 에스컬레이션): 사용자에게 상황 보고 + 결정 요청
```

### 6-1. Level 2 전략 변경 옵션

Advisor 호출을 반복해도 해결 안 될 때, 오케스트레이터가 시도할 수 있는 전략 변경:

| 전략 | 설명 | 적합한 경우 |
|------|------|-----------|
| **작업 분해** | 교착된 작업을 더 작은 단위로 나눔 | 작업이 너무 클 때 |
| **전문가 교체** | 다른 subagent_type으로 교체 | 현재 에이전트가 해당 도메인에 부적합 |
| **부분 결과 수용** | 교착 부분을 "미완료"로 표시하고 진행 | 해당 부분이 전체 차단 요소가 아닐 때 |
| **외부 도구 추가** | WebSearch/WebFetch 권한 추가 등 | 필요한 정보에 접근 불가할 때 |
| **작업 재정의** | 목표를 조금 다르게 설정 | 원래 목표가 실현 불가능할 때 |

### 6-2. Level 3 사용자 에스컬레이션 포맷

```markdown
사용자에게 전달할 에스컬레이션 보고:

## 작업 교착 — 사용자 개입 필요

**교착된 작업:** {작업 제목}
**시도 횟수:** {N}번
**Advisor 호출 횟수:** {N}회

**지금까지 시도한 것:**
1. {접근법 1} — 결과: {결과}
2. {접근법 2} — 결과: {결과}
...

**현재 막힌 이유:**
{근본 원인 분석}

**선택지:**
A. {대안 1 — 설명}
B. {대안 2 — 설명}
C. 이 부분을 건너뛰고 나머지 진행

어떻게 진행할까요?
```

---

## 7. 파인 튜닝 가이드

### 7-1. 임계값 조정 신호

아래 패턴이 보이면 STAGNATION_THRESHOLD 조정이 필요하다:

| 패턴 | 의미 | 조정 |
|------|------|------|
| Advisor가 너무 자주 호출됨 (정상 진행 중에도) | 임계값이 너무 낮음 | THRESHOLD 올리기 (2→3) |
| 루프가 길게 지속된 후 뒤늦게 Advisor 호출됨 | 임계값이 너무 높음 | THRESHOLD 낮추기 (3→2) |
| Advisor 출력 후에도 교착 반복 | Advisor 컨텍스트 품질 문제 | 컨텍스트 포맷 개선 |

### 7-2. MAX_ADVISOR_CALLS 조정

| 작업 중요도 | 권장 값 | 이유 |
|-----------|--------|------|
| 높음 (핵심 기능) | 3~5 | 충분한 재시도 허용 |
| 중간 (보조 기능) | 2~3 | 비용 대비 효과 균형 |
| 낮음 (부가 기능) | 1~2 | 빠른 부분 결과 수용 |

### 7-3. 스냅샷 수집 비용 고려

스냅샷 수집은 컨텍스트를 소모한다. 대형 _workspace/가 있을 때:
- 전체 파일 목록 대신 크기 합계와 파일 수만 비교
- 최근 N개 파일만 비교 (가장 최근 수정된 것들)
- output_summary는 500자로 제한
