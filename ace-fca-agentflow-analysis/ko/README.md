# AI Agent의 Context 관리 전략: ACE-FCA와 AgentFlow에서 배운 것

> 대규모 코드베이스에서 AI Agent를 효과적으로 활용하기 위한 두 가지 접근법 분석

---

## 1. 두 가지 연구 소개

### 1.1 ACE-FCA (Advanced Context Engineering for Coding Agents)

**출처**: HumanLayer (실무 가이드)
**대상**: 대규모 코드베이스(300k+ LOC)에서 AI Coding Agent 활용

**배경 문제**:
- AI Coding Agent가 대규모 기존 코드베이스에서 효과적으로 작동하지 않음
- Single turn으로 복잡한 작업 처리 시 context window가 빠르게 소진
- Research phase 하나만 해도 context의 60-80%를 소모

**핵심 해결책: Frequent Intentional Compaction (FIC)**

프로젝트를 세 단계로 분리하고, 각 단계의 결과를 artifact로 압축하여 전달:

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Research   │───►│     Plan     │───►│  Implement   │
│    Phase     │    │    Phase     │    │    Phase     │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       ▼                   ▼                   ▼
   research.md          plan.md               code
  (~200 lines)        (~200 lines)
```

**각 Phase의 역할**:

| Phase | 목적 | 산출물 |
|-------|------|--------|
| Research | 코드베이스 이해, 관련 파일 파악 | research.md |
| Plan | 구체적 구현 계획 수립 | plan.md |
| Implement | 계획에 따른 코드 작성 | code + tests |

**Sub-agent 활용**:
- 각 phase 내에서 tool calling 시 sub-agent를 spawn
- Sub-agent는 작업 수행 후 결과만 반환하고 context는 폐기
- Parent context의 오염 방지

---

### 1.2 AgentFlow (In-the-Flow Agentic System Optimization)

**출처**: Stanford (학술 연구, ICLR 2026)
**대상**: Tool-augmented LLM의 reasoning 품질 향상

**배경 문제**:
- 기존 방식은 single monolithic policy가 생각과 tool call을 하나의 context에서 interleave
- Long horizon task에서 scaling이 어렵고, 새로운 시나리오에 일반화가 약함

**핵심 해결책: Modular Agentic System**

하나의 task를 네 개의 specialized module로 분리:

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Planner  │───►│ Executor │───►│ Verifier │───►│Generator │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
 다음에 뭘 할지       Tool 실행      결과가 충분한가      최종 답 생성
      │               │               │               │
      └───────────────┴───────────────┴───────────────┘
                            │
                      Shared Memory
                      (상태 저장/공유)
```

**각 Module의 역할**:

| Module | 역할 | 특징 |
|--------|------|------|
| Planner | 다음 action 결정 (tool 선택 + context) | 학습 대상 (Flow-GRPO) |
| Executor | Tool 실행 | Frozen |
| Verifier | 결과 검증, STOP/CONTINUE 판단 | Frozen |
| Generator | 최종 답변 생성 | Frozen |

**Memory 구조** (실제 코드 기반):
```python
class Memory:
    def __init__(self):
        self.query = None           # 원래 질문
        self.files = []             # 관련 파일
        self.actions = {}           # Turn별 action 기록

    def add_action(self, step_count, tool_name, sub_goal, command, result):
        self.actions[f"Action Step {step_count}"] = {
            'tool_name': tool_name,
            'sub_goal': sub_goal,
            'command': command,
            'result': result,
        }
```

**실행 제한**:
- `max_steps = 10` (기본값)
- `max_time = 300초` (5분)

**핵심 결과**:
- 7B 모델 + AgentFlow가 GPT-4o를 일부 벤치마크에서 능가
- Planner만 학습해도 전체 성능 향상

---

## 2. 공통점

### 2.1 근본 원리

> **"복잡한 것을 한번에 처리하지 말고, specialized 단계로 분해하라"**

두 접근법 모두 **monolithic 처리의 한계**를 인식하고, **단계별 분해**로 해결한다.

### 2.2 구체적 공통점

| 관점 | ACE-FCA | AgentFlow |
|------|---------|-----------|
| **분해** | 프로젝트 → Phase | Task → Module |
| **전문화** | 각 Phase가 고유 역할 | 각 Module이 고유 역할 |
| **정보 흐름 제어** | Artifact로 압축 전달 | Memory에 결과만 저장 |
| **Context 오염 방지** | Phase 간 context 초기화 | Sub-agent 결과만 반환 |

### 2.3 핵심 통찰

두 접근법의 가치는 단순한 "분해"가 아니라 **정보 흐름의 bottleneck 생성**에 있다:

