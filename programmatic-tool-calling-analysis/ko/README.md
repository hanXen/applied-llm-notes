# Programmatic Tool Calling vs 기존 Tool Calling: 결과는 누가 받는가

> Tool Calling의 비용은 "결과가 어디로 가느냐"에 있다

---

## 1. 배경: Tool Calling의 발전

### 1.1 기존 Tool Calling

LLM이 tool을 사용하는 가장 기본적인 패턴이다:

```text
User → LLM → {"name": "query_db", "input": {"sql": "..."}}
                                    ↓
                              Tool Execution
                                    ↓
                                   결과 → LLM → 최종 응답
```

LLM이 JSON 형태의 인자를 생성하면, 개발자의 서버에서 tool이 실행되고, 그 결과가 다시 LLM의 context에 들어간다.

### 1.2 Parallel Tool Calling

여러 tool을 병렬로 호출하여 **round-trip 횟수**를 줄인다:

```text
User → LLM → [tool_use A, tool_use B, tool_use C]  (한 번에 병렬 호출)
                        ↓
             [Tool Execution A, B, C]
                        ↓
             [result A, result B, result C] → LLM → 최종 응답
```

round-trip은 줄었지만, **모든 결과가 LLM context에 들어가는 구조는 동일하다**.

### 1.3 CodeAct (Wang et al., 2024, ICML)

기존 LLM Agent는 JSON이나 정해진 텍스트 포맷으로 action을 생성한다. CodeAct은 이 방식의 근본적 한계(constrained action space & restricted flexibility)를 **executable Python code를 unified action space로 사용하는** 것으로 해결했다:

```text
기존:     LLM → {"tool": "search", "query": "..."}     (JSON action)
CodeAct:  LLM → result = search("..."); print(len(result))  (Code action)
```

17개 LLM 대상 실험에서 CodeAct은 기존 JSON/Text 방식보다 높은 성공률을 보였다. 가장 성능이 좋았던 GPT-4-1106-preview의 경우 Text 대비 **약 20%p 높은 성공률**(74.4% vs 53.7%)을 달성했다. 이 성능 향상이 어디서 오는지는 섹션 6에서 자세히 분석한다.

### 1.4 Programmatic Tool Calling (PTC)

Anthropic은 CodeAct의 개념을 담은 PTC를 Claude에 도입하였다.

```text
User → LLM → code_execution("""
    result = await query_db(...)
    total = sum(r['revenue'] for r in result)
    print(f"총 매출: {total}")
""")
```

기본적인 아키텍처는 CodeAct와 같다. 코드를 생성하고, Python runtime에서 실행하고, 코드 실행 결과(stdout/stderr)를 LLM에 반환한다.

PTC가 제품으로서 추가한 것은 `allowed_callers`(tool 호출 권한 제어), container lifecycle 관리, API 설계 같은 **엔지니어링 요소**다. CodeAct를 제품화한 것으로 볼 수 있다.

---

## 2. 핵심 질문: PTC가 정말 유리한가?

PTC의 장점으로 흔히 언급되는 것들이 있다:

- for문으로 반복 호출 → 토큰 절약
- round-trip 감소 → 지연시간 감소

그런데 여기서 의문이 생긴다:

> "복잡한 tool의 경우, 기존 tool calling은 인자만 넘기면 되는데, PTC는 매번 호출 코드를 생성해야 하지 않나? 안정성도 떨어지지 않나?"

이 의문은 타당하다. 그리고 이 의문을 해소하려면 **tool result가 어디로 가는지**를 정확히 이해해야 한다.

---

## 3. 결과는 누가 받는가

### 3.1 흔한 오해

PTC에서 Claude가 생성하는 것은 **tool의 구현 코드가 아니라 호출 코드**다. tool의 실제 구현은 여전히 개발자 서버에서 deterministic하게 실행된다. 이 점은 기존 tool calling과 동일하다.

```text
기존:  LLM → {"sql": "SELECT ..."}  → 서버 실행 → 결과
PTC:   LLM → result = await query_db("SELECT ...") → 서버 실행 → 결과
```

**tool 실행까지는 동일하다. 차이는 그 이후에 있다.**

### 3.2 결정적 차이: 결과의 수신자

| | 기존 Tool Calling | PTC |
|---|---|---|
| **tool result를 받는 주체** | LLM (추론 엔진) | Python runtime (container) |
| **LLM이 보는 것** | raw 결과 전체 | 코드 실행 결과(stdout/stderr)만 |
| **결과의 처리** | LLM이 토큰 단위로 읽고 추론 | Python이 변수로 처리 |

