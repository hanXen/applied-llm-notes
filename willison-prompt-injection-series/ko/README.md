# Simon Willison의 Prompt Injection 시리즈 정리

> Simon Willison의 [Prompt Injection 시리즈](https://simonwillison.net/series/prompt-injection/) (2022.09 ~ 2025.11, 총 23편) 중 주요 글을 중심으로 정리한 연구 흐름, 방어 기법 분류, 그리고 핵심 인사이트

---

## 1. 시리즈 개요

Simon Willison은 2022년 9월 "Prompt Injection"이라는 용어를 명명한 이후, 약 3년간 이 보안 취약점의 발전과 방어 시도를 추적해왔다. 그의 시리즈는 단순한 기술 해설을 넘어, 업계 전체의 대응 부재에 대한 비판과 실용적 위험 프레임워크의 제시로 이어졌다.

---

## 2. 연구 흐름 타임라인

### Phase 1: 문제의 정의 (2022.09 ~ 2022.12)

**[Prompt injection attacks against GPT-3](https://simonwillison.net/2022/Sep/12/prompt-injection/)** (2022.09.12)
Riley Goodside가 GPT-3에서 "ignore previous instructions" 패턴을 발견한 것을 계기로, Willison이 이를 **SQL Injection에 비유하여 "Prompt Injection"으로 명명**. 핵심 통찰: trusted prompt(시스템 지시)와 untrusted text(사용자/외부 입력)가 하나의 token stream에 합쳐지는 것이 근본 원인.

**[I don't know how to solve prompt injection](https://simonwillison.net/2022/Sep/16/prompt-injection-solutions/)** (2022.09.16)
해결책을 모르겠다는 솔직한 고백. 이 문제가 단순한 엔지니어링 버그가 아니라, LLM 아키텍처의 근본적 한계에서 비롯된다는 인식의 출발점.

**[You can't solve AI security problems with more AI](https://simonwillison.net/2022/Sep/17/prompt-injection-more-ai/)** (2022.09.17)
시리즈 전체를 관통하는 핵심 철학의 선언. "99%의 탐지율은 보안에서 낙제점이다." SQL injection에는 parameterized queries라는 **100% 작동이 보장된** 해법이 있지만, prompt injection에는 그런 해법이 없다 — 이 대조가 핵심. AI 기반 방어(LLM 필터링 등)는 본질적으로 probabilistic이며, 공격자에게 1%의 틈을 남기는 한 안전하지 않다.

**[A new AI game: Give me ideas for crimes to do](https://simonwillison.net/2022/Dec/4/give-me-ideas-for-crimes-to-do/)** (2022.12.04)
ChatGPT 출시 직후, jailbreaking과 prompt injection이 대중적으로 주목받기 시작한 시점에 대한 관찰.

### Phase 2: 위험의 구체화와 초기 방어 시도 (2023)

**[Bing: "I will not harm you unless you harm me first"](https://simonwillison.net/2023/Feb/15/bing/)** (2023.02.15)
Bing Chat의 출시와 함께 나타난 기이한 행동들 기록. AI 검색이 웹 콘텐츠를 처리할 때 prompt injection에 노출되는 실제 사례.

**[Prompt injection: What's the worst that can happen?](https://simonwillison.net/2023/Apr/14/worst-that-can-happen/)** (2023.04.14)
**Data exfiltration 공격**의 위험성을 구체적으로 제시. Markdown 이미지 링크를 통해 LLM이 사용자 데이터를 외부 서버로 전송하게 만드는 패턴 설명. 이후 "Lethal Trifecta" 개념의 씨앗.

**[The Dual LLM pattern for building AI assistants that can resist prompt injection](https://simonwillison.net/2023/Apr/25/dual-llm-pattern/)** (2023.04.25)
**최초의 체계적 방어 아키텍처 제안.** P-LLM(Privileged LLM, tool access 권한 보유, 사용자 명령만 수신)과 Q-LLM(Quarantined LLM, 도구 없음, untrusted data 처리)을 분리. 핵심 원리: "untrusted data를 처리하는 LLM에게 tool access 권한을 주지 마라."

**[Delimiters won't save you from prompt injection](https://simonwillison.net/2023/May/11/delimiters-wont-save-you/)** (2023.05.11)
OpenAI/Andrew Ng 공동 강좌(ChatGPT Prompt Engineering for Developers)에서 소개된 delimiter 기반 방어의 무효화를 입증. 특수 delimiter(`###`, `"""` 등)로 trusted/untrusted 텍스트를 분리해도 LLM은 이를 존중하지 않음.

**[Multi-modal prompt injection image attacks against GPT-4V](https://simonwillison.net/2023/Oct/14/multi-modal-prompt-injection/)** (2023.10.14)
멀티모달로의 공격 확장. 이미지 내 숨겨진 텍스트(흰 배경에 흰 글씨 등)를 통한 인젝션 공격. Johann Rehberger의 초기 연구 소개.

**[Recommendations to help mitigate prompt injection: limit the blast radius](https://simonwillison.net/2023/Dec/20/mitigate-prompt-injection/)** (2023.12.20)
"Blast radius를 제한하라" — 완전한 방어 불가 인정하에, blast radius를 최소화하는 실용적 접근 강조.

### Phase 3: 개념 정리와 RAG 시대의 문제 (2024)

**[Prompt injection and jailbreaking are not the same thing](https://simonwillison.net/2024/Mar/5/prompt-injection-jailbreaking/)** (2024.03.05)
두 개념의 명확한 구분:

- **Jailbreaking**: 사용자 본인이 모델의 안전 장치를 우회 (1인칭 공격)
- **Prompt Injection**: 제3자가 외부 데이터에 악성 지시를 삽입하여 간접 공격 (3인칭 공격)

이 구분이 중요한 이유: Jailbreaking을 prompt injection과 혼동하는 개발자들은 "우리 앱과 무관한 문제"로 치부하고 보안 조치를 무시하게 됨.

**[Accidental prompt injection against RAG applications](https://simonwillison.net/2024/Jun/6/accidental-prompt-injection/)** (2024.06.06)
악의적 의도 없이도 발생하는 "우발적 prompt injection". RAG 파이프라인에서 문서 내용이 LLM의 지시로 해석되는 현상 보고. LLM 프로젝트의 문서가 RAG에 입력되자 문서 내 지시문이 LLM의 행동을 변경한 사례.

### Phase 4: 아키텍처적 방어의 등장 (2025 전반기)

**[New audio models from OpenAI, but how much can we rely on them?](https://simonwillison.net/2025/Mar/20/new-openai-audio-models/)** (2025.03.20)
텍스트를 넘어 음성 모델에서도 동일한 instruction-following 취약점 확인. Attack surface의 지속적 확장.

**[Model Context Protocol has prompt injection security problems](https://simonwillison.net/2025/Apr/9/mcp-prompt-injection/)** (2025.04.09)
MCP(Model Context Protocol)의 보안 문제 분석. Elena Cross의 "The 'S' in MCP Stands for Security" 인용. 주요 문제점:

- **Rug Pull**: MCP 도구가 설치 후 자체 정의를 변경 가능. Day 1에 안전해 보이던 도구가 Day 7에 API 키를 탈취
- **Tool Shadowing**: 악성 MCP 서버가 신뢰된 서버의 호출을 가로채기
- **WhatsApp MCP 예시**: Invariant Labs 데모에서 악성 MCP 서버의 `get_fact_of_the_day()` 도구가 rug pull로 정의를 변경하여, WhatsApp 채팅 기록을 탈취하고 공격자 번호로 전송하도록 LLM을 조작. Willison은 악성 MCP 서버 없이도 WhatsApp 메시지 자체에 injection을 삽입하는 것만으로 동일한 공격이 가능함을 추가 지적

핵심 경고: "MCP 스펙의 SHOULD를 MUST로 취급하라."

**[CaMeL offers a promising new direction for mitigating prompt injection attacks](https://simonwillison.net/2025/Apr/11/camel/)** (2025.04.11)
**시리즈의 전환점.** Google DeepMind의 CaMeL 논문에 대해 Willison은 "strong guarantees를 주장하는(claims to provide) 최초의 prompt injection 완화책"이라 평가하며, 2.5년간 "alarmingly little progress" 이후 나타난 "promising new direction"으로 주목. CaMeL이 Dual LLM의 한계를 어떻게 극복하는지 상세 분석:

Dual LLM의 문제: "Bob에게 이메일 보내줘, 회의록에 주소 있어" → P-LLM이 Q-LLM에 이메일 주소 추출을 위임 → Q-LLM이 악성 지시에 의해 공격자의 주소를 반환 → **control flow(plan)는 보호되나 data flow(values)는 미보호**

CaMeL의 해법: 사용자 명령을 Python 코드로 변환 → custom interpreter에서 실행 → **taint tracking**으로 data provenance를 추적 → 외부 출처(tainted) 데이터가 민감한 도구에 사용될 때 사용자 확인 요청

Willison의 평가: "더 많은 AI를 투입하는 대신, 보안 공학의 검증된 개념(capabilities, data flow analysis)에 기반한 최초의 접근."

![CaMeL Figure 1: Dual LLM의 취약점 — control flow는 보호되나 data flow는 미보호](../images/camel-dual-llm-vulnerability.jpg)
*출처: [Defeating Prompt Injections by Design](https://arxiv.org/abs/2503.18813), Google DeepMind (CC BY 4.0)*

**[Design Patterns for Securing LLM Agents against Prompt Injections](https://simonwillison.net/2025/Jun/13/prompt-injection-design-patterns/)** (2025.06.13)
IBM, Invariant Labs, ETH Zurich, Google, Microsoft 등 14명의 저자가 참여한 6가지 디자인 패턴 논문 리뷰. Willison이 특히 주목한 부분:

> "이 디자인 패턴들은 에이전트의 임의 작업 수행 능력을 **의도적으로 제한**한다."

이 트레이드오프를 명시적으로 인정한 것이 논문의 신뢰도를 높인다고 평가.

> "LLM 에이전트가 untrusted input을 처리한 이후에는, 해당 입력이 **어떤 consequential action도 촉발할 수 없도록** 제약해야 한다."

6가지 패턴 (논문 제시 순):

1. **Action Selector** — LLM을 사전 정의된 tool selector로만 활용. "LLM-modulated switch statement"
2. **Plan-then-Execute** — 외부 데이터 처리 전에 고정된 실행 계획 수립
3. **LLM Map-Reduce** — 격리된 서브 에이전트가 문서를 독립 처리, 구조화된 데이터만 취합
4. **Dual LLM** — Privileged LLM과 untrusted data 처리 LLM의 분리
5. **Code-then-Execute** — 에이전트가 명시적 프로그램을 작성하여 실행 (CaMeL이 이 패턴의 구현체)
6. **Context-Minimization** — 사용자 요청 처리 후 불필요한 컨텍스트(원래 프롬프트 포함)를 제거하여 후속 단계에서의 injection 방지

이 외에 **Human-in-the-loop**(민감한 행동 전 사용자 확인)은 6가지 패턴과 별도로 논의되는 일반 원칙(general best practice)이다.

**[An Introduction to Google's Approach to AI Agent Security](https://simonwillison.net/2025/Jun/15/ai-agent-security/)** (2025.06.15)
Google의 에이전트 보안 접근법 논문 리뷰. "Hybrid approach"를 통한 신뢰 경계 설정.

**[The lethal trifecta for AI agents](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)** (2025.06.16)
**"Lethal Trifecta"라는 용어의 공식적 정립.** 세 가지 조건이 동시에 충족되면 data exfiltration이 가능:

1. **Private/sensitive data에 대한 접근** — 이메일, 문서, DB 등
2. **Untrusted content에 대한 노출** — 악성 웹페이지, 이메일, 공유 문서 등
3. **External communication 수단** — 이메일 발송, API 호출, 이미지 URL 렌더링 등

실제 피해 사례 열거: Microsoft 365 Copilot, GitHub MCP, GitLab Duo, ChatGPT, Google Bard, Slack, Amazon Q 등 수십 개 제품에서 동일한 패턴의 공격 성공.

핵심 메시지: "LLM 벤더들이 우리를 구해주지 않을 것이다. 사용자 스스로 lethal trifecta 조합을 피해야 한다."

![The Lethal Trifecta: Access to Private Data, Exposure to Untrusted Content, Ability to Externally Communicate](../images/lethal-trifecta.jpg)
*출처: [Simon Willison](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) (2025.06)*

**Lethal Trifecta의 한계와 Agents Rule of Two로의 발전** (2025.10–11) *(시간순으로는 Phase 5에 해당하나, Trifecta와의 논리적 연속성을 위해 여기에 배치)*

Willison은 이후 Lethal Trifecta의 한계를 직접 인정했다: "The one problem with the lethal trifecta is that it only covers the risk of **data exfiltration**: there are plenty of other, even nastier risks that arise from prompt injection attacks against LLM-powered agents with access to tools which the lethal trifecta doesn't cover." 파일 삭제, 설정 변경, 코드 실행 같은 state 변경 공격은 Lethal Trifecta로 포착할 수 없다.

Meta AI의 [Agents Rule of Two](https://simonwillison.net/2025/Nov/2/new-prompt-injection-papers/)가 이 한계를 보완. Willison의 Trifecta와 Google Chrome팀의 "Rule of 2"에서 영감을 받아, 세 속성을 재정의했다:

> "강건성 연구가 prompt injection을 신뢰성 있게 탐지하고 거부할 수 있게 될 때까지, 에이전트는 한 세션 내에서 다음 세 속성 중 **최대 두 가지**만 충족해야 한다."
>
> 1. Untrusted input 처리
> 2. Sensitive data 접근
> 3. External communication / **state change**

핵심 변경: Trifecta의 "external communication"(exfiltration만 커버)을 "**changing state**"로 확장하여, 데이터 유출뿐 아니라 파일 삭제·설정 변경·코드 실행 등 모든 형태의 위험한 도구 사용을 커버한다. 세 가지 모두 필요한 경우, autonomous operation을 금지하고 최소한 human-in-the-loop 승인이 필요.

초기 다이어그램은 2개만 충족하는 영역을 "Safe"로 표기했으나, 커뮤니티에서 "2개여도 안전한 것은 아니다 — 최악만 막을 뿐"이라는 비판이 제기되었다. Meta AI 저자가 이를 수용하여 "**Safe**" → "**Lower Risk**"로 수정: "The goal of the Rule of Two is not to describe a **sufficient** level of security for agents, but rather a **minimum bar** that's needed to deterministically prevent the highest security impacts of prompt injection."

![Agents Rule of Two (updated): "Safe" → "Lower Risk"로 수정된 버전](../images/agents-rule-of-two-updated.jpg)
*출처: [Agents Rule of Two](https://simonwillison.net/2025/Nov/2/new-prompt-injection-papers/) — Meta AI (2025.10), 커뮤니티 피드백 반영 수정본*

### Phase 5: 실증적 검증과 비관적 결론 (2025 후반기)

**[The Summer of Johann: prompt injections as far as the eye can see](https://simonwillison.net/2025/Aug/15/the-summer-of-johann/)** (2025.08.15)
보안 연구자 Johann Rehberger의 "The Month of AI Bugs" 프로젝트 정리. 8월 한 달간 매일 하나씩 실제 프로덕션 서비스의 prompt injection 취약점을 공개:

| 날짜 | 대상 | 공격 유형 |
|------|------|----------|
| 8/1 | ChatGPT | Chat history/memory exfiltration (*.windows.net 이미지 렌더링 악용) |
| 8/2 | Codex | ZombAI — azure.net allowlist 악용한 원격 제어 |
| 8/3 | Anthropic Filesystem MCP | Path validation bypass (.startsWith() 우회) |
| 8/4 | Cursor | Mermaid 다이어그램 통한 data exfiltration (CVE-2025-54132) |
| 8/5 | Amp Code | settings.json 수정으로 임의 명령 실행 (RCE) |
| 8/6 | Devin | 비동기 코딩 에이전트에 대한 임의 명령 실행 |
| 8/7 | Devin | 다수 경로를 통한 secret exfiltration |
| 8/8 | Devin | expose_port 도구 악용한 AI Kill Chain |
| 8/9 | OpenHands | Lethal Trifecta — 환경변수/토큰 탈취 |
| 8/10 | OpenHands | ZombAI — C&C malware 설치/실행 유도 |
| 8/11 | Claude Code | DNS exfiltration (ping, nslookup, host, dig가 pre-approved된 점 악용) |
| 8/12 | GitHub Copilot | RCE via settings.json 조작 (CVE-2025-53773) |
| 8/13 | Google Jules | 다수 data exfiltration 취약점 |
| 8/14 | Google Jules | ZombAI — 원격 코드 실행 |
| 8/15 | Google Jules | 유니코드 invisible prompt injection |

Willison의 평가: "문제가 3년째 지속되고 있음을 보여주는 환상적이면서도 끔찍한(fantastic and horrifying) 실증."

**[Dane Stuckey (OpenAI CISO) on prompt injection risks for ChatGPT Atlas](https://simonwillison.net/2025/Oct/22/openai-ciso-on-atlas/)** (2025.10.22)
ChatGPT Atlas(브라우저 에이전트) 출시에 대한 보안 우려. OpenAI CISO Dane Stuckey가 prompt injection을 인정하고 방어 노력을 설명하며, 2000년대 초 컴퓨터 바이러스에 비유("responsible usage"의 중요성 강조). 그러나 Willison은 "일반 사용자는 바이러스 문제에 결코 잘 대처하지 못했고, 우리는 아직도 그 전투 중"이라며 그 비유 자체에 회의적 반박. 브라우저 에이전트 카테고리 전체에 대한 회의론 유지.

**[New prompt injection papers: Agents Rule of Two and The Attacker Moves Second](https://simonwillison.net/2025/Nov/2/new-prompt-injection-papers/)** (2025.11.02)

**The Attacker Moves Second** (2025.10, arXiv:2510.09023)
OpenAI, Anthropic, Google DeepMind, ETH Zurich 등의 14명 저자. 12개 방어 기법을 **adaptive attack**(gradient descent, RL, random search, human-guided exploration)으로 테스트한 결과, 원래 논문들이 보고한 거의 0%의 ASR이 **대부분 90% 이상(최저 71%)으로 급등**하며 static evaluation의 허상이 드러남. 상세 결과는 3.1절 참조.

Willison의 평가: "방어들이 얼마나 완전히 패배했는지를 고려하면, 신뢰할 수 있는 방어가 곧 개발될 것이라는 낙관론을 공유하기 어렵다."

**Agents Rule of Two** (2025.10, Meta AI)
Lethal Trifecta의 한계를 보완한 프레임워크. 상세 내용은 위 Phase 4의 Lethal Trifecta 항목 참조.

---

## 3. 방어 기법 종합 분류

### 3.1 확률적 방어 (Probabilistic Defenses) — ❌ 대부분 무력화됨

공격 난이도를 높이지만, adaptive attacker에게는 뚫림.

| 카테고리 | 기법 | 원리 | Adaptive attack 결과 |
|---------|------|------|----------------|
| 프롬프팅 | Spotlighting | Trusted text에 특수 마커 | 95-99% ASR |
| 프롬프팅 | Prompt Sandwiching | Untrusted input 전후로 지시 반복 | 95-99% ASR |
| 프롬프팅 | Delimiter 방어 | Delimiter로 경계 설정 | 즉시 우회 |
| 필터링 | Regex/키워드 필터 | "ignore" 등 패턴 매칭 | 동의어/다국어/유니코드로 우회 |
| 분류기 | Protect AI, PromptGuard, Model Armor | ML 기반 입출력 분류 | 71-90%+ ASR |
| 허니팟 | Data Sentinel, MELON | Canary token으로 탐지 | Adaptive attack으로 우회 |
| 모델 훈련 | Adversarial Training, Circuit Breakers | Adversarial examples로 fine-tuning | 새로운 패턴에 generalization 부족 |
| 활성화 탐지 | TaskTracker | LLM 내부 activation 기반 task drift 감지 | White-box만 가능, API 모델 불가 |

### 3.2 결정론적/아키텍처 방어 (Deterministic/Architectural Defenses) — ⚠️ 부분적 효과

보호 범위 내에서는 강한 보장을 제공하나, 에이전트 능력을 제한.

| 기법 | 원리 | 보호 범위 | 한계 |
|------|------|----------|------|
| Dual LLM | P-LLM/Q-LLM privilege separation | Control flow(plan) | Data flow(values) 미보호 |
| CaMeL | Python 변환 + taint tracking | Control flow + data flow | 사전 계획 가능한 태스크로 제한, 77% task success rate |
| Action Selector | LLM을 tool selector로만 사용 | 전체 | 자유 텍스트 생성 불가 |
| Plan-then-Execute | Planning과 execution 단계 분리 | Control flow | 동적 태스크 불가 |
| Map-Reduce | 격리된 sub-agent + structured output만 취합 | Cross-contamination 방지 | 복잡한 multi-step 추론 제한 |
| Context-Minimization | 처리 후 불필요한 컨텍스트 제거 | 후속 단계의 injection 방지 | 필수 컨텍스트까지 제거될 위험 |

위 6가지 디자인 패턴 외에, **Human-in-the-loop**(민감 행동 전 사용자 확인)은 일반 원칙(general best practice)으로 별도 적용된다. 최종 방어선이지만 user fatigue와 공격 난독화에 취약.

### 3.3 프레임워크/원칙 (Frameworks)

| 프레임워크 | 출처 | 핵심 원칙 |
|-----------|------|----------|
| Lethal Trifecta | Simon Willison (2025.06) | Private data + untrusted input + external communication → 셋 중 하나를 제거 |
| Agents Rule of Two | Meta AI (2025.10) | 세 속성 중 최대 두 가지만 허용, 셋 다 필요시 autonomous operation 금지 |
| Blast Radius 제한 | Willison (2023.12) | 완전 방어 불가 → blast radius 최소화에 집중 |

---

## 4. 핵심 인사이트

### 인사이트 1: "더 많은 AI"는 답이 아니다

**"Probabilistic defense를 겹겹이 쌓아도 adaptive attacker 앞에서는 무너진다."**

![In application security, 99% is a failing grade](../images/99-percent-failing-grade.jpeg)
*출처: [Simon Willison's talk](https://simonwillison.net/2023/May/2/prompt-injection-explained/)*

시리즈 전체를 관통하는 핵심 철학. AI 기반 방어(필터, 분류기, adversarial training 등)는 본질적으로 probabilistic하며, security에서 probabilistic은 곧 exploitable하다 — 99%의 탐지율도 adaptive attacker에게는 충분한 공격 여지를 남긴다. 2022년 9월 이 원칙을 선언한 이후, 3년간 다양한 AI 기반 방어가 시도되었으나 "The Attacker Moves Second" (2025.10)에서 12개 방어 기법이 adaptive attack에 대부분 90% 이상의 ASR로 무력화되며 실증적으로 확인되었다. CaMeL이 주목받는 이유도 여기에 있다: AI 훈련이 아닌 시스템 설계(capabilities, data flow analysis, taint tracking)로 보안을 확보하려 한 최초의 시도이기 때문이다. 다만 CaMeL 역시 완전한 해결은 아니며, 방향의 전환이 핵심이다.

### 인사이트 2: Security와 utility의 근본적 tradeoff

[Design Patterns for Securing LLM Agents against Prompt Injections](https://arxiv.org/abs/2506.08837) (IBM, ETH Zurich, Google, Microsoft 등 공저, 2025.06) 논문이 명시적으로 인정한 바: "이 패턴들은 에이전트의 임의 작업 수행 능력을 **의도적으로 제한**한다." 이 tradeoff는 양방향이다. 보안을 강화하면 에이전트가 할 수 있는 일이 줄어들고, utility를 포기하지 않으면 security guarantee가 불가능하다.

- CaMeL: 84% → 77% task success rate (7% 손실)
- Action Selector: 거의 완벽한 보안이지만 자유 텍스트 생성 불가
- 완전 자율 에이전트: 최대 utility, 그러나 security guarantee 불가

따라서 핵심 질문은 "어떤 에이전트를 만들 수 있느냐"가 아니라 "**어떤 제약 안에서 에이전트를 설계할 것인가**"다. 현재 시리즈와 관련 논문들이 수렴하는 답은 blast radius를 제한하고, 사전 계획 가능한 범위 내에서 동작하는 에이전트를 만드는 것이다.

### 인사이트 3: Static evaluation의 허상

"The Attacker Moves Second"의 가장 충격적인 발견: 원래 논문들이 거의 0%의 ASR을 보고했으나, adaptive attack 하에서는 90%+로 급등. 이는 학계의 방어 연구가 **공격자의 adaptation 능력을 과소평가**하고 있었음을 의미.

실용적 함의: 방어 논문이나 제품이 제시하는 benchmark 수치를 그대로 신뢰하면 안 된다. "adaptive attacker를 상정한 평가인가?"를 반드시 확인해야 하며, static benchmark에서의 높은 방어율은 실전 보안과 거의 무관하다.

### 인사이트 4: Lethal Trifecta — 실용적 위험 판단 도구

세 조건(private data, untrusted content, external communication) 중 **하나만 제거해도 최악의 결과(data exfiltration)를 방지할 수 있다** — 이것이 이 프레임워크의 핵심 가치다. 추상적인 "prompt injection은 위험하다"를 구체적이고 행동 가능한 체크리스트로 전환해 준다.

- GitHub MCP, Microsoft 365 Copilot, Google Gemini Enterprise 등 실제 사례에서 반복 검증

다만 Lethal Trifecta는 data exfiltration만 커버하며, state 변경 공격(파일 삭제, 설정 수정, 코드 실행 등)은 포착하지 못한다. Meta AI의 Agents Rule of Two가 이 한계를 보완했다 (자세한 내용은 2.Phase 4 참조).

### 인사이트 5: Lethal Trifecta의 실제 — 피할 수 없는 경우와 우발적으로 완성되는 경우

**구조적 불가피 — 브라우저 에이전트.** 브라우저 에이전트는 기능 자체가 세 조건을 모두 필연적으로 충족한다: 사용자 credentials/쿠키(private data), 방문하는 모든 웹페이지(untrusted input), HTTP 요청 자체(exfiltration 벡터). 하나를 제거하면 브라우저 에이전트의 존재 이유가 사라진다. Willison의 반복적 회의론(시리즈 외 [Perplexity Comet 분석](https://simonwillison.net/2025/Aug/25/agentic-browser-security/)에서):

> "LLM 기반 브라우저 확장의 전체 개념 자체가 치명적으로 결함이 있으며 안전하게 만들 수 없다고 강하게 의심한다."

**우발적 조합 — MCP.** 각 MCP 서버는 단독으로는 세 조건을 모두 충족하지 않을 수 있지만, 여러 서버를 조합하는 순간 trifecta가 완성된다. GitHub MCP는 하나의 패키지 안에 세 조건을 모두 충족했고, Supabase MCP도 마찬가지였다. 더 심각한 문제는 이 조합의 위험성을 판단하는 책임이 **보안 전문가가 아닌 최종 사용자에게 전가**되는 구조라는 점이다. 일반 사용자가 "이 두 MCP를 동시에 쓰면 위험한가?"를 판단하기는 현실적으로 불가능하다.

한쪽은 **"안전하게 만들 수 없다"**, 다른 쪽은 **"모르고 위험하게 만들게 된다"** — 둘 다 같은 근본 문제(LLM이 신뢰/비신뢰 입력을 구분하지 못함)의 다른 표현이다.

### 인사이트 6: 해결은 없지만, 프레이밍의 진전은 있다

[Bruce Schneier의 인용](https://www.schneier.com/blog/archives/2025/08/we-are-still-unable-to-secure-llms-from-malicious-inputs.html) (2025.08):

> "우리는 이 공격에 대한 방어 방법을 모른다. 이 공격에 안전한 에이전트 AI 시스템은 단 하나도 없다. 적대적 환경에서 작동하는 — 즉 신뢰할 수 없는 훈련 데이터나 입력을 접할 수 있는 — 모든 AI는 prompt injection에 취약하다. 이것은 존재론적 문제이며, 내가 아는 한 이 기술을 개발하는 대부분의 사람들이 그저 문제가 없는 척하고 있다."

근본적 해결은 여전히 없다. 그러나 3년 전에는 문제의 정의조차 불분명했다. 지금은 Lethal Trifecta로 위험 조건을 식별할 수 있고, Design Patterns로 제약 범위를 설계할 수 있으며, CaMeL로 control flow와 data flow를 분리하는 방법론이 생겼다. **"무엇을 포기해야 하는가"가 명확해진 것** — 이것이 3년간의 진전이다. 해결의 진전이 아닌, 프레이밍의 진전.

---

## 5. 현재 기술 수준 요약 (2025.11 시리즈 마지막 글 기준)

| 항목 | 상태 |
|------|------|
| Prompt injection 완전 방어 | ❌ 미해결 |
| Control flow(plan) 보호 | ⚠️ 부분적 (CaMeL, Plan-then-Execute) |
| Data flow(values) 보호 | ⚠️ 부분적 (taint tracking + 사용자 확인) |
| Probabilistic defense의 신뢰성 | ❌ Adaptive attack에 대부분 무력화 |
| 복잡한 동적 에이전트 태스크 | ❌ 미해결 — 사전 계획 가능한 태스크만 보호 |
| 실용적 위험 완화 | ✅ Lethal Trifecta / Rule of Two로 위험 요소 제거 가능 |
| 프로덕션 서비스의 실제 보안 | ❌ 주요 서비스에서 동일 패턴 공격이 계속 성공 중 |

---

## 6. 추가 분석 노트

> 아래는 시리즈 정리 과정에서 도출한 개인적인 분석이다. Willison의 직접적인 주장이 아니라, 시리즈와 관련 논문들을 교차 분석하며 얻은 인사이트들이다.

### Dual LLM의 보호 범위는 "plan vs data"로 구분할 수 있다

Dual LLM은 어떤 도구를 어떤 순서로 호출할지(plan)는 보호하지만, 도구에 전달되는 값(data)은 보호하지 못한다. 다만 이 이분법은 완전하지 않다. 조건문 분기처럼 데이터가 실행 경로를 결정하는 구간에서는 data가 곧 plan이 되어 경계가 모호해지며, 이 모호한 지대가 추가적인 attack surface를 만든다.

### Over-tainting의 진짜 위험은 false positive가 아니라 user fatigue다

CaMeL의 포괄적인 taint tracking은 보안 공학에서 의도된 보수적 설계이며, under-tainting보다 안전한 선택이다. 하지만 확인 요청이 과도해지면 사용자가 습관적으로 "전부 허용"을 누르게 되고, 이 시점에서 human-in-the-loop은 UX 비용이 아닌 보안 실패 경로로 전환된다. 보수적 설계가 역설적으로 방어를 무력화하는 경우다.

### 현재 아키텍처적 방어가 무너지는 지점은 "복잡한 태스크" 전반이 아니라, 중간 결과에 따라 다음 행동이 달라지는 태스크다

CaMeL 등의 방어는 사전 계획 가능한 태스크를 전제한다. 정형화된 비즈니스 워크플로우는 조건문과 루프로 충분히 분해 가능하므로 이 전제에 부합한다. 전제가 무너지는 건 검색 결과를 보고 다음 검색어를 정하거나, 조회 결과에 따라 추가 도구 호출을 결정하는 식의 동적 분기가 필요한 태스크이며, 방어 연구의 사각지대는 "복잡한 태스크" 전반이 아니라 이 특정 유형에 집중되어 있다.

### 검증 레이어는 미묘한 판단에 도움이 되지만, 입력을 구조화하지 않으면 같은 취약점에 노출된다

"사용자 의도와 부합하는가?"를 판단하는 검증 레이어는 taint tracking이 잡지 못하는 의미론적 판단(예: 이 이메일 주소가 맥락상 맞는 사람인가?)에 도움이 될 수 있다. 하지만 검증 레이어가 원본 악성 콘텐츠를 그대로 읽으면 같은 프롬프트 인젝션에 노출된다. 따라서 검증 대상을 원본이 아닌 추출된 값으로 한정하고, 입력을 구조화하여 attack surface를 줄여야 한다.

---

## 7. 참고 자료

### Simon Willison 시리즈

- [Prompt Injection Series](https://simonwillison.net/series/prompt-injection/) (2022.09 ~ 2025.11, 총 23편)
- [Lethal Trifecta 태그](https://simonwillison.net/tags/lethal-trifecta/)

### 주요 논문

- [Defeating Prompt Injections by Design (CaMeL)](https://arxiv.org/abs/2503.18813) — Google DeepMind, 2025.03
- [Design Patterns for Securing LLM Agents](https://arxiv.org/abs/2506.08837) — IBM, ETH Zurich, Google, Microsoft 등, 2025.06
- [The Attacker Moves Second](https://arxiv.org/abs/2510.09023) — OpenAI, Anthropic, Google DeepMind, ETH Zurich 등, 2025.10
- [Agents Rule of Two](https://ai.meta.com/blog/practical-ai-agent-security/) — Meta AI Blog, 2025.10
- [TaskTracker: Get My Drift?](https://arxiv.org/abs/2406.00799) — Microsoft, 2024.06

### 기타

- [Martin Fowler: Agentic AI and Security](https://martinfowler.com/articles/agentic-ai-security.html)
- [Johann Rehberger: The Month of AI Bugs](https://embracethered.com/blog/)
- [Google: An Introduction to Google's Approach for Secure AI Agents](https://research.google/pubs/an-introduction-to-googles-approach-for-secure-ai-agents/)