```
[전체 과정의 raw context] → [Bottleneck] → [압축된 결과]

ACE-FCA:  Research 전체 과정 → Bottleneck → research.md
AgentFlow: Tool 실행 과정 → Memory → 결과만 저장
```

이 bottleneck이 noise는 버리고 signal만 전달하게 만든다.

---

## 3. 차이점

### 3.1 적용 레벨

| | ACE-FCA | AgentFlow |
|---|---------|-----------|
| **레벨** | Macro (프로젝트) | Micro (Task) |
| **단위** | 수십~수백 tool calls | 최대 10 steps |
| **시간** | 수 시간 ~ 수 일 | 수 분 |

### 3.2 핵심 관심사

```
ACE-FCA:  "Context를 어떻게 관리할 것인가?"
          └─► Context 폭발 방지가 목적

AgentFlow: "Task를 어떻게 잘 수행할 것인가?"
          └─► 실행 품질 향상이 목적
```

### 3.3 정보 전달 방식

| | ACE-FCA | AgentFlow |
|---|---------|-----------|
| **매체** | 파일 (.md artifact) | 객체 (Memory) |
| **영속성** | 영구 (파일로 저장) | 휘발 (세션 종료 시 삭제) |
| **가독성** | 사람이 읽고 검토 가능 | 내부 상태 |

### 3.4 사람 개입

| | ACE-FCA | AgentFlow |
|---|---------|-----------|
| **검토 지점** | Phase 완료 시 사람이 검토 | 없음 (자동) |
| **검증 주체** | 사람 | Verifier 모듈 |
| **철학** | Human-in-the-loop | Fully automated |

### 3.5 Horizon 비교

```
ACE-FCA Phase:
├── Task 1 (수십 tool calls)
├── Task 2 (수십 tool calls)
├── Task 3 (수십 tool calls)
└── ... (수백 tool calls 가능)

AgentFlow:
├── Step 1
├── Step 2
├── ...
└── Step 10 (max)
```

**→ Scale이 다름**

---

## 4. 통합 방안 (내 생각)

두 연구를 보고 내가 정리해본 통합 방안이다.

### 4.1 통합 구조