이것을 그림으로 표현하면:

**기존 Tool Calling:**

```text
func1(a) 실행 → 결과 10,000행 ──────────────► LLM context window
                                               (10,000행을 토큰으로 읽음)
```

**PTC:**

```text
func1(a) 실행 → 결과 10,000행 → Python 변수 (result)
                                      │
                                      ▼
                                Python이 처리
                                (sort, filter, sum...)
                                      │
                                      ▼
                          코드 실행 결과: "총 매출: $2.95M"
                                      │
                                      ▼
                              LLM context window
                              (처리된 결과만 읽음)
```

> Anthropic 공식 문서: "Tool results from programmatic invocations do not count toward your input/output token usage. Only the final code execution result and Claude's response count."

---

## 4. Parallel Tool Calling과의 비교

"그러면 Parallel Tool Calling이랑 뭐가 다른가?"라는 의문이 자연스럽게 따라온다. 둘 다 round-trip을 줄이는데, 핵심 차이는 **결과가 context에 누적되는 방식**이다.

### 4.1 Parallel Tool Calling의 흐름

```text
LLM: [tool_use A, tool_use B, tool_use C]
       ↓            ↓            ↓
  서버 실행A    서버 실행B    서버 실행C
       ↓            ↓            ↓
  result A      result B     result C
       └────────────┴────────────┘
                    ↓
            전부 LLM context에 들어감 ✅
                    ↓
              LLM: 최종 응답
```

### 4.2 PTC의 흐름

```text
LLM: code_execution("""
    result_a = await tool_a(...)
    result_b = await tool_b(...)
    result_c = await tool_c(...)
    print(summary)
""")
       ↓            ↓            ↓
  tool 실행A    tool 실행B    tool 실행C
       ↓            ↓            ↓
  Python 변수   Python 변수  Python 변수   ← LLM은 이걸 못 봄 ❌
       └────────────┴────────────┘
                    ↓
          Python이 처리 후 print()
                    ↓
        코드 실행 결과만 LLM context에 들어감 ✅
                    ↓
              LLM: 최종 응답
```

실제 실행 메커니즘은 위 그림보다 복잡하다. 각 `await`에서 코드 실행이 일시 중지되고, API가 개발자에게 `tool_use` 블록을 반환한다. 개발자가 tool을 실행하여 결과를 API에 돌려보내면 코드가 재개된다. 하지만 핵심은 동일하다: **tool result는 Python 변수로 들어가고, LLM은 최종 코드 실행 결과만 본다.**

### 4.3 구체적 비용 비교

DB에서 10,000행을 반환하는 tool을 3개 지역에 대해 호출하는 경우:

| | 기존 (Sequential) | Parallel | PTC |
|---|---|---|---|
| round-trip | 3회 | 1회 | 1회 |
| LLM context에 들어가는 데이터 | 30,000행 (누적) | 30,000행 | 코드 실행 결과만 |
| 중간 재추론 필요 | 매 호출마다 | 없음 | 없음 |
| input 토큰 비용 | 매우 큼 | 큼 | 작음 |

**→ Parallel과 PTC의 결정적 차이는 round-trip이 아니라, tool result가 LLM context에 들어가느냐 여부다.**

---

## 5. "Tool을 정교하게 설계하면 되지 않나?"

여기서 또 하나의 합리적인 의문이 생긴다:

> "tool이 처음부터 필터링된 결과만 반환하도록 설계하면, PTC 없이도 같은 효과 아닌가?"

맞다. tool의 response를 정교하게 설계하면 기존 방식으로도 context 부담을 줄일 수 있다. 하지만 현실적 한계가 있다:

### 5.1 Tool은 범용적으로 설계된다

```text
query_db("SELECT * FROM sales WHERE region = 'West'")
```

이 tool이 반환하는 결과를 어떻게 가공할지는 **호출할 때마다 다르다**:

| 상황 | 필요한 가공 |
|------|------------|
| "총 매출 알려줘" | `sum(revenue)` |
| "상위 5개 고객" | `sorted(...)[:5]` |
| "월별 추이" | `groupby(month)` |
| "이상치 탐지" | `[r for r in result if r['revenue'] > threshold]` |

Tool 자체에 이 모든 가공 로직을 넣으면 tool의 parameter가 복잡해지고, 새로운 요구사항이 나올 때마다 tool을 수정해야 한다.

### 5.2 PTC의 가치: LLM이 가공 로직을 유연하게 생성

PTC에서는 **"어떻게 가공할지"를 LLM이 매번 코드로 결정**한다:

