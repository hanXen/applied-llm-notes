# LLM 기반 모의침투 자동화: 핵심 기법 심층 분석

> Strix와 CAI의 소스 코드 분석을 통해 도출한, 순수 LLM 대비 모의침투 효과를 높이는 설계 기법과 그 근거
>
> **레포지토리 (코드 레퍼런스 기준 커밋):**
> - Strix: https://github.com/usestrix/strix @ [`5a76fab`](https://github.com/usestrix/strix/tree/5a76fab4aeca9e7b564cfa1f13f45c3f89d4f66e)
> - CAI: https://github.com/aliasrobotics/cai @ [`e22a122`](https://github.com/aliasrobotics/cai/tree/e22a1220f764e2d7cf9da6d6144926f53ca01cde)
>
> **상세 분석**: [Strix 심층 분석](strix-deep-analysis.md) | [CAI 심층 분석](cai-deep-analysis.md)

---

## 서론: 순수 LLM의 한계

ChatGPT나 Claude에 직접 "이 사이트를 침투 테스트해줘"라고 하면:

1. **도구가 없다** — nmap, sqlmap, 브라우저를 직접 실행 못함
2. **Context가 유한하다** — 긴 테스트 과정에서 앞의 발견 사항을 잊음
3. **구조화된 Workflow가 없다** — 체계적으로 모든 취약점 유형을 순회하지 못함
4. **전문성이 분산된다** — IDOR 전문가와 SQLi 전문가를 동시에 되기 어려움
5. **제어 장치가 없다** — 환각으로 false positive 보고, 비용 폭주

이 문서는 Strix와 CAI가 각각 **어떤 기법으로** 이 한계를 극복하는지, **왜 그 기법이 효과적인지**를 코드 근거와 함께 분석한다.

---

## 1. Prompt 엔지니어링 — LLM을 Pentester로 만드는 법

> 해결하는 문제: 순수 LLM은 "해킹해줘"에 일반적 조언만 한다

### 1.1 Strix: 공격적 자율 지시 + Skill 주입

