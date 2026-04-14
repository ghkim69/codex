# CLAUDE.md — codex 저장소 가이드

Claude Code 에이전트 하네스를 위한 **스킬 라이브러리 및 설계 문서 저장소**.

---

## 저장소 구조

```
codex/
├── skills/
│   ├── harness/                   # 에이전트 팀 & 스킬 아키텍트 메타 스킬
│   │   ├── SKILL.md               # 핵심 스킬 정의 (트리거, 워크플로우, 체크리스트)
│   │   └── references/            # 상세 참조 문서 (필요할 때만 로드)
│   │       ├── agent-design-patterns.md
│   │       ├── orchestrator-template.md
│   │       ├── team-examples.md
│   │       ├── skill-writing-guide.md
│   │       ├── skill-testing-guide.md
│   │       └── qa-agent-guide.md
│   │
│   └── gatekeeper-advisor/        # 교착 감지 & Advisor 판단 로직 스킬
│       ├── skill.md               # 핵심 스킬 정의 (두 트리거 경로 + ACP 포맷)
│       └── references/
│           ├── self-correction-prompts.md   # 도메인별 자가 진단 변형 + ACP 예시
│           ├── outer-loop-stagnation.md     # Outer Loop 알고리즘 상세
│           ├── context-compression.md       # ACP 토큰 버짓 + 압축 규칙
│           ├── agent-definitions.md         # advisor.md + gatekeeper.md 정의 전문
│           └── orchestrator-integration.md  # 팀/서브에이전트 통합 전체 예시
│
├── docs/
│   └── ai-workforce-optimization-system.md  # AI 기반 인력운영 최적화 시스템 설계서
│
├── README.md                      # 스킬 라이브러리 전체 참조 문서
└── CLAUDE.md                      # 이 파일
```

---

## 주요 스킬 요약

### harness (`skills/harness/SKILL.md`)

**역할:** 도메인/프로젝트에 맞는 에이전트 팀을 설계하고, 전문 에이전트와 스킬을 생성하는 메타 스킬.

**트리거:** "하네스 구성해줘", "에이전트 팀 설계해줘", "하네스 엔지니어링"

**핵심 원칙:**
- 모든 에이전트는 `프로젝트/.claude/agents/{name}.md`에 정의
- 모든 에이전트는 `model: "opus"` 사용
- 에이전트 팀 모드가 기본 (서브 에이전트는 예외적으로 선택)
- skill.md 본문은 500줄 이내 (초과 시 references/로 분리)

**참조 우선순위:**
1. `team-examples.md` — 실전 예시 6개 (새 하네스 설계 시 참조)
2. `agent-design-patterns.md` — 아키텍처 패턴 6종 + Gatekeeper-Advisor 메타 패턴
3. `orchestrator-template.md` — 팀/서브에이전트 오케스트레이터 템플릿
4. `skill-writing-guide.md` — 스킬 작성 가이드 (행동 패턴 포함)
5. `skill-testing-guide.md` — 테스트 방법론
6. `qa-agent-guide.md` — QA 에이전트 설계 (경계면 검증 + Gatekeeper 통합)

---

### gatekeeper-advisor (`skills/gatekeeper-advisor/skill.md`)

**역할:** 에이전트가 루프에 빠지거나 진전이 없을 때 Advisor(Opus)를 호출할 시점을 결정하는 판단 체계.

**트리거:** "에이전트가 루프에 빠졌어", "진전이 없을 때 Advisor 호출 로직", "교착 감지 구현"

**두 경로:**

| 경로 | 주체 | 작동 방식 |
|------|------|----------|
| **Self-Correction** | 작업 에이전트 | [P][C][D][B][R] 자가 진단 → 임계값 충족 시 Advisor 직접 호출 |
| **Outer Loop** | 오케스트레이터 | StateSnapshot 비교 → 연속 2회 무변화 시 Advisor 주입 |

**핵심 상수:**
- `STAGNATION_THRESHOLD = 2` — 연속 N회 무변화 시 교착 판단
- `MAX_ADVISOR_CALLS = 3` — 초과 시 사용자 에스컬레이션
- ACP 총 토큰 상한: **680 토큰**

**참조 우선순위:**
1. `agent-definitions.md` — advisor.md + gatekeeper.md 복사/붙여넣기 템플릿 (프로젝트 적용 시)
2. `orchestrator-integration.md` — 팀/서브에이전트 통합 코드 전문
3. `context-compression.md` — ACP 구성 절차 + 토큰 버짓 규칙
4. `self-correction-prompts.md` — 도메인별 자가 진단 변형 + ACP 압축 전/후 예시
5. `outer-loop-stagnation.md` — 스냅샷 알고리즘 + 다중 에이전트 교착 + 에스컬레이션

---

## 개발 규칙

### 브랜치 전략

- 피처 브랜치: `claude/{기능명}-{ID}` (예: `claude/gatekeeper-advisor-logic-NBzlu`)
- 기본 브랜치: `main`

### 커밋 메시지

```
{type}: {한국어 요약}

- {변경사항 1}
- {변경사항 2}

https://claude.ai/code/session_{세션ID}
```

타입: `feat`(신규), `fix`(수정), `docs`(문서), `refactor`(리팩터)

### 스킬 수정 시 체크리스트

- [ ] skill.md 본문이 500줄 이내인가?
- [ ] references/ 파일에 300줄 이상이면 목차(ToC)가 있는가?
- [ ] 새 참조 파일은 skill.md에 포인터가 추가됐는가?
- [ ] README.md의 해당 스킬 섹션에 새 파일이 링크됐는가?
- [ ] 트리거 검증 (should-trigger / should-NOT-trigger)을 업데이트했는가?

### 새 스킬 추가 시

1. `skills/{skill-name}/skill.md` 생성 (frontmatter: name, description 필수)
2. `skills/{skill-name}/references/` 생성 (필요한 경우)
3. `.claude-plugin/plugin.json`의 version 업데이트 (SemVer)
4. `README.md`에 스킬 섹션 추가

---

## ACP 빠른 참조

Advisor(Opus)를 호출할 때 전달하는 압축 컨텍스트 포맷:

```
[TRIGGER] {결정 규칙 코드 — C=낮음+D=아니오, D×3, OL:stagnation×2 등}
[GOAL]    {달성 목표 2문장 이내}
[HARD_CONSTRAINTS] {변경 불가 기술 제약}
[ATTEMPTS]
  A1: [{유형}] {시도 한 줄} → {결과 5단어}
  ×N: [반복] {N회 동일 결과 콜랩스}
[BLOCKER]
  type: {에러 유형}
  loc:  {파일:라인}
  msg:  {에러 메시지 첫 줄}
[HYPOTHESIS]
  H1 ({신뢰도}%): {가설}
[ASK] {단일 질문}
```

목표: 전체 ACP ≤ 360 토큰 / 상한: 680 토큰.

상세 규칙: `skills/gatekeeper-advisor/references/context-compression.md`