```python
# LLM이 상황에 맞게 생성하는 코드
result = await query_db("SELECT * FROM sales WHERE region = 'West'")

# 총 매출이 필요한 경우:
total = sum(r['revenue'] for r in result)
print(f"총 매출: ${total:,.0f}")
```

```python
# 상위 5개가 필요한 경우:
result = await query_db("SELECT * FROM sales WHERE region = 'West'")
top5 = sorted(result, key=lambda x: x['revenue'], reverse=True)[:5]
for r in top5:
    print(f"{r['customer']}: ${r['revenue']:,.0f}")
```

**→ Tool은 범용으로 두고, 가공은 LLM이 유연하게 코드로 처리한다.**

---

## 6. 성능 향상의 핵심은 무엇인가?

CodeAct 논문은 코드 action의 장점으로 네 가지를 제시한다:

| 장점 | 설명 |
|------|------|
| **(1) Dynamic adjustment** | Python 인터프리터와의 multi-turn 상호작용을 통해 실행 결과를 관찰하고, 이전 action을 수정하거나 새 action을 생성 |
| **(2) Existing software packages** | PyPI의 기존 패키지를 직접 활용하여 action space를 확장 |
| **(3) Code familiarity** | LLM이 사전학습에서 대량의 코드를 접했기 때문에 코드 생성에 친숙 |
| **(4) Control and data flow** | 루프, 조건문, 변수 저장 등을 통해 여러 tool을 조합하고 중간 결과를 재사용 |

### 6.1 논문의 실험 구조

논문은 두 가지 실험으로 이 장점들의 기여를 확인한다.

**§2.2 (API-Bank)** — atomic tool use(단일 tool 호출)만 테스트한다. 논문의 1차 목적은 RQ1, 즉 "코드 친숙도(장점 3)가 도움이 되는가?"를 검증하는 것이다. 단일 호출이기 때문에 control and data flow(장점 4)의 효과는 자연스럽게 제거된다. 논문 Introduction에서 이를 다음과 같이 설명한다:

> "To demonstrate benefit (3), our first experiment (§2.2) compares CodeAct to baselines on basic tasks involving atomic tool use (i.e., only one tool is used per action), ablating the control and data flow advantage offered by CodeAct."

결과: 논문에 따르면 **comparable** 수준이었다. 코드 친숙도만으로는 유의미한 차이를 만들지 못한다는 것을 보여준다.

**§2.3 (M3ToolEval)** — 복잡한 multi-tool 조합 태스크를 테스트한다. Control and data flow(장점 4)가 포함된다. 결과: GPT-4-1106-preview 기준 Text 대비 약 20%p 높은 성공률, 평균 2.2턴 적은 상호작용.

두 실험을 대조하면:

| 실험 | control and data flow | 결과 |
|------|----------------------|------|
| §2.2 (API-Bank) | 제거됨 (단일 호출) | comparable |
| §2.3 (M3ToolEval) | 포함됨 (multi-tool) | 큰 폭의 성능 향상 |

§2.2에서 §2.3으로 갈 때 추가되는 것이 control and data flow이고, 이때 성능 차이가 크게 벌어진다. **논문은 이를 통해 control and data flow가 CodeAct 성능 향상의 핵심 요인이라는 것을 보여준다.**

### 6.2 context 효율성 관점

> 여기서부터는 논문의 주장이 아니라 **나의 해석**이다.

논문은 control and data flow의 이점을 "여러 tool을 조합하고 중간 결과를 변수에 저장하여 재사용하는 능력"으로 설명한다. 논문에 "context efficiency"나 "context engineering" 같은 용어는 등장하지 않는다.

하지만 이 메커니즘을 context 관점에서 다시 보면:

```text
control flow (루프, 조건문):
  JSON:    5번 왕복 → 매 턴마다 이전 결과가 context에 누적
  CodeAct: for문으로 1턴에 처리 → 누적 자체가 없음

data flow (변수 저장, 재사용):
  JSON:    tool result 전체가 LLM context에 들어감
  CodeAct: 변수에 저장, 코드 실행 결과만 전달
```

**결국 control and data flow가 하는 일은 LLM context에 들어가는 정보를 줄이는 것이다.**  다만 2024년 초에는 "context engineering"이라는 관점이 지금처럼 보편화되지 않았기 때문에, 논문에서 context라는 단어를 찾기 어렵다. 개념은 있었지만 그것을 부르는 이름이 아직 없었던 시기였다.

---

## 7. (내 생각) 트레이드오프와 시사점

### 7.1 PTC가 유리한 경우

