# codex

Claude Code 에이전트 하네스 스킬 라이브러리 및 AI 시스템 설계 문서 저장소.

---

## 스킬 라이브러리

### harness — 에이전트 팀 & 스킬 아키텍트

도메인/프로젝트에 맞는 에이전트 팀을 설계하고, 전문 에이전트를 정의하며, 에이전트가 사용할 스킬을 생성하는 메타 스킬.

- **위치:** `skills/harness/`
- **트리거:** "하네스 구성해줘", "에이전트 팀 설계해줘", "하네스 엔지니어링"
- **참조 문서:**
  - [에이전트 설계 패턴](skills/harness/references/agent-design-patterns.md)
  - [오케스트레이터 템플릿](skills/harness/references/orchestrator-template.md)
  - [팀 예시 (6개)](skills/harness/references/team-examples.md)
  - [스킬 작성 가이드](skills/harness/references/skill-writing-guide.md)
  - [스킬 테스트 가이드](skills/harness/references/skill-testing-guide.md)
  - [QA 에이전트 가이드](skills/harness/references/qa-agent-guide.md) — 경계면 검증 + QA 에이전트 Self-Correction 통합 포함

---

### gatekeeper-advisor — 교착 감지 & Advisor 판단 로직

Sonnet(작업 에이전트)이 루프에 빠지거나 진전이 없을 때 Advisor(Opus)를 호출할 시점을 결정하는 Gatekeeper 판단 체계. 두 경로로 작동한다:

| 경로 | 감지 방식 | 트리거 |
|------|----------|--------|
| **Self-Correction** | 에이전트 자가 진단 ([P][C][D][B][R] 5축) | 임계값 충족 시 Advisor 직접 호출 |
| **Outer Loop** | 오케스트레이터 상태 스냅샷 비교 | 연속 2회 무변화 시 Advisor 주입 |

Advisor 호출 시 전체 히스토리 대신 **ACP(Advisor Context Package, ≤680 토큰)**로 압축하여 Opus 비용을 최소화한다.

- **위치:** `skills/gatekeeper-advisor/`
- **트리거:** "에이전트가 루프에 빠졌어", "진전이 없을 때 Advisor 호출 로직", "교착 감지 구현", "stagnation 감지"
- **참조 문서:**
  - [Self-Correction 프롬프트 변형](skills/gatekeeper-advisor/references/self-correction-prompts.md)
  - [Outer Loop 상태 추적](skills/gatekeeper-advisor/references/outer-loop-stagnation.md)
  - [ACP 압축 알고리즘](skills/gatekeeper-advisor/references/context-compression.md)
  - [에이전트 정의 템플릿](skills/gatekeeper-advisor/references/agent-definitions.md)
  - [오케스트레이터 통합 예시](skills/gatekeeper-advisor/references/orchestrator-integration.md)

---

### software-dev-team — 풀스택 개발 팀 (Gatekeeper-Advisor 내장)

백엔드 API · 프론트엔드 UI · QA 검증을 담당하는 3인 에이전트 팀. 개별 에이전트에 Self-Correction 프로토콜이 내장되어 있고, 오케스트레이터의 Gatekeeper 루프가 외부에서 교착을 감지해 Advisor(Opus)를 주입한다.

- **위치:** `skills/software-dev-team/`
- **트리거:** "풀스택 개발 팀", "백엔드/프론트엔드/QA 에이전트", "Gatekeeper-Advisor 실전 예시"
- **에이전트:**
  - [advisor.md](skills/software-dev-team/agents/advisor.md) — ACP 수신 → 교착 원인 진단 → 교정 접근법 발신
  - [backend-dev.md](skills/software-dev-team/agents/backend-dev.md) — REST API 구현 + Self-Correction 내장
  - [frontend-dev.md](skills/software-dev-team/agents/frontend-dev.md) — UI/훅/라우팅 구현 + Self-Correction 내장
  - [qa-inspector.md](skills/software-dev-team/agents/qa-inspector.md) — API↔훅 경계면 검증 + QA Self-Correction
- **참조 문서:**
  - [recovery-playbook.md](skills/software-dev-team/references/recovery-playbook.md) — 교착 패턴별 ACP 예시 + Advisor 교정 성공 사례

---

## 도메인 설계 문서

### AI 기반 인력운영 최적화 시스템

MES 실시간 생산정보, 작업실적, 공정별 작업부하, 작업자 숙련도, 근무 가용정보를 통합 분석하여 생산공정별 최적 인력배치안을 자동 산출하는 AI 기반 시스템 설계.

- [설계서 전문](docs/ai-workforce-optimization-system.md)
- MES/ERP/HR/근태 데이터 통합 실시간 의사결정 구조
- 공정 부하 예측(XGBoost/LGBM) + 작업자 역량 스코어링 + 수리최적화(OR-Tools) 결합
- 생산성 8~15% 향상, 납기 20%+ 개선, 병목 15~30% 감소 목표

---

> 저장소 내비게이션 및 개발 규칙: [CLAUDE.md](CLAUDE.md)

---

## 스킬 구조

```
skills/
├── harness/                          # 에이전트 팀 & 스킬 아키텍트
│   ├── SKILL.md
│   └── references/
│       ├── agent-design-patterns.md  # 6개 아키텍처 패턴 + Gatekeeper-Advisor 메타 패턴
│       ├── orchestrator-template.md  # 오케스트레이터 템플릿 (팀/서브에이전트)
│       ├── team-examples.md          # 실전 예시 6개
│       ├── skill-writing-guide.md    # 스킬 작성 가이드 (행동 패턴 포함)
│       ├── skill-testing-guide.md    # 테스트 방법론 (행동 패턴 테스트 포함)
│       └── qa-agent-guide.md         # QA 에이전트 설계 가이드
│
├── gatekeeper-advisor/               # 교착 감지 & Advisor 판단 로직
│   ├── skill.md
│   └── references/
│       ├── self-correction-prompts.md
│       ├── outer-loop-stagnation.md
│       ├── context-compression.md    # ACP 압축 알고리즘
│       ├── agent-definitions.md      # advisor + gatekeeper 에이전트 정의
│       └── orchestrator-integration.md
│
└── software-dev-team/                # 풀스택 개발 팀 (Gatekeeper-Advisor 내장 실전 예시)
    ├── skill.md                      # 오케스트레이터 + Phase 3 Gatekeeper 모니터링 루프
    ├── agents/
    │   ├── advisor.md                # ACP 수신 → 교착 원인 진단 → 교정 접근법
    │   ├── backend-dev.md            # REST API 구현 + Self-Correction 내장
    │   ├── frontend-dev.md           # UI/훅/라우팅 구현 + Self-Correction 내장
    │   └── qa-inspector.md           # API↔훅 경계면 검증 + QA Self-Correction
    └── references/
        └── recovery-playbook.md      # 교착 패턴 + ACP 예시 + Advisor 교정 성공 사례
```