```
┌─────────────────────────────────────────────────────────────────┐
│  ACE-FCA (Macro Level) - 프로젝트 흐름 관리                          │
│                                                                 │
│  ┌───────────┐      ┌───────────┐      ┌───────────┐            │
│  │  Research │ ──►  │   Plan    │ ──►  │ Implement │            │
│  └─────┬─────┘      └─────┬─────┘      └─────┬─────┘            │
│        │                  │                  │                  │
│        ▼                  ▼                  ▼                  │
│   research.md         plan.md             code                  │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  각 Phase 내부                                                    │
│                                                                 │
│  Task 1 ─► Sub-agent ─► 결과만 반환 (context 버림)                  │
│  Task 2 ─► Sub-agent ─► 결과만 반환 (context 버림)                  │
│  Task 3 ─► Sub-agent ─► 결과만 반환 (context 버림)                  │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Sub-agent 내부 (Micro Level) - 조건에 따라 선택                     │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Option A: 강한 모델 + 쉬운 문제                              │    │
│  │                                                         │    │
│  │   Claude/GPT-4 ──► 바로 처리 ──► 결과 반환                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Option B: 약한 모델 or 어려운 문제                           │    │
│  │                                                         │    │
│  │   AgentFlow 구조:                                        │    │
│  │   Planner → Executor → Verifier → Generator             │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 각 레벨의 역할

| Level | 담당 | 적용 |
|-------|------|------|
| **Macro (ACE-FCA)** | Context 관리 | 적용 권장 |
| **Micro (AgentFlow)** | Task 수행 품질 | 조건부 적용 |

### 4.3 Micro Level 적용 기준

| 조건 | Sub-agent 방식 | 이유 |
|------|----------------|------|
| 강한 모델 + 쉬운 문제 | 직접 호출 | 오버헤드 회피 |
| 강한 모델 + 어려운 문제 | AgentFlow | 품질 향상 |
| 약한 모델 + 쉬운 문제 | AgentFlow | 품질 보완 |
| 약한 모델 + 어려운 문제 | AgentFlow (권장) | 품질 확보 |

**핵심 insight**:
> AgentFlow는 "약한 모델을 강하게 만들거나, 어려운 문제를 풀 수 있게 만드는 구조화 방법"

### 4.4 구체적 예시: Research Phase

```
Research Phase
│
├── Task 1: "src/ 폴더 구조 파악"
│   └── Sub-agent (AgentFlow)
│       ├── Planner: "먼저 디렉토리 구조 확인"
│       ├── Executor: tree 명령 실행
│       ├── Verifier: "구조 파악됨, STOP"
│       └── Generator: 구조 요약 반환
│       └── (Memory 폐기)
│
├── Task 2: "에러 핸들링 패턴 파악"
│   └── Sub-agent (AgentFlow)
│       ├── Planner: "error 관련 파일 검색"
│       ├── Executor: grep 실행
│       ├── Verifier: "더 필요, CONTINUE"
│       ├── Planner: "핵심 파일 내용 확인"
│       ├── Executor: 파일 읽기
│       ├── Verifier: "충분함, STOP"
│       └── Generator: 패턴 요약 반환
│       └── (Memory 폐기)
│
├── Task 3: "테스트 구조 파악"
│   └── ...
│
└── 결과 종합 → research.md 생성
```

### 4.5 왜 Sub-task에 AgentFlow 방식이 권장되는가

**"단순 작업"이라고 생각되는 것**:
```
"이 파일 읽어줘" → tool call → 파일 내용 반환
```

**실제로 일어나는 일**:
```
파일 내용 (500줄) 전체가 parent context에 들어감
→ context 오염
→ ACE-FCA가 막으려는 바로 그 문제
```

**실제로 해야 하는 것**:
```
"이 파일에서 에러 핸들링 관련 부분만 찾아서 요약해줘"
→ sub-agent가 읽고 → 분석하고 → 요약해서 반환
→ 이미 multi-step 작업
```

**→ 대부분의 의미 있는 sub-task는 "실행 → 분석 → 요약" 패턴이 필요하며, 이는 AgentFlow 구조에 자연스럽게 맵핑됨**

---

## 5. 트레이드오프 및 고려사항 (내 생각)

통합 방안을 적용할 때 예상되는 트레이드오프를 정리해봤다.

### 5.1 주요 트레이드오프: 오버헤드

```
매 sub-task마다 AgentFlow 구동 시:
- LLM 호출 횟수 증가: Planner + Executor + Verifier + Generator
- 지연시간 증가: 순차 실행
- 비용 증가: API 호출 비용
```

**완화 방안**:
- 정말 복잡한 task에만 full AgentFlow 적용
- 단순 task는 경량화된 버전 (Executor → Summarizer만)
- 독립적인 task는 병렬 처리

### 5.2 적용 범위

이 통합 프레임워크는 **대규모 Brownfield 코드베이스**를 대상으로 한다:
- 300k+ LOC 규모
- 기존 코드 이해 필요
- 복잡한 수정/기능 추가

**적합하지 않은 경우**:
- 작은 프로젝트
- Greenfield 개발
- 단순 버그 수정

### 5.3 LLM 성능과의 관계

| LLM 성능 | Macro (ACE-FCA) | Micro (AgentFlow) |
|----------|-----------------|-------------------|
| 강함 | 여전히 필요 (context 관리) | 선택적 |
| 약함 | 권장 | 권장 |

**핵심**:
- **Context 관리 (Macro)**: 모델 성능과 무관하게 필요
- **Task 품질 (Micro)**: 모델 성능에 따라 필요도 변화

---

## 6. 결론

### 6.1 핵심 요약

```
Macro (ACE-FCA)
  → Context 관리 문제 해결
  → 모델 성능과 무관하게 대규모 프로젝트에서 적용 권장

Micro (AgentFlow)
  → Task 수행 품질 문제 해결
  → 모델 성능 / 문제 난이도에 따라 선택적 적용
```

### 6.2 통합 프레임워크 한 줄 정리

> "ACE-FCA로 context를 관리하고, 필요시 AgentFlow로 task 수행 품질을 높인다"

### 6.3 실무 적용 가이드

1. **대규모 프로젝트 착수 시**: ACE-FCA의 Research → Plan → Implement 흐름 적용
2. **각 Phase 내 Sub-task 실행 시**: Sub-agent spawn하여 결과만 반환
3. **Sub-agent 구현 선택**:
   - 강한 모델 + 단순 작업: 직접 호출
   - 그 외: AgentFlow 구조 적용
4. **오버헤드 관리**: 병렬화, 경량화 버전 활용

---

## 참고 자료

- [ACE-FCA: Advanced Context Engineering for Coding Agents](https://github.com/humanlayer/advanced-context-engineering-for-coding-agents)
- [Youtube: No Vibes Allowed: Solving Hard Problems in Complex Codebases – Dex Horthy, HumanLayer](https://www.youtube.com/watch?v=rmvDxxNubIg)
- [Youtube: Advanced Context Engineering for Agents](https://www.youtube.com/watch?v=IS_y40zY-hc)
- [AgentFlow: In-the-Flow Agentic System Optimization](https://arxiv.org/abs/2510.05592)
- [AgentFlow GitHub Repository](https://github.com/lupantech/AgentFlow)
- [AgentFlow Project Page](https://agentflow.stanford.edu/)