| 조건 | 이유 |
|------|------|
| **다중 tool 호출 (3개+)** | round-trip 제거 + context 누적 방지 |
| **독립적 병렬 작업이 많은 경우** | 50개 endpoint 체크 등, 수가 많으면 round-trip 차이가 큼 |
| **tool result가 큰 경우** | 대량 데이터를 코드로 필터링 후 요약만 전달 |
| **조건부 로직** | 중간 결과에 따라 다음 호출 결정 |
| **early termination** | 조건 충족 시 즉시 중단 |

### 7.2 기존 Tool Calling이 유리한 경우

| 조건 | 이유 |
|------|------|
| **단일 tool 호출** | 코드 실행 환경 오버헤드만 추가됨 |
| **tool result가 작고 간단** | context 절약 효과 미미 |
| **즉각적 사용자 피드백 필요** | PTC는 코드 완료 후에야 응답 |
| **코드 생성 능력이 낮은 모델** | 코드 품질이 안정성에 직결됨 |

참고로, 현재 대부분의 open/closed 모델은 각 provider의 tool calling 포맷에 맞게 fine-tune되어 있다. 앞으로 PTC 방식에 최적화된 모델이 등장할 가능성은 있지만, 현 시점에서는 이 점도 선택 시 고려할 부분이다. 특히 파라미터가 작은 모델의 경우, PTC 방식이 기존 tool calling에 비해 안정성이 떨어질 수 있다.

### 7.3 안정성에 대한 고려

"생성된 코드는 JSON 인자보다 실패 가능성이 높지 않을까?"라는 우려가 있을 수 있다:

```text
기존: {"sql": "SELECT ..."}          ← 구조가 단순, 실패 적음
PTC:  result = await query_db(...)    ← 코드 전체가 올바라야 함
      filtered = [r for r in result if ...]
      print(summary)
```

하지만 PTC에서 생성되는 코드는 대부분 for문, 리스트 컴프리헨션, 조건문 같은 단순한 Python 패턴이고, 에러가 나면 stderr로 반환되어 재시도가 가능하다. 모델이 점차 발전할수록, 코드 안정성이 큰 장벽이 되기는 어려울 것으로 보인다.

### 7.4 ACE-FCA와의 공통점

[이전 글](../../ace-fca-agentflow-analysis/ko/README.md)에서 분석한 ACE-FCA도 동일한 원리를 따른다:

```text
ACE-FCA:      [Research 전체 과정] → Bottleneck → research.md (압축된 결과)
CodeAct/PTC:  [Tool result 전체]   → Bottleneck → 코드 실행 결과 (압축된 결과)
```

**정보 흐름에 bottleneck을 만들어 noise는 버리고 signal만 전달한다.** 적용 레벨은 다르지만 원리는 같다:

| | ACE-FCA | CodeAct/PTC |
|---|---|---|
| **무엇을 압축** | Phase의 전체 context | Tool result |
| **어디서 압축** | Sub-agent → artifact | Python runtime → 코드 실행 결과 |
| **누가 받음** | 다음 Phase의 LLM | 같은 턴의 LLM |
| **레벨** | Macro (프로젝트) | Micro (단일 턴) |

이 접근법들을 관통하는 인사이트는 다음과 같다:

> **"LLM의 context window는 비싼 자원이다. 거기에 들어가는 정보를 최소화하라."**

---

## 8. 결론

```text
기존 Tool Calling:
  Tool result → LLM이 직접 읽음
  → 결과가 크면 context 비용 급증

Parallel Tool Calling:
  Tool result → LLM이 직접 읽음 (round-trip만 줄임)
  → context 비용 문제는 동일

CodeAct/PTC:
  Tool result → Python이 받아서 처리 → 코드 실행 결과만 LLM에 전달
  → context 비용 대폭 절감
```

> PTC의 본질은 **"tool result의 수신자를 LLM에서 Python runtime으로 바꾸는 것이다"**.

| 상황 | 권장 방식 |
|------|----------|
| 단순 단일 호출, 결과 작음 | 기존 Tool Calling |
| 독립적 다중 호출, 수가 많음 | **PTC** |
| 다중 호출 + 결과 큼 + 가공 필요 | **PTC** |
| 조건부 로직 / 루프 / early exit | **PTC** |

---

## 참고 자료

- [CodeAct: Executable Code Actions Elicit Better LLM Agents (ICML 2024)](https://arxiv.org/abs/2402.01030)
- [CodeAct GitHub Repository](https://github.com/xingyaoww/code-act)
- [Anthropic: Programmatic Tool Calling](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling)
- [Anthropic: Code Execution Tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool)
- [Anthropic: Tool Use Overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