**시스템 Prompt** ([`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja), 407줄):

Jinja2 Template 기반으로 동적 생성되며, 핵심 지시는:

- **공격적 톤**: "GO SUPER HARD on all targets", "2000+ steps MINIMUM", "UNLEASH FULL CAPABILITY"
- **완전 자율**: "NEVER ask for user input or confirmation - always proceed autonomously"
- **Bug Bounty Mindset**: "One critical vulnerability > 100 informational findings. If it wouldn't earn $500+ on bug bounty platform, keep searching"
- **10가지 필수 취약점 Checklist**: IDOR, SQLi, SSRF, XSS, XXE, RCE, CSRF, Race Condition, Business Logic, Auth/JWT

**Phase 기반 Workflow 강제**:
- Black-box Phase 1: 전체 정찰
- White-box Phase 1: 코드 매핑
- Phase 2: 취약점 유형 × 컴포넌트별 Sub-Agent 생성

**Skill 시스템** ([`skills/`](https://github.com/usestrix/strix/tree/5a76fab/strix/skills), 26개 Skill 파일):
- YAML frontmatter가 있는 마크다운 파일로 구성
- Agent당 최대 5개 Skill이 시스템 Prompt에 동적 주입 — `{{ get_skill(skill_name) }}`
- 스캔 모드별 자동 추가 — [`quick.md`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/scan_modes/quick.md) / [`standard.md`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/scan_modes/standard.md) / [`deep.md`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/scan_modes/deep.md)
- 예시 — IDOR Skill ([`idor.md`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/vulnerabilities/idor.md), 213줄): 공격 표면 식별, 고가치 타겟, 정찰법, 우회 기법, 체이닝 전략을 모두 담음

**Prompt 구성 흐름**:
```
시스템 Prompt (Jinja2, 407줄)
  ├── 코어 Instruction (공격적 자율 행동 지시)
  ├── 스캔 모드별 가이드 (quick/standard/deep Skill)
  ├── 취약점 Skill들 (최대 5개 동적 주입)
  ├── 도구 Schema (XML, Registry에서 자동 생성)
  ├── 환경 정보 (Kali Linux Container 도구 목록)
  └── Agent Identity Metadata (이름, ID)
```

### 1.2 CAI: 체계적 방법론 + 메모리 주입

**마스터 Template** ([`system_master_template.md`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/core/system_master_template.md), 190줄):

Mako Template 기반으로 동적 생성되며, 핵심 지시는:

- **방법론 중심**: 체계적 단계별 접근 (정찰 → 위협 모델링 → 테스트 → 익스플로잇 → Report)
- **안전 우선**: "Prefer safe, read-only tests first", "Do not attempt data deletion or service disruption"
- **환경 Context 자동 주입** — [L128-157](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/core/system_master_template.md#L128-L157): OS 정보, IP, hostname, 워드리스트 경로 등 **런타임에 실시간 수집**
- **메모리 주입** — [L98-105](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/core/system_master_template.md#L98-L105): 이전 Session의 Episodic/Semantic 기억을 Prompt에 통합

**Agent별 전문 Prompt**:
- [`system_red_team_agent.md`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/system_red_team_agent.md) — "gain root access and find flags", 비인터랙티브 명령 강제 ([L22-31](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/system_red_team_agent.md#L22-L31)), 셸 Session 관리 ([L38-60](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/system_red_team_agent.md#L38-L60))
- [`system_web_pentester.md`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/system_web_pentester.md) — 7단계 방법론 ([L48-154](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/system_web_pentester.md#L48-L154)), "10-15 custom test cases" (Business-Logic Abuse Backlog, [L95](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/system_web_pentester.md#L95))

**Prompt 구성 흐름**:
```
마스터 Template (Mako, 190줄)
  ├── Agent Instruction (Agent별 .md 파일)
  ├── 압축된 이전 대화 Context
  ├── 메모리 주입 (Episodic/Semantic from Qdrant)
  ├── 추론 블록 (reasoning model 출력)
  └── 환경 Context (OS, IP, 워드리스트 등 런타임 수집)
```

### 1.3 Insight: 왜 이 기법들이 효과적인가

| 관점 | Strix | CAI |
|------|-------|-----|
| **톤** | 공격적/무제한 ("GO SUPER HARD") | 체계적/안전 우선 ("safe, read-only first") |
| **자율성** | 완전 자율 ("절대 사용자에게 묻지 마라") | 반자율 (HITL, Ctrl+C 중단) |
| **지식 주입** | Skill 파일 (마크다운, Agent당 5개) | 메모리 시스템 (Vector DB, 이전 Session 학습) |
| **도구 Schema** | XML 기반 Custom 포맷 | OpenAI function calling JSON Schema |
| **환경 정보** | Container 내 고정 (Kali 도구 목록) | 런타임 동적 수집 (OS, IP, 워드리스트 탐색) |
| **방법론** | 취약점 유형 기반 (10가지 필수 체크) | Kill Chain Phase 기반 (정찰 → 익스플로잇 → C2) |
| **Workflow 강제** | Prompt에서 Phase 강제 | Agent 선택으로 Workflow 결정 |
| **스캔 모드** | 3단계 (quick/standard/deep) → reasoning effort 연동 | 환경변수 기반 (MAX_TURNS, PRICE_LIMIT) |

**핵심 Insight**:

1. **공격성 vs 방법론은 커버리지 도메인에 의해 결정된다.**
- Strix의 "GO SUPER HARD"는 웹 앱 CI/CD 자동화에서 비롯된다 — 사람이 없는 Pipeline에서 Agent가 일찍 포기하면 안 되므로, 과도할 정도로 지속성을 강제한다.
- CAI의 "safe, read-only first"는 Kill Chain 전체(정찰→C2)를 다루기 때문이다 — 다단계 작전에서 순서를 무시한 공격적 탐색은 오히려 비효율적이다.  
- **Trade-off: 공격적 톤은 웹 앱의 숨은 Edge Case를 발굴하는 데 유리하지만, 체계적 방법론은 Kill Chain처럼 순서가 중요한 작전에서 더 안정적이다.**

2. **정적 Skill vs 동적 메모리는 대상의 예측 가능성에 따른 선택이다.**
- Strix가 마크다운 Skill 파일을 쓰는 이유: 웹 취약점은 OWASP Top 10으로 잘 분류되어 있어, IDOR 213줄짜리 정형화된 Playbook이 경험적 학습보다 신뢰할 수 있다.
- CAI가 Vector DB 메모리를 쓰는 이유: 네트워크/IoT/포렌식까지 다루는 범위에서 정적 Playbook으로 모든 시나리오를 커버할 수 없으므로, 과거 경험에서 유사 사례를 검색하는 방식이 더 확장 가능하다.  
- **Trade-off: 정적 Skill은 처음 보는 타겟에도 일관된 품질을 보장하지만 업데이트가 수동이고, 동적 메모리는 반복 사용으로 개선되지만 첫 실행 품질이 낮다.**

---

## 2. 멀티Agent 아키텍처 — 전문가 팀 구성

> 해결하는 문제: 하나의 LLM에 모든 것을 맡기면 전문성이 분산된다

### 2.1 Strix: 동적 Agent 생성 (트리 구조)

**Agent 시스템** ([`agents/StrixAgent/`](https://github.com/usestrix/strix/tree/5a76fab/strix/agents/StrixAgent)):
- **Root Agent (StrixAgent)**: 전체 침투 테스트를 조율하는 메인 Coordinator
- **Sub-agents**: 특정 태스크(IDOR, SQLi, XSS 등)에 **런타임에 동적 생성**되는 전문 Agent — [`agents_graph_actions.py:188-281`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py#L188-L281)
- **Agent Graph**: 방향 그래프(`_agent_graph`)로 Agent 관계 추적 (노드: ID/이름/태스크/상태, 엣지: delegation)
- **상태 관리**: 각 Agent가 독립적 `AgentState` 유지 — [`state.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/state.py)

**Agent Loop** ([`base_agent.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/base_agent.py)):
```
while iteration < 300:
  1. 상태 확인 (stop/waiting/failed 체크)
  2. iteration++
  3. 85% 도달 시 경고 주입, 97% 시 마무리 강제
  4. LLM.generate() → Streaming 응답
  5. 첫 </function> 태그에서 Streaming 중단 ← Token 절약 핵심 최적화
  6. 도구 호출 파싱 (XML)
  7. Sandbox/로컬 분기 실행
  8. finish 시그널 확인 → Loop 종료
```
- 반복당 정확히 **1개 도구 호출** (순차)
- `max_iterations = 300` — [`base_agent.py:50`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/base_agent.py#L50)
- 85%/97% 2단계 경고 — [`base_agent.py:183-197`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/base_agent.py#L183-L197)
- **Streaming 조기 중단**: 첫 `</function>` 태그에서 중단하여 불필요한 출력 Token 절약 — [`llm.py:146-152`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py#L146-L152)

**도구 격리**: `contextvars` 기반으로 Agent ID별 Browser/Terminal/Python 인스턴스 격리 — [`tools/__init__.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/__init__.py)

### 2.2 CAI: 사전 정의 Agent 풀 + Pattern 조합

**Agent 시스템** (~20개 Agent) — [`src/cai/agents/`](https://github.com/aliasrobotics/cai/tree/e22a122/src/cai/agents), [`factory.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/factory.py):

| 카테고리 | Agent | 용도 |
|----------|---------|------|
| **공격** | [`red_teamer.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/red_teamer.py), [`web_pentester.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/web_pentester.py), [`bug_bounter.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/bug_bounter.py) | 침투 테스트 |
| **방어** | [`blue_teamer.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/blue_teamer.py), [`wifi_security_tester.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/wifi_security_tester.py) | 방어 및 하드닝 |
| **조사** | [`dfir.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/dfir.py), [`memory_analysis_agent.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/memory_analysis_agent.py), [`reverse_engineering_agent.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/reverse_engineering_agent.py) | 포렌식/분석 |
| **모바일/IoT** | [`android_sast_agent.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/android_sast_agent.py), [`subghz_sdr_agent.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/subghz_sdr_agent.py) | 모바일/IoT 보안 |
| **전문** | [`codeagent.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/codeagent.py), [`thought.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/thought.py), [`flag_discriminator.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/flag_discriminator.py), [`reporter.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/reporter.py) | 유틸리티/조율 |
| **게임이론** | redteam_gctr, blueteam_gctr, purple_team_agents | CPT 분석 |

**5가지 Orchestration Pattern** (PatternType Enum) — [`patterns/`](https://github.com/aliasrobotics/cai/tree/e22a122/src/cai/agents/patterns), [`pattern.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/patterns/pattern.py):
- **Swarm**: 분산형 P2P + 동적 Handoff — [`red_team.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/patterns/red_team.py)
- **Hierarchical**: 탑다운 태스크 할당 (PlannerAgent → 전문가)
- **Sequential**: 순차 Pipeline (A → B → C)
- **Conditional**: 조건부 분기
- **Parallel**: 병렬 실행 (`CAI_PARALLEL=N`) — [`parallel_offensive_patterns.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/patterns/parallel_offensive_patterns.py)

**Agent Loop** ([`run.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/sdk/agents/run.py)):
```
while current_turn < max_turns:
  1. 시스템 Prompt 동적 렌더링 (메모리/환경 주입)
  2. 입력 Guardrail 실행 (첫 턴만, 비동기 병렬)
  3. LLM 호출 (도구 + Handoff 포함)
  4. 응답 처리 → 도구 호출 + Handoff + 컴퓨터 액션 분류
  5. 도구들 asyncio.gather()로 병렬 실행 ← 핵심 차이
  6. 출력 Guardrail 실행
  7. 다음 스텝 결정: FinalOutput / Handoff / RunAgain
```
- 턴당 **다수 도구 병렬 실행**
- Guardrail Pipeline 내장
- Handoff 라우팅 통합

### 2.3 동시성 모델

**Strix** — 넓은 병렬:

Sub-Agent가 각각 독립 Thread(daemon)로 실행된다. 10-20개 Sub-Agent가 각각 407줄 시스템 Prompt + 5개 Skill을 독립 로드하므로 Prompt 중복 비용이 발생한다. `cache_control` 기반 Prompt Caching으로 반복 비용을 완화한다 — [`llm.py:309-325`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py#L309-L325).

**CAI** — 깊은 순차:

하나의 활성 Agent가 순차 전환되며, 전체 History가 Agent 간 누적된다. `/compact` 압축으로 History 증가 비용을 완화한다. 단, `CAI_PARALLEL=N` 환경변수 설정으로 병렬 실행도 가능하다 — [`parallel_offensive_patterns.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/patterns/parallel_offensive_patterns.py).

| 관점 | Strix | CAI |
|------|-------|-----|
| **모델** | 넓은 병렬 (Multi-Thread) | 깊은 순차 (순차 전환) |
| **활성 Agent** | 동시에 10-20개 | 한 번에 1개 (기본) |
| **비용 Pattern** | Prompt 중복 (시스템 Prompt ×N) | History 누적 (점진 증가) |
| **비용 완화** | Prompt Caching (`cache_control`) | `/compact` 압축 |
| **병렬 옵션** | 기본 (Sub-Agent = Thread) | `CAI_PARALLEL=N`으로 활성화 |

### 2.4 Insight: 왜 멀티Agent가 효과적인가

| 관점 | Strix | CAI |
|------|-------|-----|
| **Agent 생성** | 동적 (런타임에 필요 시) | 정적 (~20개 사전 정의) |
| **구조** | 루트 + N개 Sub-Agent (트리) | Agent 풀 + 5가지 Pattern |
| **동시성** | Multi-Thread 병렬 | 순차 전환 (한 번에 하나만 활성) |
| **Agent Loop** | 반복당 1도구, 300회 한도, Streaming 조기 중단 | 턴당 N도구 병렬, Guardrail Pipeline |

**핵심 Insight**:

1. **역할 분할이 성능을 높인다.**
- IDOR을 찾는 Agent와 SQLi를 찾는 Agent는 서로 다른 전문 지식(Skill 파일)을 로드한다. 순수 LLM 한 Session에 모든 지식을 넣으면 Context가 희석된다.

2. **동적 생성 vs 사전 정의는 대상의 예측 가능성에 의한 선택이다.**
- Strix가 런타임에 Agent를 생성하는 이유: 웹 앱의 구조(엔드포인트, 인증 방식, 기술 스택)는 정찰 후에야 알 수 있으므로, 필요한 전문가를 미리 정의할 수 없다.
- CAI가 ~20개를 미리 정의하는 이유: Kill Chain Phase(정찰→익스플로잇→권한 상승→C2)는 타겟과 무관하게 일정하므로, 역할 분류가 사전에 가능하다.
- **Trade-off: 동적 생성은 타겟 맞춤형이지만 Agent Spawn 비용이 발생하고, 사전 정의는 즉시 투입 가능하지만 불필요한 Agent가 포함될 수 있다.**

3. **"반복당 1도구" vs "턴당 N도구"는 추론 Pattern의 차이다.**
- Strix의 1도구 방식은 매 결과를 분석한 후 다음 행동을 결정하는 deliberate reasoning을 강제한다 — 웹 앱에서 한 응답이 다음 프로브를 결정하므로 적합하다.
- CAI의 N도구 병렬은 독립적인 정찰 명령(포트 스캔, 서비스 열거 등)을 Batch 처리한다 — 네트워크 정찰에서 프로브 간 의존성이 낮으므로 적합하다.
- **Trade-off: 1도구 방식은 적응적이지만 느리고, N도구 병렬은 빠르지만 중간 결과에 의한 방향 전환이 어렵다.**

---

## 3. 도구 통합 — LLM에게 손과 발을 주는 법

> 해결하는 문제: 순수 LLM은 텍스트만 생성하고 실제 도구를 실행 못한다

### 3.1 Strix 툴킷 (13+ 도구)

| 도구 | 설명 | 핵심 기여 |
|------|------|-----------|
| **browser** | Playwright 기반 멀티탭 브라우저 자동화 (goto, click, type, scroll, JS 실행) | JS 렌더링 SPA 테스트 가능 |
| **proxy** | HTTP Proxy - 요청/응답 조작, HTTPQL 필터링, 스코프 규칙 | 트래픽 분석/변조 |
| **terminal** | tmux 기반 셸 - PS1 Pattern(`[STRIX_$?]$`)으로 exit code 추출, 멀티 Session | 임의 명령 실행 |
| **python** | IPython 셸 - Proxy 함수 자동 주입, stdout/stderr 캡처 | Custom 스크립트 |
| **agents_graph** | Sub-Agent 생성, 메시징, 상태 모니터링 | 멀티Agent 조율 |
| **file_edit** | 코드/설정 파일 편집 (openhands_aci 통합) | 자동 패치 |
| **reporting** | CVSS v3.1 계산, PoC 코드 포함, 개선 단계 | 구조화된 Report |
| **thinking** | Claude extended thinking 통합 | 확장 추론 |
| **web_search** | Perplexity API 기반 OSINT | 외부 정보 수집 |

- **도구 등록**: Decorator 기반 Registry, XML Schema, DefusedXML 보안 파싱 (XXE 방지) — [`registry.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/registry.py)
- **도구 호출 포맷**: `<function=tool_name><parameter=name>value</parameter></function>` (XML)
- **실행**: 반복당 1개 순차

### 3.2 CAI 툴킷 (24+ 도구, Kill Chain 기반)

| 카테고리 | 주요 도구 | 핵심 기여 |
|----------|------|-----------|
| **정찰** | [`nmap.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/reconnaissance/nmap.py), [`shodan.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/reconnaissance/shodan.py), [`google_search.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/web/google_search.py) | 네트워크/OSINT 정찰 |
| **실행** | [`generic_linux_command.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/reconnaissance/generic_linux_command.py) | 셸 Session 관리 + 명령 실행 |
| **분석** | [`capture_traffic.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/network/capture_traffic.py), [`code_interpreter.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/misc/code_interpreter.py) | 트래픽 분석/코드 실행 |
| **추론** | [`reasoning.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/misc/reasoning.py), [`rag.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/misc/rag.py) | 전략 분석/메모리 통합 |
| **C2** | [`command_and_control/`](https://github.com/aliasrobotics/cai/tree/e22a122/src/cai/tools/command_and_control) | 명령 제어 채널 |
| **웹** | [`js_surface_mapper.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/web/js_surface_mapper.py), [`webshell_suit.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/web/webshell_suit.py) | JS 자산 정찰, Webshell |

- **도구 호출 포맷**: OpenAI function calling JSON Schema
- **실행**: 턴당 N개 병렬 (`asyncio.gather`)
- ⚠️ 4개 Kill Chain 카테고리(`exploitation/`, `lateral_movement/`, `privilege_scalation/`, `data_exfiltration/`)는 디렉토리만 존재, 구현체 없음

### 3.3 Sandboxing: 도구 실행 환경

**Strix**: Docker Container 필수 — [`docker_runtime.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py)
- Kali Linux 이미지 — [L107](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py#L107)
- Container 내부 Tool Server, 포트 48081 — [L24](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py#L24)
- Token 인증 `secrets.token_urlsafe(32)` — [L124](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py#L124)
- `NET_ADMIN`, `NET_RAW` 권한 — [L134](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py#L134)
- `pentester` 사용자 (root 아님)
- **도구 실행 경로**: 호스트(Agent 로직) → HTTP POST → Container(도구 실행) → JSON 응답

**CAI**: Docker 선택적 — [`common.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/common.py) (1600+ 라인 공통 실행 엔진)
- 실행 환경 3분기: Docker (`docker exec` + TTY), CTF (`ctf.get_shell()`), 호스트 (PTY + `os.setsid()`)
- SSH (`paramiko`)를 통한 원격 실행도 지원
- 셸 Session 관리 (persistent sessions: S1, S2, ...) — [`generic_linux_command.py:70-115`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/reconnaissance/generic_linux_command.py#L70-L115)
- 10초 무출력 시 Process 자동 종료 (idle detection)
- **도구 실행 경로**: Agent Process 내에서 직접 실행 (in-process)

### 3.4 Insight: 왜 도구 통합이 효과적인가

| 관점 | Strix | CAI |
|------|-------|-----|
| **도구 수** | 13+ | 24+ |
| **도구 Schema** | XML Custom 포맷 | OpenAI function calling JSON |
| **실행** | 반복당 1개 순차 | 턴당 N개 병렬 |
| **Sandbox** | Docker 필수 (격리 강제) | Docker 선택 (유연성 우선) |
| **핵심 차별 도구** | Playwright 브라우저, HTTP Proxy | Kill Chain 도구 (C2, SDR, PCAP) |

**영역별 도구 비교**:

| 영역 | Strix | CAI |
|------|-------|-----|
| 셸 실행 | tmux 터미널 (영구 Session, PS1 exit code) | generic_linux_command + Session 관리 (S1, S2, ...) |
| 브라우저 | Playwright 멀티탭 (스크린샷 → LLM 비전) | 없음 (curl, HTTP 레벨 도구) |
| HTTP Proxy | Caido 기반 (HTTPQL 필터링, 요청 변조) | 없음 |
| 코드 실행 | Python (IPython, Proxy 함수 자동 주입) | 14개 언어 + CodeAgent (CodeAct Pattern) |
| 네트워크 정찰 | nmap (Container 내) | nmap + Shodan + Google Dorking |
| Webshell | 없음 | webshell_suit |
| C2 | 없음 | command_and_control |
| OSINT | Perplexity API | Perplexity + Google Custom Search |
| 추론 | think (부작용 없음) | think + write_key_findings (파일 영구 저장) |
| Reporting | CVSS 계산 + LLM 중복 검사 | Reporter Agent (HTML 보고서) |

**핵심 Insight**:

1. **PoC 실행이 false positive을 줄인다.**
- "SQLi 가능성 있음"이 아니라 sqlmap으로 실제 추출한 데이터를 보여준다. 순수 LLM은 추측만 할 수 있다.

2. **도구 Schema가 LLM의 환각을 제어한다.**
- XML이든 JSON이든, 구조화된 Schema는 LLM이 "올바른 형식의 도구 호출"만 하도록 강제한다. 순수 LLM은 자유 텍스트로 "이렇게 하면 됩니다"라고 설명만 한다.

3. **도구 범위가 커버리지를 결정한다.**
- Strix는 Playwright/Proxy로 웹 앱 깊이를 확보하고, CAI는 Kill Chain 전체(정찰~C2)를 도구화해 넓이를 확보한다.

---

## 4. Handoff — Agent 간 협력 구조

> 해결하는 문제: 정찰 결과를 익스플로잇 Agent에 정확히 전달해야 한다

### 4.1 Strix: 계층적 위임 (Hierarchical Delegation)

**메커니즘**: 루트 Agent가 Sub-Agent를 **daemon Thread**로 동적 생성

```
Root Agent (StrixAgent)
  ├── create_agent() → SQLi Validation Agent [Thread]
  ├── create_agent() → XSS Testing Agent [Thread]
  └── create_agent() → IDOR Discovery Agent [Thread]
       └── create_agent() → IDOR Reporting Agent [Thread]
```

**핵심 구현** ([`agents_graph_actions.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py)):
- `create_agent()` — [L188-281](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py#L188-L281): Sub-Agent를 새 Thread로 생성
- `send_message_to_agent()` — [L285-352](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py#L285-L352): 비동기 Message 큐 (type: query/instruction/information, priority: low~urgent)
- `agent_finish()` — [L356-384](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py#L356-L384): 완료 시 부모에게 XML Report 전송
- **Context 상속**: `inherit_context=True` ([L192](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py#L192))이면 `<inherited_context_from_parent>` 태그로 전달
- **Identity 격리**: "You are NOT your parent agent. You are a NEW, SEPARATE sub-agent" 주입
- **공유 자원**: 모든 Agent가 동일 `/workspace`와 Proxy History 공유
- **Message 포맷**: XML 구조 (`<inter_agent_message>`, `<agent_completion_report>`)

### 4.2 CAI: Swarm 기반 전환 (Peer-to-Peer Transfer)

**메커니즘**: Agent 간 **양방향 Handoff**를 통한 제어권 전환

```
Thought Agent ←→ Red Team Agent ←→ DNS/SMTP Agent
                    ↕
              Bug Bounty Agent
```

**핵심 구현** ([`handoffs.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/sdk/agents/handoffs.py)):
- `handoff()` — [L150-236](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/sdk/agents/handoffs.py#L150-L236): Handoff를 도구로 등록 (e.g., `transfer_to_redteam_agent`)
- `Handoff` 클래스 — [L53-115](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/sdk/agents/handoffs.py#L53-L115): Handoff 정의
- LLM이 Handoff 도구를 호출하면 제어권이 타겟 Agent로 이동
- **Message History 공유**: Handoff 시 전체 대화 기록이 새 Agent로 전달
- **Agent 복제**: `agent.clone()`으로 독립 사본 생성 — [`red_team.py:17-19`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/patterns/red_team.py#L17-L19)

**Pattern 예시** ([`red_team.py:44-46`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/patterns/red_team.py#L44-L46)):
```python
# 양방향 에지 구성
redteam_agent.handoffs.append(dns_smtp_handoff)    # Red → DNS
dns_smtp_agent.handoffs.append(redteam_handoff)    # DNS → Red
thought_agent.handoffs.append(redteam_handoff)     # Thought → Red
```

### 4.3 Insight: 왜 Handoff가 효과적인가

| 관점 | Strix | CAI |
|------|-------|-----|
| **구조** | 트리 (부모→자식, 단방향) | 그래프 (P2P, 양방향) |
| **생성 방식** | 동적 (런타임에 필요 시 생성) | 정적 (사전 정의된 Agent 풀) |
| **동시성** | 멀티 Thread 병렬 실행 | 순차 전환 (한 번에 하나만 활성) |
| **통신** | 비동기 Message 큐 (XML) | 대화 기록 전체 전달 |
| **Context** | 선택적 상속 + Identity 격리 | 전체 History 공유 |
| **제어 주체** | 부모 Agent가 결정 | LLM이 자율 결정 (도구 선택) |
| **복귀** | `agent_finish()` → 부모에 보고 | 역방향 Handoff로 복귀 |
| **확장성** | 수십 개 Sub-Agent 병렬 가능 | Agent 수 = 사전 등록된 Handoff 수 |

**핵심 Insight**:

1. **Handoff = 구조화된 정보 전달.**
- 순수 LLM 대화에서는 "아까 nmap 결과에서..."라고 참조하면 정확도가 떨어진다. Strix는 XML Report로, CAI는 전체 History 전달로 정보 손실을 방지한다.

2. **트리 vs 그래프는 다른 문제를 풀다.** 
- Strix의 트리 구조는 "하나의 웹 앱을 여러 각도로 동시 공격"에 최적화되어 있고, CAI의 그래프 구조는 "Kill Chain 단계 간 유기적 전환"에 최적화되어 있다.

3. **Context 전달 방식이 비용을 결정한다.** 
- Strix의 "선택적 상속"은 필요한 정보만 전달해 Token을 절약하고, CAI의 "전체 History 공유"는 정보 손실 없이 전달하지만 Token이 누적된다.

---

## 5. 메모리와 학습 — 경험 축적

> 해결하는 문제: LLM의 Context 윈도우는 유한하고, Session이 끝나면 모든 것을 잊는다

### 5.1 Strix: 보안 인식 대화 압축

**MemoryCompressor** — [`memory_compressor.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py):
- 대화 기록이 **100K Token 초과** 시 초기 Message 요약 — [L12](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py#L12) (`MAX_TOTAL_TOKENS = 100_000`)
- LLM 기반 요약 (10개씩 청크 → `<context_summary>` 태그) — [L172-225](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py#L172-L225)
- 최근 15개 Message는 원본 유지 — [L13](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py#L13) (`MIN_RECENT_MESSAGES = 15`)
- **보안 인식 압축**: 취약점, 자격증명, 시스템 아키텍처, 실패한 시도를 **우선 보존**
- 이미지 처리: 최근 3개만 유지, 나머지 텍스트 참조로 대체
- **Session 간 학습 없음** — 각 스캔은 독립적

### 5.2 CAI: Vector DB 기반 Session 간 학습

**Episodic + Semantic + RAG** — [`memory.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/memory.py):
- **Episodic**: Qdrant Vector DB에 타겟별 시간순 기록 저장 — [L14-18](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/memory.py#L14-L18)
- **Semantic**: 모든 타겟 데이터를 단일 글로벌 컬렉션에 Vector화 — [L20-24](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/memory.py#L20-L24)
- **RAG Pipeline**: LLM 요약 → Vector 임베딩 → 유사도 검색
- **Session 간 학습**: 이전 침투 테스트 경험을 다음 Session에 활용 — [L36-40](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/memory.py#L36-L40) (Online Learning)
- **Online/Offline 모드**: 실시간 업데이트 또는 Batch 학습
- ⚠️ **코드 이슈**: `memory.py`에 `model_name` 미정의 변수 (L200, 215, 229), `query_agent`에 도구 미정의이나 `tool_choice="required"` 설정

### 5.3 취약점 중복 제거

**Strix**: LLM 기반 자동 중복 검사 — [`dedupe.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/dedupe.py)
- `check_duplicate()` — [L141-217](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/dedupe.py#L141-L217): 새 취약점을 기존 Report와 LLM으로 비교
- 동일 근본 원인, 동일 컴포넌트, 동일 공격 Vector = 중복
- 다른 엔드포인트의 동일 유형 = 중복 아님

**CAI**: 자동 중복 제거 없음
- Reporter Agent가 수동 정리
- Flag discriminator로 CTF flag 판별만 존재

### 5.4 Insight: 왜 메모리가 효과적인가

| 관점 | Strix | CAI |
|------|-------|-----|
| **Session 내** | 100K Token 초과 시 LLM 압축 (보안 정보 우선) | 동일 원리 |
| **Session 간** | 없음 (각 스캔 독립) | Qdrant Vector DB (Episodic + Semantic) |
| **중복 제거** | LLM 기반 자동 | 없음 (수동) |

**핵심 Insight**:

1. **"보안 인식" 압축이 핵심이다.**
- 일반적인 대화 요약은 모든 내용을 균등하게 압축하지만, Strix는 취약점/자격증명을 우선 보존한다. 이것이 단순 truncation과의 차이다.

2. **Session 간 학습은 반복 테스트에서 가치가 크다.**
- 같은 타겟을 주기적으로 스캔할 때, CAI는 "지난번에 이 포트에서 이 취약점을 찾았다"를 기억한다.
- Strix는 매번 처음부터 시작한다.

3. **중복 제거는 Report 품질을 결정한다.** 
- Strix의 LLM 기반 중복 검사는 "같은 SQLi를 다른 파라미터에서 발견"한 경우를 구분한다. 
- CAI에는 이 기능이 없어 수동 정리가 필요하다.

---

## 6. 안전장치와 제어 — 환각·False Positive·비용 관리

> 해결하는 문제: LLM은 환각하고, 비용이 폭주하고, 위험한 행동을 할 수 있다

### 6.1 Guardrail

**Strix**: Guardrail 없음
- Prompt Injection 방어 없음
- 타겟 웹 앱이 악의적 응답을 반환하면 Agent가 영향받을 수 있음

**CAI**: 3계층 방어 — [`guardrails.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/guardrails.py), [`sdk/guardrail.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/sdk/agents/guardrail.py)
1. **입력 Guardrail**: Pattern 매칭, 유니코드 Homoglyph 탐지 ([L81-131](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/guardrails.py#L81-L131)), Base64/32 디코딩
2. **출력 Guardrail**: 위험 명령 탐지 (역쉘, 포크밤), 데이터 유출 방지
3. **탐지 Pattern** (~15개 핵심 정규식 + Unicode/Base64/32 체크) — [L40-78](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/guardrails.py#L40-L78): Instruction 오버라이드, 숨겨진 XML 태그, 인코딩 트릭, 역할 조작
4. 설정: `CAI_GUARDRAILS=true/false`, Tripwire — [`guardrail.py:29-32`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/sdk/agents/guardrail.py#L29-L32)

### 6.2 비용 제어

**Strix** — [`llm.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py):
- `litellm.completion_cost()` 호출 — [L252](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py#L252)
- `RequestStats` 클래스 (input/output tokens, cost) — [L42-57](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py#L42-L57)
- **비용 한도 없음** — 사용자가 수동 모니터링 필요
- Streaming 중 첫 `</function>` 태그에서 중단 → 출력 Token 절약

**CAI** — [`util.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/util.py):
- `CostTracker` 클래스: Session/Agent/Interaction 3단계 비용 추적 — [L392-440](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/util.py#L392-L440)
- `CAI_PRICE_LIMIT` — [L422-426](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/util.py#L422-L426): 달러 한도 초과 시 **자동 중단**
- `GlobalUsageTracker` — [`global_usage_tracker.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/sdk/agents/global_usage_tracker.py): 글로벌 사용량 파일 락 기반 추적 (`~/.cai/usage.json`)

**비용 구조 이론적 비교**:

| 비용 요인 | Strix | CAI |
|-----------|-------|-----|
| **Agent당 오버헤드** | 높음 (시스템 Prompt 407줄 + Skill 5개 매번 주입) | 중간 (Agent별 Prompt 61-191줄) |
| **도구 호출 효율** | 반복당 1개 → 도구 결과마다 LLM 호출 | 턴당 N개 → 다수 결과를 한 번에 처리 |
| **Sub-Agent** | 수십 개 병렬 → 각각 독립 LLM Session (비용 ×N) | Swarm → 순차 전환 (History 누적) |
| **비용 Pattern** | **폭넓은 병렬 비용** (시스템 Prompt 반복) | **깊은 순차 비용** (History 점진 증가) |

### 6.3 False Positive와 탐지 품질

**Strix ("GO SUPER HARD" 접근)**:
- 장점: "2000+ steps", "$500+ Bug Bounty 가치" 강조 → 깊이 파고듦, LLM 중복제거 + PoC 검증
- 위험: Guardrail 부재 → Prompt Injection에 취약, 공격적 Payload → 타겟 부작용 가능

**CAI ("Safe First" 접근)**:
- 장점: 3계층 Guardrail, "read-only first", HITL, 메모리로 이전 False Positive 학습
- 위험: 보수적 → 깊이 숨겨진 취약점 미발견, Guardrail 과민 탐지 가능

**이론적 탐지 품질**:

| 관점 | Strix | CAI |
|------|-------|-----|
| **True Positive** | 높음 (공격적 + PoC 검증 필수) | 중간 (체계적이지만 보수적) |
| **False Positive** | 낮음 (Validation Agent + LLM 중복제거) | 중간 (자동 중복제거 없음) |
| **False Negative** | 낮음 (2000+ steps 강제) | 중간 (safe-first로 일부 False Negative) |
| **타겟 안전성** | 중간 (비파괴 주장, 보장 없음) | 높음 (read-only + HITL) |

> ⚠️ 이론적 분석입니다. 실제 비교에는 DVWA 등 동일 타겟에서의 실험이 필요합니다.

### 6.4 Insight: 안전장치의 Trade-off

**핵심 Insight**:

1. **Guardrail vs 공격 효과는 Trade-off다.** 
- CAI의 Guardrail은 Prompt Injection을 방어하지만, 정상적인 공격 Payload도 차단할 수 있다.
- Strix는 Guardrail 없이 공격 효과를 극대화하지만, 타겟의 악의적 응답에 취약하다.

2. **PoC 검증이 false positive을 줄이는 효과적인 방법이다.**
- Strix의 Validation Agent는 발견된 취약점을 실제 실행해서 확인한다.
- CAI에는 이 자동 검증 Loop가 없다.

3. **비용 제어는 프로덕션 환경에서 필수다.** 
- Strix는 Sub-Agent가 무한히 생성될 수 있고 비용 한도가 없다.
- CAI의 `CAI_PRICE_LIMIT`은 실수로 인한 비용 폭주를 방지한다.

---

## 7. 종합 비교

### 도구 개요

| 항목 | Strix (v0.7.0) | CAI (v0.5.10) |
|------|----------------|---------------|
| **포지션** | DevSecOps 웹 앱 자동화 | 종합 사이버보안 Framework |
| **개발** | usestrix (커뮤니티) | Alias Robotics (기업) |
| **라이선스** | Apache-2.0 (상업 무료) | Open Core (상업 €350/월) |
| **Python** | 3.12+ | 3.9+ |
| **SDK** | 독자 개발 | OpenAI Agents SDK 포크 |
| **LLM 추상화** | LiteLLM ~1.81.1 | OpenAI SDK 1.75.0 + LiteLLM |
| **GitHub Stars** | ~19.5K | ~6.9K |

### 아키텍처 비교

| 영역 | Strix | CAI |
|------|-------|-----|
| **Template** | Jinja2 (~407줄) | Mako (~190줄) |
| **톤** | 공격적/완전 자율 | 체계적/안전 우선 |
| **Agent 생성** | 동적 (런타임) | 정적 (~20개 사전 정의) |
| **구조** | 트리 (부모→자식) | 그래프 (P2P 양방향) |
| **동시성** | Multi-Thread 병렬 | 순차 전환 (Parallel 옵션) |
| **Agent Loop** | 1도구/반복, 300회 한도 | N도구/턴 병렬, Guardrail 내장 |
| **도구 Schema** | XML Custom | OpenAI function calling JSON |
| **핵심 도구** | Playwright, Caido Proxy | Session 관리, CodeAgent, C2 |
| **Sandbox** | Docker 필수 | Docker 선택 (4환경 자동 라우팅) |
| **Session 간 메모리** | 없음 | Qdrant Vector DB |
| **중복 제거** | LLM 자동 | 없음 |
| **Guardrail** | 없음 | 3계층 (Pattern + 유니코드 + AI) |
| **비용 한도** | 없음 | `CAI_PRICE_LIMIT` 자동 중단 |
| **모델 지정** | 전체 통일 | Agent별 개별 지정 가능 |

### 보안 테스트 커버리지

| 영역 | Strix | CAI |
|------|:-----:|:---:|
| 웹 앱 / API / SAST | ✅ | ✅ |
| 브라우저 자동화 (Playwright) | ✅ | ❌ |
| HTTP Proxy (HTTPQL) | ✅ | ❌ |
| 네트워크 침투 테스트 | ⚠️ (기본) | ✅ |
| 바이너리/리버싱 | ❌ | ✅ |
| 모바일 (Android) | ❌ | ✅ |
| IoT/임베디드/무선 | ❌ | ✅ |
| 포렌식/메모리/PCAP | ❌ | ✅ |
| 벤치마크 / CTF 자동 풀이 | ✅ ([XBEN](https://github.com/usestrix/benchmarks/tree/main/XBEN) 104 챌린지, 96%) | ✅ (CAIBench 100+ 챌린지) |

**요약**: Strix = 웹 앱 깊이, CAI = 보안 영역 넓이

---

## 부록 A: 라이선스 핵심

- **Strix**: Apache-2.0 → 개인/상업 모두 **무료**
- **CAI**: Open Core → 연구/학습 무료, 상업적 사용 시 **CAI PRO €350/월** 필요
- CAI SDK 코드(`src/cai/agents/`)는 MIT, 나머지 Alias Robotics 코드는 Research-Use Proprietary

## 부록 B: 선택 가이드

| 사용 사례 | 추천 | 이유 |
|-----------|:----:|------|
| 웹 앱 CI/CD 보안 자동화 | Strix | GitHub Actions, 헤드리스, exit code |
| 자동 코드 패치 및 PR | Strix | file_edit + 검증 Loop |
| 상업적 프로덕트 통합 | Strix | Apache-2.0 무료 |
| 종합 침투 테스트 (네트워크+웹) | CAI | Kill Chain 기반, 네트워크 도구 풍부 |
| CTF/보안 교육 | CAI | CAIBench 100+ 챌린지 |
| 포렌식/사고 대응 | CAI | DFIR, 메모리/PCAP 분석 |
| IoT/모바일 보안 | CAI | Android SAST, Sub-GHz SDR |

## 부록 C: 향후 연구 방향

- [ ] 동일 타겟(DVWA 등) 대상 기법별 효과 정량 비교
- [ ] LLM 모델별(GPT-5, Claude, alias1) 성능 차이 분석
- [ ] 비용 효율성 비교 (동일 취약점 발견까지의 Token/비용 측정)
- [ ] 하이브리드 설계: Strix의 병렬 Agent + 자동 패치 + CAI의 메모리/Guardrail
- [ ] CAI의 Session 간 학습이 반복 스캔 효율성에 미치는 영향 측정
- [ ] Strix의 Guardrail 부재가 실제 공격 시나리오에서 미치는 위험 평가