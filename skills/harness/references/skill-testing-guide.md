# 스킬 테스트 & 반복 개선 가이드

하네스에서 생성한 스킬의 품질을 검증하고 반복적으로 개선하는 방법론. SKILL.md Phase 6의 보충 레퍼런스.

---

## 목차

1. [테스트 프레임워크 개요](#1-테스트-프레임워크-개요)
2. [테스트 프롬프트 작성법](#2-테스트-프롬프트-작성법)
3. [실행 테스트: With-skill vs Baseline](#3-실행-테스트-with-skill-vs-baseline)
4. [정량적 평가: Assertion 기반 채점](#4-정량적-평가-assertion-기반-채점)
5. [전문 에이전트 활용](#5-전문-에이전트-활용)
6. [반복 개선 루프](#6-반복-개선-루프)
7. [Description 트리거 검증](#7-description-트리거-검증)
8. [워크스페이스 구조](#8-워크스페이스-구조)
9. [행동 패턴 테스트 (Gatekeeper-Advisor)](#9-행동-패턴-테스트) — 타이밍·상태 기반 검증

---

## 1. 테스트 프레임워크 개요

스킬 품질 검증은 **정성적 평가**와 **정량적 평가**의 조합이다.

| 평가 유형 | 방법 | 적합한 스킬 |
|----------|------|-----------|
| **정성적** | 사용자가 산출물을 직접 리뷰 | 문체, 디자인, 창작물 등 주관적 품질 |
| **정량적** | assertion 기반 자동 채점 | 파일 생성, 데이터 추출, 코드 생성 등 객관적 검증 가능 |

핵심 루프: **작성 → 테스트 실행 → 평가 → 개선 → 재테스트**

---

## 2. 테스트 프롬프트 작성법

### 원칙

테스트 프롬프트는 **실제 사용자가 입력할 법한 구체적이고 자연스러운 문장**이어야 한다. 추상적이거나 인공적인 프롬프트는 테스트 가치가 낮다.

### 나쁜 예

```
"PDF를 처리하라"
"데이터를 추출하라"
"차트를 생성하라"
```

### 좋은 예

```
"다운로드 폴더에 있는 'Q4_매출_최종_v2.xlsx'에서 C열(매출)과 D열(비용)을
사용해서 이익률(%) 열을 추가해줘. 그리고 이익률 기준으로 내림차순 정렬."
```

```
"이 PDF에서 3페이지 표를 추출해서 CSV로 변환해줘. 표 헤더가 2줄로
되어 있어서 첫 번째 줄은 카테고리, 두 번째 줄이 실제 열 이름이야."
```

### 프롬프트 다양성

- **공식적 / 캐주얼** 톤 혼합
- **명시적 / 암시적** 의도 혼합 (파일 형식을 직접 말하는 경우 vs 맥락으로 추론해야 하는 경우)
- **단순 / 복잡** 작업 혼합
- 일부는 약어, 오타, 캐주얼한 표현 포함

### 커버리지

2~3개 프롬프트로 시작하되, 다음을 커버하도록 설계:
- 핵심 사용 사례 1개
- 엣지 케이스 1개
- (선택) 복합 작업 1개

---

## 3. 실행 테스트: With-skill vs Baseline

### 3-1. 비교 실행 구조

각 테스트 프롬프트에 대해 두 개의 서브에이전트를 **동시에** 스폰한다:

**With-skill 실행:**
```
프롬프트: "{테스트 프롬프트}"
스킬 경로: {스킬 경로}
출력 경로: _workspace/iteration-N/eval-{id}/with_skill/outputs/
```

**Baseline 실행:**
```
프롬프트: "{테스트 프롬프트}"  (동일)
스킬: 없음
출력 경로: _workspace/iteration-N/eval-{id}/without_skill/outputs/
```

### 3-2. Baseline 선택

| 상황 | Baseline |
|------|----------|
| 새 스킬 생성 | 스킬 없이 같은 프롬프트 실행 |
| 기존 스킬 개선 | 수정 전 스킬 버전 (스냅샷 보존) |

### 3-3. 타이밍 데이터 캡처

서브에이전트 완료 알림에서 `total_tokens`와 `duration_ms`를 **즉시** 저장한다. 이 데이터는 알림 시점에만 접근 가능하고 이후 복구할 수 없다.

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

---

## 4. 정량적 평가: Assertion 기반 채점

### 4-1. Assertion 작성

산출물이 객관적으로 검증 가능한 경우, 자동 채점을 위한 assertion을 정의한다.

**좋은 assertion:**
- 객관적으로 참/거짓 판별 가능
- 서술적인 이름으로 결과만 봐도 무엇을 검사하는지 명확
- 스킬의 핵심 가치를 검증

**나쁜 assertion:**
- 스킬 유무와 무관하게 항상 통과하는 것 (예: "출력이 존재한다")
- 주관적 판단이 필요한 것 (예: "잘 작성되었다")

### 4-2. 프로그래밍 가능한 검증

assertion이 코드로 검증 가능하면 스크립트로 작성한다. 눈으로 확인하는 것보다 빠르고 신뢰성 있으며, iteration마다 재사용 가능.

### 4-3. Non-discriminating assertion 주의

"두 구성 모두에서 100% 통과"하는 assertion은 스킬의 차별적 가치를 측정하지 못한다. 이런 assertion을 발견하면 제거하거나, 더 도전적인 assertion으로 교체한다.

### 4-4. 채점 결과 스키마

```json
{
  "expectations": [
    {
      "text": "이익률 열이 추가됨",
      "passed": true,
      "evidence": "E열에 'profit_margin_pct' 열 확인"
    },
    {
      "text": "이익률 기준 내림차순 정렬",
      "passed": false,
      "evidence": "정렬 없이 원본 순서 유지됨"
    }
  ],
  "summary": {
    "passed": 1,
    "failed": 1,
    "total": 2,
    "pass_rate": 0.50
  }
}
```

---

## 5. 전문 에이전트 활용

테스트/평가 과정에서 전문 역할의 에이전트를 활용하면 품질이 향상된다.

### 5-1. Grader (채점자)

assertion 기반 채점을 수행하고, 산출물에서 검증 가능한 주장(claim)을 추출하여 교차 검증한다.

**역할:**
- assertion별 통과/실패 판정 + 근거 제시
- 산출물에서 사실적 주장을 추출하고 검증
- eval 자체의 품질에 대한 피드백 (assertion이 너무 쉽거나 모호한 경우 제안)

### 5-2. Comparator (블라인드 비교자)

두 산출물을 A/B로 익명화하여, 어떤 것이 스킬을 사용한 결과인지 모르는 상태에서 품질을 판정한다.

**활용 시점:** "새 버전이 정말 더 나은가?"를 엄밀하게 확인하고 싶을 때. 일반적인 반복 개선에서는 생략 가능.

**판정 기준:**
- 내용: 정확성, 완성도
- 구조: 조직화, 포맷팅, 사용성
- 종합 점수

### 5-3. Analyzer (분석자)

벤치마크 데이터에서 통계적 패턴을 분석한다:
- Non-discriminating assertion (두 구성 모두 통과 → 차별력 없음)
- 고분산 eval (결과가 실행마다 크게 달라짐 → 불안정)
- 시간/토큰 트레이드오프 (스킬이 품질은 높이지만 비용도 높이는 경우)

---

## 6. 반복 개선 루프

### 6-1. 피드백 수집

사용자에게 산출물을 보여주고 피드백을 받는다. 빈 피드백은 "이상 없음"으로 해석한다.

### 6-2. 개선 원칙

1. **피드백을 일반화하라** — 테스트 예시에만 맞는 좁은 수정은 오버피팅이다. 원리 수준에서 수정한다.
2. **무게를 벌지 않는 것은 제거하라** — 트랜스크립트를 읽고, 스킬이 에이전트에게 비생산적인 작업을 시키고 있다면 해당 부분을 삭제한다.
3. **Why를 설명하라** — 사용자의 피드백이 간결하더라도, 왜 그것이 중요한지 이해하고 그 이해를 스킬에 반영한다.
4. **반복 작업은 번들링하라** — 모든 테스트 실행에서 동일한 헬퍼 스크립트가 생성되면, `scripts/`에 미리 포함한다.

### 6-3. 반복 절차

```
1. 스킬 수정
2. 새 iteration-N+1/ 디렉토리에 모든 테스트 케이스 재실행
3. 사용자에게 결과 제시 (이전 iteration과 비교)
4. 피드백 수집
5. 다시 수정 → 반복
```

**종료 조건:**
- 사용자가 만족
- 피드백이 모두 비어 있음 (모든 산출물 이상 없음)
- 의미 있는 개선이 더 이상 없음

### 6-4. 초안 → 재검토 패턴

스킬 수정 시, 초안을 작성한 후 **새로운 시각으로 다시 읽고** 개선한다. 한 번에 완벽하게 쓰려 하지 말고, 초안-검토 사이클을 거친다.

---

## 7. Description 트리거 검증

### 7-1. 트리거 Eval 쿼리 작성

20개의 eval 쿼리를 작성한다 — should-trigger 10개 + should-NOT-trigger 10개.

**쿼리 품질 기준:**
- 실제 사용자가 입력할 법한 구체적이고 자연스러운 문장
- 파일 경로, 개인적 맥락, 열 이름, 회사명 등 구체적 디테일 포함
- 길이, 톤, 형식 다양하게 혼합
- 명확한 정답보다 **경계 케이스(edge case)**에 집중

**Should-trigger 쿼리 (8~10개):**
- 다양한 표현의 같은 의도 (공식적/캐주얼)
- 스킬/파일 유형을 명시적으로 말하지 않지만 분명히 필요한 경우
- 비주류 사용 사례
- 다른 스킬과 경쟁하지만 이 스킬이 이겨야 하는 경우

**Should-NOT-trigger 쿼리 (8~10개):**
- **Near-miss가 핵심** — 키워드가 유사하지만 다른 도구/스킬이 적합한 쿼리
- 명백히 무관한 쿼리("피보나치 함수 작성")는 테스트 가치 없음
- 인접 도메인, 모호한 표현, 키워드 겹침 but 맥락이 다른 경우

### 7-2. 기존 스킬 충돌 검증

새 스킬의 description이 기존 스킬의 트리거 영역과 겹치지 않는지 확인한다:

1. 기존 스킬 목록의 description을 수집
2. 새 스킬의 should-trigger 쿼리가 기존 스킬을 잘못 트리거하지 않는지 확인
3. 충돌 발견 시 description의 경계 조건을 더 명확히 기술

### 7-3. 자동 최적화 (선택적 고급 기능)

description 최적화가 필요한 경우:

1. 20개 eval 쿼리를 Train(60%) / Test(40%) split
2. 현재 description으로 트리거 정확도 측정
3. 실패 케이스를 분석하여 개선된 description 생성
4. Test set 기준으로 best description 선택 (Train set 기준이 아님 — 과적합 방지)
5. 최대 5회 반복

> 이 과정은 `claude -p`를 사용하는 자동화 스크립트로 수행한다. 토큰 비용이 높으므로 스킬이 충분히 안정화된 후 최종 단계에서 실행한다.

---

## 8. 워크스페이스 구조

테스트/평가 결과를 체계적으로 관리하는 디렉토리 구조:

```
{skill-name}-workspace/
├── iteration-1/
│   ├── eval-descriptive-name-1/
│   │   ├── eval_metadata.json
│   │   ├── with_skill/
│   │   │   ├── outputs/
│   │   │   ├── timing.json
│   │   │   └── grading.json
│   │   └── without_skill/
│   │       ├── outputs/
│   │       ├── timing.json
│   │       └── grading.json
│   ├── eval-descriptive-name-2/
│   │   └── ...
│   └── benchmark.json
├── iteration-2/
│   └── ...
└── evals/
    └── evals.json
```

**규칙:**
- eval 디렉토리는 숫자가 아닌 **서술적 이름** 사용 (예: `eval-multi-page-table-extraction`)
- 각 iteration은 독립 디렉토리에 보존 (이전 iteration 덮어쓰기 금지)

---

## 9. 행동 패턴 테스트 (Gatekeeper-Advisor)

Gatekeeper-Advisor처럼 **타이밍·상태 변화에 반응하는 행동 패턴**은 일반 스킬과 다른 테스트 접근이 필요하다. "올바른 파일을 생성했는가"가 아니라 "올바른 시점에 올바른 결정을 했는가"를 검증한다.

### 9-1. 테스트 대상 행동

| 행동 | 검증 질문 |
|------|---------|
| Self-Correction 트리거 | 조건 충족 시 호출됐는가? 조건 미충족 시 억제됐는가? |
| Outer Loop 교착 감지 | 2회 무변화 후 Advisor가 호출됐는가? |
| ACP 품질 | 토큰이 680 이내인가? 핵심 정보가 빠지지 않았는가? |
| 에스컬레이션 | MAX_ADVISOR_CALLS 초과 시 사용자에게 보고됐는가? |
| 비개입 | 정상 진행 중에는 Advisor를 호출하지 않았는가? |

### 9-2. Self-Correction 트리거 타이밍 테스트

**테스트 방식:** 조작된 자가 진단 결과(mock)를 입력으로 주고, 결정 규칙 테이블의 판단이 맞는지 확인한다.

```
# 테스트 케이스 작성 형식

케이스 ID | [P] | [C] | [D] | [B] | [R] | 예상 판단
TC-01    |  25 | 낮음 | 아니오 | 있음 | 아니오 | Advisor 호출 필수 ← C=낮음+D=아니오
TC-02    |  75 | 높음 | 예   | 없음 | 예   | 계속 진행
TC-03    |  50 | 중간 | 예   | 있음 | 예   | Advisor 호출 권장 ← B=있음+C=중간
TC-04    |  25 | 중간 | 아니오 | 없음 | 아니오 | Advisor 호출 필수 ← P≤25+R=아니오

경계 케이스 (near-miss):
TC-05    |  50 | 중간 | 예   | 없음 | 예   | 계속 진행 (C=중간+D=예는 호출 안 함)
TC-06    |  75 | 낮음 | 예   | 없음 | 예   | 계속 진행 (C=낮음이지만 D=예, B=없음, P높음)
```

**[D]=아니오 연속 카운터 테스트 (중요):**
```
# 3회 연속 [D]=아니오 → 필수 호출
이터레이션 1: D=아니오 → count=1, 호출 안 함
이터레이션 2: D=아니오 → count=2, 호출 안 함
이터레이션 3: D=아니오 → count=3, Advisor 호출 필수

# 중간에 D=예가 오면 리셋
이터레이션 1: D=아니오 → count=1
이터레이션 2: D=예    → count=0 (리셋)
이터레이션 3: D=아니오 → count=1, 호출 안 함
```

### 9-3. Outer Loop 교착 감지 테스트

**테스트 방식:** StateSnapshot 시퀀스를 직접 구성하여 `compare_snapshots`의 판단을 검증한다.

```
# 교착 감지 테스트 케이스

케이스 A: 명확한 교착
  snapshot[1]: files=[{path:"01.md", size:0, mtime:"T1"}], tasks=[{status:"in_progress"}]
  snapshot[2]: files=[{path:"01.md", size:0, mtime:"T1"}], tasks=[{status:"in_progress"}]
  예상: is_meaningful=false → stagnation_count=1

  snapshot[3]: snapshot[2]와 동일
  예상: is_meaningful=false → stagnation_count=2 → Advisor 호출

케이스 B: 의미 없는 변화 (교착으로 인정 안 함)
  snapshot[2]: files에 temp.lock 추가됨
  예상: is_meaningful=false (임시 파일 필터)

케이스 C: 정상 진행
  snapshot[2]: files에 새 파일 추가, size > 0
  예상: is_meaningful=true → stagnation_count=0

케이스 D: 재시도 메시지 필터
  snapshot[2]: output_summary="다시 시도합니다" (snapshot[1]과 동일 패턴)
  예상: is_meaningful=false (재시도 메시지 필터)
```

### 9-4. ACP 품질 검증

**측정 지표:**

```
# 각 Advisor 호출에 대해 다음을 측정

1. 토큰 수:
   acp_tokens = len(tokenize(acp))
   assert acp_tokens <= 680, f"ACP 상한 초과: {acp_tokens}"

2. 필수 섹션 존재:
   assert "[TRIGGER]" in acp
   assert "[GOAL]" in acp
   assert "[ATTEMPTS]" in acp
   assert "[BLOCKER]" in acp
   assert "[ASK]" in acp

3. 기각된 가설 제거 확인:
   # [ATTEMPTS]에 있는 접근법이 [HYPOTHESIS]에 재등장하지 않아야 함

4. 압축률 측정 (선택):
   raw_context_tokens = len(tokenize(full_history))
   compression_ratio = acp_tokens / raw_context_tokens
   # 목표: 50% 이상 압축 (compression_ratio < 0.5)
```

**ACP 품질 체크리스트 자동화:**
```python
def validate_acp(acp_text, full_history_text):
    checks = {
        "token_budget": len(tokenize(acp_text)) <= 680,
        "has_trigger": "[TRIGGER]" in acp_text,
        "has_goal": "[GOAL]" in acp_text,
        "has_attempts": "[ATTEMPTS]" in acp_text,
        "has_blocker": "[BLOCKER]" in acp_text,
        "has_ask": "[ASK]" in acp_text,
        "single_ask": acp_text.count("[ASK]") == 1,
        "compression_50pct": len(tokenize(acp_text)) < len(tokenize(full_history_text)) * 0.5
    }
    failed = [k for k, v in checks.items() if not v]
    return len(failed) == 0, failed
```

### 9-5. 비개입 테스트 (과도한 호출 방지)

정상 진행 중인 에이전트에 대해 Advisor가 **호출되지 않아야** 하는 케이스:

```
# 비개입 시나리오

시나리오 1: 모든 이터레이션에서 파일이 꾸준히 증가
  → Advisor 호출 횟수: 0 (stagnation_count이 2에 도달하지 않음)

시나리오 2: Self-Correction 결과 [C]=높음
  → Advisor 호출 없음 (결정 규칙 미충족)

시나리오 3: 1회 지연 후 즉시 회복 (stagnation_count=1로 리셋)
  → Advisor 호출 없음 (임계값 2 미달)
```

**비개입 테스트가 중요한 이유:** Advisor를 너무 자주 호출하면 Opus 비용이 급증한다. "호출 안 해야 할 때 호출 안 하는" 것도 "호출해야 할 때 호출하는" 것만큼 중요하다.

### 9-6. 엔드-투-엔드 행동 검증

단위 테스트 외에 전체 흐름을 검증하는 시나리오 테스트:

```
시나리오: 교착 감지 → ACP 생성 → Advisor 호출 → 해소 → 재모니터링

Steps:
1. 교착 에이전트 시뮬레이션 (2회 이상 동일 상태 스냅샷)
2. Advisor 호출 여부 확인 (횟수, 타이밍)
3. ACP 품질 검증 (9-4 체크리스트)
4. Advisor 응답 후 stagnation_count 리셋 확인
5. 재모니터링 재개 확인
6. 최종 산출물 생성 확인 (_workspace/에 파일 생성)

성공 기준:
[ ] Advisor가 정확히 stagnation_count=2 시점에 1회 호출됨
[ ] ACP 토큰 ≤ 680
[ ] Advisor 호출 후 stagnation_count = 0으로 리셋
[ ] 이후 이터레이션에서 정상 진행 (파일 생성 확인)
[ ] MAX_ADVISOR_CALLS 초과 시 사용자 에스컬레이션 포맷 검증
[ ] `_workspace/`가 보존됨 (사후 교착 패턴 분석용)
```
