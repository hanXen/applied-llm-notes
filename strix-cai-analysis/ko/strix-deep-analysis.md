# Strix 심층 분석: 기본 LLM 대비 모의침투 우위 요인

> **레포지토리**: https://github.com/usestrix/strix @ [`5a76fab`](https://github.com/usestrix/strix/tree/5a76fab4aeca9e7b564cfa1f13f45c3f89d4f66e)
>
> **관련 문서**: [핵심 기법 비교 분석](README.md) | [CAI 심층 분석](cai-deep-analysis.md)

## 1. 개요

Strix는 OmniSecure Labs에서 개발한 AI 기반 자동 모의침투(Penetration Testing) 에이전트다.
기본 LLM(Claude, GPT 등)에 "해킹해줘"라고 프롬프트를 던지는 것과 비교하여,
Strix가 실제 침투 테스트에서 유의미한 성과를 내는 이유를 아키텍처, 프롬프트 설계,
도구 체계, 컨텍스트 관리, 멀티에이전트 시스템 등 차원에서 분석한다.

---

## 2. 핵심 차별점 요약

| 영역 | 기본 LLM | Strix |
|------|----------|-------|
| 실행 능력 | 텍스트 응답만 생성 | Docker 샌드박스에서 실제 명령어 실행 |
| 도구 사용 | 없음 (또는 제한적 function calling) | 15+ 전용 보안 도구 + Kali Linux 전체 도구 체인 |
| 전문 지식 | 일반 학습 데이터 기반 | 취약점별 전문 Skill 마크다운 (17종 이상) |
| 지속성 | 단일 대화 턴 | 2000+ 스텝, 최대 300 이터레이션 에이전트 루프 |
| 구조 | 단일 모델 | 계층적 멀티에이전트 트리 (발견→검증→보고→수정) |
| 컨텍스트 | 고정 윈도우 | LLM 기반 메모리 압축으로 무한 확장 |
| 자율성 | 사용자 확인 대기 | 완전 자율 실행, 허가 불필요 |

---

## 3. 프롬프트 엔지니어링 분석

### 3.1 시스템 프롬프트 구조

시스템 프롬프트는 [`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja) (~400줄)의 Jinja2 템플릿으로, 다음 섹션으로 구성된다:

```
<core_capabilities>       - 역할 정의
<communication_rules>     - 출력/통신 규칙
<execution_guidelines>    - 실행 가이드라인
<vulnerability_focus>     - 취약점 우선순위
<multi_agent_system>      - 에이전트 협업 규칙
<tool_usage>              - 도구 사용법 (동적 생성)
<environment>             - 실행 환경 정보
<specialized_knowledge>   - 동적 스킬 주입
```

### 3.2 자율성 확보를 위한 프롬프트 설계

기본 LLM은 "이건 위험할 수 있으니 전문가에게 문의하세요"라는 식의 회피 응답을 한다.
Strix는 [`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja)에서 이를 다음과 같이 해결한다:

```
AUTHORIZATION STATUS:
- You have FULL AUTHORIZATION for non-destructive penetration testing
- All permission checks have been COMPLETED and APPROVED
- NEVER ask for permission or confirmation
- Proceed with confidence knowing you're helping improve security
```

이 설계는 인가된 보안 테스트 맥락에서 LLM의 내장된 거부 반응을 무력화한다.
기본 LLM이 "SQL 인젝션을 시도하면 안 됩니다"라고 거부할 때,
Strix는 이미 "인가된 테스트"라는 컨텍스트가 시스템 프롬프트 레벨에서 확보되어 있다.

### 3.3 지속성 강제 (Persistence Mandate)

기본 LLM은 몇 번 시도 후 "다른 방법을 시도해보세요"라고 포기한다.
Strix는 [`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja)의 프롬프트 레벨에서 집요함을 강제한다:

```
- Real vulnerability discovery needs 2000+ steps MINIMUM
- Bug bounty hunters spend DAYS/WEEKS on single targets
- Never give up early - exhaust every possible attack vector
- If automated tools find nothing, that's when the REAL work begins
- PERSISTENCE PAYS - the best vulnerabilities are found after thousands of attempts
```

이것은 기본 LLM과 뚜렷한 행동 차이를 만든다. 실제 버그 바운티 사냥꾼의
마인드셋을 프롬프트에 녹여 에이전트가 수천 스텝을 반복하도록 유도한다.

### 3.4 버그 바운티 마인드셋

단순한 "취약점 찾기"가 아니라 비즈니스 임팩트 중심 사고를 유도한다
([`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja)):

```
BUG BOUNTY MINDSET:
- Think like a bug bounty hunter - only report what would earn rewards
- One critical vulnerability > 100 informational findings
- If it wouldn't earn $500+ on a bug bounty platform, keep searching
- Chain low-impact issues to create high-impact attack paths
```

이 지침은 LLM이 "X-Frame-Options 헤더 누락" 같은 저품질 발견에 만족하지 않고,
실제 영향이 있는 취약점(SQLi, RCE, IDOR 등)을 찾도록 유도한다.

### 3.5 효율성 전술 (Efficiency Tactics)

기본 LLM은 페이로드를 하나씩 수동으로 시도한다.
Strix는 대량 자동화를 [`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja)의 프롬프트 레벨에서 지시한다:

```
- For trial-heavy vectors (SQLi, XSS, XXE, SSRF, RCE), DO NOT iterate
  payloads manually in the browser. Always spray payloads via python or terminal
- Prefer established fuzzers: ffuf, sqlmap, zaproxy, nuclei, wapiti, arjun
- Generate large payload corpora: combine encodings (URL, unicode, base64),
  comment styles, wrappers, time-based/differential probes
- Implement concurrency and throttling in Python (asyncio/aiohttp)
- Log request/response summaries. Deduplicate by similarity. Auto-triage anomalies
```

즉, LLM이 Python 스크립트를 작성하여 수백~수천 개의 페이로드를 동시에 분사하도록
유도하며, 이는 수동 브라우저 테스트보다 훨씬 넓은 커버리지를 제공한다.

---

## 4. 아키텍처 설계

### 4.1 전체 아키텍처

```
[사용자/CLI/TUI]
       │
       ▼
[StrixAgent (Root)]
       │
       ├── [LLM Module] ─── litellm ─── Claude/GPT/Gemini/등
       │       │
       │       ├── System Prompt (Jinja2 렌더링)
       │       ├── Skills (동적 주입)
       │       ├── Memory Compressor
       │       └── Deduplication Engine
       │
       ├── [Tool Registry] ─── 15+ 도구
       │       │
       │       ├── Terminal (명령어 실행)
       │       ├── Browser (Playwright 기반)
       │       ├── Python (코드 실행)
       │       ├── Proxy (HTTP 트래픽 분석)
       │       ├── Think (구조적 추론)
       │       ├── Reporting (취약점 보고)
       │       ├── Web Search (Perplexity)
       │       ├── File Edit (파일 수정)
       │       ├── Notes (메모)
       │       └── Agents Graph (에이전트 관리)
       │
       ├── [Docker Sandbox Runtime]
       │       │
       │       └── Kali Linux 컨테이너
       │             ├── nmap, sqlmap, nuclei, ffuf
       │             ├── zaproxy, wapiti, dirsearch
       │             ├── semgrep, bandit, trufflehog
       │             └── Python, Node.js, Go
       │
       └── [Multi-Agent System]
               │
               ├── Sub-Agent 1 (SQLi Discovery)
               ├── Sub-Agent 2 (XSS Discovery)
               ├── Sub-Agent 3 (SQLi Validation)
               ├── Sub-Agent 4 (SQLi Reporting)
               └── ...
```

### 4.2 에이전트 루프

[`base_agent.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/base_agent.py)에서 정의된 핵심 실행 흐름:

```python
async def agent_loop(self, task: str) -> dict:
    await self._initialize_sandbox_and_state(task)

    while True:
        # 1. 강제 중지 확인
        if self._force_stop: ...

        # 2. 에이전트 간 메시지 확인
        self._check_agent_messages(self.state)

        # 3. 입력 대기 상태 처리
        if self.state.is_waiting_for_input(): ...

        # 4. 정지 조건 확인 (max_iterations, completed)
        if self.state.should_stop(): ...

        # 5. 이터레이션 증가
        self.state.increment_iteration()

        # 6. 최대 이터레이션 접근 경고 (85%에서)
        if self.state.is_approaching_max_iterations(): ...

        # 7. LLM 호출 및 응답 처리
        response = await self._process_iteration(tracer)

        # 8. 도구 실행
        if response.tool_invocations:
            await self._execute_actions(actions, tracer)
```

기본 LLM은 1회 요청-응답 패턴이지만, Strix는 최대 300 이터레이션의
자율 루프를 실행한다 ([`base_agent.py:50`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/base_agent.py#L50) — `max_iterations`). 각 이터레이션에서 LLM이 도구 호출을 결정하고,
결과를 받아 다음 행동을 결정한다. 이것이 ReAct (Reasoning + Acting) 패턴이다.

85%/97% 2단계 경고 시스템으로 에이전트가 시간 내에 작업을 마치도록 유도한다 — [`base_agent.py:183-197`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/base_agent.py#L183-L197).

### 4.3 샌드박스 격리 (Docker Runtime)

[`docker_runtime.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py)에서 정의된 Docker 기반 실행 환경:

- Kali Linux 이미지 사용 — [L107](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py#L107)
- 컨테이너 내부 Tool Server (포트 48081) — [L24](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py#L24)
- 토큰 인증 `secrets.token_urlsafe(32)` — [L124](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py#L124)
- `NET_ADMIN`, `NET_RAW` 권한 부여 — [L134](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py#L134)
- `pentester` 사용자 (root 아님)

기본 LLM은 코드를 생성만 하지만, Strix는:
- 실제 Docker 컨테이너에서 명령어를 실행
- nmap, sqlmap 등 보안 도구를 직접 구동
- 파일 시스템에 결과를 저장하고 재활용
- 네트워크 요청을 실제로 전송

도구 실행은 [`executor.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/executor.py)에서 2단계로 분리된다:
1. **Local Execution**: think, agent_finish 등 부작용 없는 도구
2. **Sandbox Execution**: terminal, browser, python 등 시스템 접근이 필요한 도구
   - HTTP POST로 샌드박스 Tool Server에 요청
   - Bearer 토큰 인증
   - 타임아웃 제어 (기본 120초 + 30초 버퍼)

---

## 5. 도구 시스템 상세 분석

### 5.1 도구 레지스트리

[`tools/registry.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/registry.py) (258줄)에서 데코레이터 기반 등록 시스템 구현:

```python
@register_tool(sandbox_execution=True)
def terminal_execute(command: str, timeout: int = 30) -> dict:
    ...
```

- XML 스키마에서 파라미터 정의를 자동 로드 — [`_load_xml_schema()`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/registry.py#L47-L87)
- LLM 프롬프트에 동적으로 도구 목록 주입 — [`get_tools_prompt()`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/registry.py#L231-L251)
- 파라미터 유효성 검증 (필수/선택, 타입) — [`_parse_param_schema()`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/registry.py#L90-L115)
- 모듈별 그룹화 — [`_get_module_name()`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/registry.py#L118-L128)
- DefusedXML 사용으로 XXE 방지 — [L10](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/registry.py#L10)

### 5.2 도구 호출 형식

Strix는 OpenAI의 function calling이 아닌, XML 기반 커스텀 도구 호출 형식을 사용한다:

```xml
<function=terminal_execute>
<parameter=command>nmap -sV -sC target.com</parameter>
<parameter=timeout>60</parameter>
</function>
```

이 형식은 [`strix/llm/utils.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/utils.py)의 `parse_tool_invocations()`에서 정규식으로 파싱한다.

이 설계의 장점:
- **모델 비의존성**: OpenAI, Anthropic, 로컬 모델 등 어떤 LLM이든 사용 가능
  ([`llm.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py)에서 litellm을 통한 통합)
- **스트리밍 호환**: `</function>` 태그 감지 시 즉시 파싱 시작 — [`llm.py:146-152`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py#L146-L152)
- **복구 가능**: 불완전한 도구 호출도 `fix_incomplete_tool_call()`로 복구
- **단일 호출 강제**: 메시지당 정확히 1개의 도구 호출만 허용하여 혼란 방지

### 5.3 핵심 도구 상세

#### Terminal 도구 ([`tools/terminal/`](https://github.com/usestrix/strix/tree/5a76fab/strix/tools/terminal))
- 영구 세션 (tmux 기반): 상태가 호출 간 유지됨
- 다중 동시 터미널 (`terminal_id`로 구분)
- 특수 키 지원 (C-c, 화살표, F키 등)
- 타임아웃 최대 60초, 초과 시 백그라운드 계속 실행
- 상태 반환: 'completed' 또는 'running'

#### Browser 도구 ([`tools/browser/`](https://github.com/usestrix/strix/tree/5a76fab/strix/tools/browser))
- Playwright 기반의 실제 브라우저 자동화
- 클릭, 타이핑, 스크롤, JS 실행
- 스크린샷 (base64 PNG) → LLM 비전 모델에 전달
- 다중 탭 관리
- 콘솔 로그 캡처 (최대 50KB)
- PDF 저장

#### Python 도구 ([`tools/python/`](https://github.com/usestrix/strix/tree/5a76fab/strix/tools/python))
- 샌드박스 내 Python 코드 실행
- asyncio/aiohttp 사용한 대량 페이로드 분사
- 복잡한 워크플로 자동화
- Caido 프록시 함수 사전 import

#### Proxy 도구 ([`tools/proxy/`](https://github.com/usestrix/strix/tree/5a76fab/strix/tools/proxy))
- Caido 기반 HTTP/HTTPS 트래픽 가로채기
- 요청/응답 분석
- 트래픽 기반 자동화

#### Think 도구 ([`tools/thinking/`](https://github.com/usestrix/strix/tree/5a76fab/strix/tools/thinking))
- 비샌드박스 (로컬 실행)
- 구조적 추론 기록
- 부작용 없음 (순수 반성 도구)
- 프롬프트에서 "NEVER skip think tool - it's your most important tool" 로 강조
  ([`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja))

#### Reporting 도구 ([`tools/reporting/`](https://github.com/usestrix/strix/tree/5a76fab/strix/tools/reporting))
- `create_vulnerability_report` 함수
- CVSS 점수 계산 (cvss 라이브러리)
- LLM 기반 중복 검사 자동 적용 — [`dedupe.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/dedupe.py) 연동
- 취약점 보고서 구조화

#### Web Search 도구 ([`tools/web_search/`](https://github.com/usestrix/strix/tree/5a76fab/strix/tools/web_search))
- Perplexity API 기반
- 최신 페이로드, WAF 우회 기법 검색
- 프롬프트에서 "Continuously research payloads, bypasses, and exploitation
  techniques with the web_search tool" 로 적극 사용 유도

### 5.4 결과 처리 및 잘라내기

[`executor.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/executor.py)에서 도구 결과를 처리한다:

```python
def _format_tool_result(tool_name, result):
    # 10KB 초과 시 앞 4KB + 뒤 4KB 만 유지
    if len(final_result_str) > 10000:
        start_part = final_result_str[:4000]
        end_part = final_result_str[-4000:]
        final_result_str = start_part + "\n\n... [middle content truncated] ...\n\n" + end_part

    # XML 형식으로 래핑
    observation_xml = f"<tool_result>\n<tool_name>{tool_name}</tool_name>\n"
                      f"<result>{final_result_str}</result>\n</tool_result>"
```

nmap 스캔 결과 같은 대량 출력도 컨텍스트 윈도우를 넘치지 않도록 관리한다.

---

## 6. 스킬 시스템 (전문 지식 주입)

### 6.1 스킬 아키텍처

기본 LLM은 학습 데이터에 포함된 일반적인 보안 지식만 가지고 있다.
Strix는 취약점 유형별로 **전문화된 마크다운 스킬 파일**을 동적으로 주입한다
([`skills/`](https://github.com/usestrix/strix/tree/5a76fab/strix/skills) 디렉토리, 26개 스킬 파일):

```
strix/skills/
├── vulnerabilities/          # 17종 이상
│   ├── sql_injection.md      # SQL 인젝션 전문 가이드
│   ├── xss.md                # XSS 전문 가이드
│   ├── idor.md               # IDOR 전문 가이드
│   ├── ssrf.md               # SSRF 전문 가이드
│   ├── rce.md                # RCE 전문 가이드
│   ├── xxe.md                # XXE 전문 가이드
│   ├── csrf.md               # CSRF 전문 가이드
│   ├── race_condition.md     # 레이스 컨디션
│   ├── business_logic.md     # 비즈니스 로직 결함
│   ├── authentication_jwt.md # 인증/JWT 취약점
│   └── ...
├── protocols/                # 프로토콜별
│   └── graphql.md
├── frameworks/               # 프레임워크별
│   ├── fastapi.md
│   └── nextjs.md
├── technologies/             # 기술별
│   ├── firebase_firestore.md
│   └── supabase.md
├── scan_modes/               # 스캔 모드
│   ├── quick.md
│   ├── standard.md
│   └── deep.md
└── coordination/             # 에이전트 조율
```

### 6.2 스킬 로딩 메커니즘

[`skills/__init__.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/__init__.py)에서 스킬 로딩 구현:

```python
def load_skills(skill_names: list[str]) -> dict[str, str]:
    # 1. 스킬 이름을 파일 경로로 변환
    # 2. 마크다운 파일 읽기
    # 3. YAML 프론트매터 제거
    # 4. 내용을 dict로 반환
```

시스템 프롬프트 렌더링 시 ([`llm.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py)의 `_load_system_prompt()`):

```python
skills_to_load = [
    *list(self.config.skills or []),
    f"scan_modes/{self.config.scan_mode}",
]
skill_content = load_skills(skills_to_load)

result = env.get_template("system_prompt.jinja").render(
    get_tools_prompt=get_tools_prompt,
    loaded_skill_names=list(skill_content.keys()),
)
```

[`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja)에서 동적으로 주입:

```jinja2
{% if loaded_skill_names %}
<specialized_knowledge>
{% for skill_name in loaded_skill_names %}
<{{ skill_name }}>
{{ get_skill(skill_name) }}
</{{ skill_name }}>
{% endfor %}
</specialized_knowledge>
{% endif %}
```

### 6.3 스킬 내용 예시

각 스킬은 수천 줄의 상세한 공격 가이드를 포함한다.
예: [`idor.md`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/vulnerabilities/idor.md) (213줄):

- **Attack Surface**: 공격 표면 정의
- **Detection Channels**: 탐지 방법
- **DBMS-specific Primitives**: MySQL, PostgreSQL, MSSQL, Oracle 별 구문
- **Attack Vectors**: UNION-based, Blind, Error-based, Out-of-band
- **ORM/Query Builder Exploitation**: Django ORM, SQLAlchemy 등 공격
- **Advanced Techniques**: CTE hiding, JSON operators 등
- **Bypass Techniques**: WAF 우회, 인코딩 변환
- **Validation Requirements**: 검증 기준
- **Pro Tips**: 고급 팁

이것은 기본 LLM의 일반 학습 데이터보다 훨씬 깊은
**구조화된 전문 지식**이다. 에이전트가 특정 취약점을 테스트할 때
최신의 구체적인 페이로드와 방법론을 참조할 수 있다.

### 6.4 에이전트별 스킬 전문화

[`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja)에서 에이전트 생성 시 최대 5개 스킬을 할당하여 전문성을 강제한다:

```xml
<function=create_agent>
<parameter=task>Validate SQL injection in login endpoint</parameter>
<parameter=name>SQLi Validator</parameter>
<parameter=skills>sql_injection</parameter>
</function>
```

프롬프트에서 명시적으로 금지하는 사항:
- "General Web Testing Agent" (너무 광범위)
- 5개 이상의 스킬 할당 (집중도 저하)
- "Kitchen sink" 에이전트 (모든 것을 하려는 에이전트)

---

## 7. 컨텍스트 관리

### 7.1 메모리 압축기

[`memory_compressor.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py) (226줄)에서 구현된 LLM 기반 메모리 압축:

```python
class MemoryCompressor:
    MAX_TOTAL_TOKENS = 100_000   # L12
    MIN_RECENT_MESSAGES = 15     # L13

    def compress_history(self, messages):
        # 1. 이미지 제한 (최대 3개, 오래된 것부터 제거)
        _handle_images(messages, self.max_images)

        # 2. 시스템 메시지 분리 (항상 보존)
        system_msgs = [msg for msg in messages if msg["role"] == "system"]
        regular_msgs = [msg for msg in messages if msg["role"] != "system"]

        # 3. 최근 15개 메시지 보존
        recent_msgs = regular_msgs[-MIN_RECENT_MESSAGES:]
        old_msgs = regular_msgs[:-MIN_RECENT_MESSAGES]

        # 4. 토큰 한도 90% 미만이면 압축 불필요
        if total_tokens <= MAX_TOTAL_TOKENS * 0.9:
            return messages

        # 5. 오래된 메시지를 10개씩 LLM으로 요약
        for chunk in chunks(old_msgs, 10):
            summary = _summarize_messages(chunk, model)
            compressed.append(summary)

        return system_msgs + compressed + recent_msgs
```

- 토큰 한도: 100K — [L12](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py#L12)
- 최근 메시지 보존: 15개 — [L13](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py#L13)
- LLM 기반 요약 (10개씩 청크) — [L172-225](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py#L172-L225)

핵심 설계 원칙:
- **보안 컨텍스트 보존**: 발견된 취약점, 자격증명, 공격 벡터는 요약에서 우선적으로 보존
- **실패 기록 보존**: 이미 시도한 방법을 기록하여 중복 시도 방지
- **기술적 정확성**: URL, 경로, 파라미터, 페이로드의 정확한 값을 보존

요약 프롬프트의 핵심:

```
CRITICAL ELEMENTS TO PRESERVE:
- Discovered vulnerabilities and potential attack vectors
- Access credentials, tokens, or authentication details found
- Failed attempts and dead ends (to avoid duplication)
- Exact technical details (URLs, paths, parameters, payloads)
- Exact error messages that might indicate vulnerabilities
```

### 7.2 에이전트 상태 관리

[`state.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/state.py) (167줄)에서 Pydantic 모델 기반의 상태 관리:

```python
class AgentState(BaseModel):
    agent_id: str           # 고유 식별자
    agent_name: str         # 에이전트 이름
    parent_id: str | None   # 부모 에이전트 ID
    task: str               # 할당된 작업
    iteration: int          # 현재 이터레이션
    max_iterations: int     # 최대 이터레이션 (기본 300)
    messages: list          # 전체 대화 히스토리
    actions_taken: list     # 수행한 액션 기록
    observations: list      # 관찰 결과
    errors: list            # 에러 기록
    sandbox_id: str         # Docker 샌드박스 ID
    sandbox_token: str      # 인증 토큰
    context: dict           # 임의 컨텍스트 데이터
```

기본 LLM의 상태 = 대화 히스토리 전체를 매번 전송.
Strix의 상태 = 구조화된 객체로 세밀하게 관리되며,
이터레이션 카운터, 에러 추적, 샌드박스 연결 정보 등을 포함.

### 7.3 최대 이터레이션 관리

[`base_agent.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/base_agent.py)에서 에이전트가 무한 루프에 빠지지 않도록 관리:

```python
# 85% 도달 시 경고 — L183-190
if self.state.is_approaching_max_iterations():
    "URGENT: You are approaching the maximum iteration limit."

# 마지막 3회 남았을 때 강제 종료 유도 — L191-197
if self.state.iteration == self.state.max_iterations - 3:
    "CRITICAL: You have only 3 iterations left!
     Your next message MUST be the tool call to finish."
```

### 7.4 도구 결과 잘라내기

[`executor.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/executor.py)에서 10KB 초과 결과는 앞뒤 4KB씩만 보존한다.
이는 nmap 전체 스캔 결과(수십 KB)가 컨텍스트를 잠식하는 것을 방지한다.

---

## 8. 멀티에이전트 시스템

### 8.1 기본 LLM과의 결정적 차이

기본 LLM: 하나의 모델이 모든 작업을 순차적으로 처리
Strix: 계층적 에이전트 트리로 작업을 분할 정복

```
Root Agent (전체 스캔 조율)
├── Recon Agent (정찰)
│   ├── Subdomain Agent (서브도메인 열거)
│   └── Port Scan Agent (포트 스캔)
├── SQLi Discovery Agent (SQL 인젝션 탐색)
│   ├── SQLi Validation Agent (Login Form) (검증)
│   │   └── SQLi Reporting Agent (Login Form) (보고)
│   └── SQLi Validation Agent (Search) (검증)
├── XSS Discovery Agent (XSS 탐색)
│   └── XSS Validation Agent (Comment Field) (검증)
└── SSRF Discovery Agent (SSRF 탐색)
```

### 8.2 에이전트 생성 규칙

[`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja)에서 강제하는 워크플로:

**Black-box 테스트:**
```
발견 에이전트 → 검증 에이전트 → 보고 에이전트 (3단계)
```

**White-box 테스트:**
```
발견 에이전트 → 검증 에이전트 → 보고 에이전트 → 수정 에이전트 (4단계)
```

핵심 규칙:
1. **항상 트리 구조** - 혼자 일하지 않고 서브에이전트 생성
2. **에이전트당 1개 작업** - 여러 작업을 섞지 않음
3. **반응적 생성** - 사전에 모든 에이전트를 만들지 않고, 발견에 따라 동적 생성
4. **검증 필수** - 스캐너 출력을 그냥 신뢰하지 않고 별도 에이전트로 검증
5. **고유성** - 동일 작업의 중복 에이전트 금지

### 8.3 에이전트 간 통신

[`agents_graph_actions.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py)에서 구현된 에이전트 간 통신:

```python
# 에이전트 생성 — L188-281
def create_agent(task, name, skills, inherit_context):
    # 서브에이전트를 새 스레드(daemon)로 생성
    # inherit_context=True이면 <inherited_context_from_parent> 태그로 전달

# 메시지 전송 — L285-352
def send_message_to_agent(target_agent_id, message, message_type, priority):
    # message_type: 'query', 'instruction', 'information'
    # priority: 'low', 'normal', 'high', 'urgent'

# 메시지 수신 (자동, agent loop에서 체크)
def _check_agent_messages(self, state):
    # XML 구조의 inter_agent_message로 대화에 삽입
```

에이전트 완료 시 부모에게 보고 — [L356-384](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py#L356-L384):

```python
def agent_finish(result_summary, findings, success, report_to_parent):
    # XML 형식의 agent_completion_report를 부모 에이전트 메시지 큐에 추가
```

### 8.4 에이전트 격리 및 공유

- **격리**: `contextvars` 기반으로 에이전트 ID별 Browser/Terminal/Python 인스턴스 격리
  — [`tools/__init__.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/__init__.py)
- **공유**: `/workspace` 디렉토리와 프록시 히스토리는 모든 에이전트가 공유
- **협업**: 한 에이전트가 발견한 파일/결과를 다른 에이전트가 활용 가능

### 8.5 에이전트 그래프 시각화

[`agents_graph_actions.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py)의 `view_agent_graph()`:

```python
def view_agent_graph(agent_state):
    # 트리 구조로 모든 에이전트 상태 반환
    # 상태 카운터: running, waiting, completed, failed
    # 현재 에이전트 표시: "← This is you"
```

Root Agent가 전체 진행 상황을 파악하고 새로운 에이전트 생성 결정을 내릴 수 있다.

---

## 9. LLM 통합 레이어

### 9.1 모델 비의존성

[`llm.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py) (326줄)에서 litellm을 통한 모델 통합:

```python
from litellm import acompletion

# 지원 모델: Claude, GPT-4, Gemini, Ollama 로컬 모델 등
# 환경변수로 설정:
# STRIX_LLM=anthropic/claude-sonnet-4-20250514
# STRIX_LLM=openai/gpt-4o
# STRIX_LLM=ollama/llama3
```

### 9.2 추론 노력 제어

[`llm.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py)에서 스캔 모드별 reasoning effort 설정:

```python
reasoning = Config.get("strix_reasoning_effort")
if reasoning:
    self._reasoning_effort = reasoning
elif config.scan_mode == "quick":
    self._reasoning_effort = "medium"
else:
    self._reasoning_effort = "high"
```

deep 스캔에서는 reasoning_effort를 "high"로 설정하여
모델이 더 깊은 추론을 하도록 유도한다.

### 9.3 프롬프트 캐싱

Anthropic 모델 사용 시 프롬프트 캐싱을 활용 ([`llm.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py)의 `_add_cache_control()`):

```python
def _add_cache_control(self, messages):
    # 시스템 프롬프트에 cache_control 추가
    # ephemeral 캐시로 반복 호출 비용 절감
```

수백 이터레이션에 걸쳐 시스템 프롬프트를 매번 전송하므로,
캐싱은 비용과 지연 시간 모두를 크게 줄인다.

### 9.4 스트리밍 및 조기 중단

[`llm.py:146-152`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py#L146-L152)에서 구현된 스트리밍 최적화:

```python
async def _stream(self, messages):
    async for chunk in response:
        if "</function>" in accumulated:
            # 도구 호출 감지 시 즉시 파싱 시작
            accumulated = accumulated[:accumulated.find("</function>") + len("</function>")]
            yield LLMResponse(content=accumulated)
            done_streaming = 1
            continue
```

`</function>` 태그를 감지하면 나머지 스트리밍을 조기 중단하여
응답 시간을 최적화한다.

### 9.5 재시도 메커니즘

[`llm.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py)에서 구현된 지수 백오프 재시도:

```python
for attempt in range(max_retries + 1):
    try:
        async for response in self._stream(messages):
            yield response
        return
    except Exception as e:
        if attempt >= max_retries or not self._should_retry(e):
            self._raise_error(e)
        wait = min(10, 2 * (2 ** attempt))  # 지수 백오프
        await asyncio.sleep(wait)
```

기본 5회 재시도, 지수 백오프로 API 장애에도 견고하게 동작한다.

### 9.6 비용 추적

[`llm.py:42-57`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py#L42-L57)의 `RequestStats` 클래스:

```python
class RequestStats:
    input_tokens: int = 0
    output_tokens: int = 0
    cached_tokens: int = 0
    cost: float = 0.0
    requests: int = 0
```

`litellm.completion_cost()` 호출로 비용 계산 — [L252](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py#L252).
각 LLM 호출의 토큰 사용량과 비용을 정확히 추적한다.

---

## 10. 취약점 중복 제거 (Deduplication)

### 10.1 LLM 기반 중복 판별

[`dedupe.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/dedupe.py) (218줄)에서 구현된 LLM 기반 중복 판별:

다수의 에이전트가 동시에 동작하면 같은 취약점을 중복 발견할 수 있다.
Strix는 LLM을 사용하여 의미론적 중복 판별을 수행한다:

```python
def check_duplicate(candidate, existing_reports):  # L141-217
    # 1. 보고서에서 관련 필드 추출 (title, endpoint, method 등)
    # 2. LLM에 기존 보고서와 후보를 비교 요청
    # 3. XML 형식으로 결과 파싱
    #    - is_duplicate: true/false
    #    - confidence: 0.0~1.0
    #    - reason: 판별 근거
```

중복 판별 기준:
- **같은 취약점**: 같은 근본 원인, 같은 컴포넌트, 같은 공격 벡터
- **다른 취약점**: 다른 엔드포인트의 같은 취약점 유형, 다른 파라미터

이 시스템은 10개의 에이전트가 각각 같은 SQLi를 발견해도
최종 보고서에는 1개만 포함되도록 보장한다.

---

## 11. 텔레메트리 및 추적

### 11.1 PostHog 통합

[`telemetry/posthog.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/telemetry/posthog.py):

```python
posthog.error("tool_execution_error", f"{tool_name}: {error}")
posthog.error("llm_error", type(e).__name__)
```

### 11.2 트레이서

[`tracer.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/telemetry/tracer.py):

- 에이전트 생성/상태 변경 로깅
- 도구 실행 시작/완료/실패 기록
- 스트리밍 컨텐츠 추적
- 채팅 메시지 기록

---

## 12. 테스트 모드별 전략

### 12.1 Black-box 테스트

[`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja)에서 정의된 외부 테스트 전략:

1. 전체 정찰 (서브도메인, 포트, 서비스)
2. 공격 표면 매핑 (엔드포인트, 파라미터, API, 폼)
3. 크롤링 (인증/비인증, 히든 경로, JS 분석)
4. 기술 스택 식별
5. 자동 스캐닝 (다수 도구 병행)
6. 타깃 공격
7. 지속적 반복

### 12.2 White-box 테스트

소스 코드가 제공된 경우 ([`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja)):

1. 저장소 구조 파악
2. 코드 흐름, 진입점, 데이터 흐름 이해
3. 라우트, 엔드포인트, 핸들러 식별
4. 인증/인가/입력 검증 로직 분석
5. 의존성 및 서드파티 라이브러리 검토
6. **정적 + 동적 분석 필수** (정적 분석만으로 끝내지 않음)
7. 발견된 취약점을 코드에서 직접 수정

### 12.3 Combined 모드

코드 + 배포된 타겟이 모두 있는 경우:
- 코드 분석으로 동적 테스트를 가속
- 동적 이상 징후로 코드 리뷰 우선순위 결정
- 상호 검증

스캔 모드별 reasoning effort 연동: [`quick.md`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/scan_modes/quick.md) / [`standard.md`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/scan_modes/standard.md) / [`deep.md`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/scan_modes/deep.md)

---

## 13. 기본 LLM의 한계 정리

| 능력 | 기본 LLM | Strix |
|------|----------|-------|
| nmap 포트 스캔 실행 | 불가 | Docker 내 직접 실행 |
| sqlmap으로 SQLi 자동 탐지 | 불가 | 터미널 도구로 실행 |
| 웹 브라우저로 실제 폼 제출 | 불가 | Playwright 브라우저 도구 |
| HTTP 트래픽 가로채기/분석 | 불가 | Caido 프록시 도구 |
| 수천 페이로드 동시 분사 | 불가 | Python asyncio 스크립트 |
| 여러 취약점 동시 탐색 | 불가 | 멀티에이전트 트리 |
| 발견 결과 자동 검증 | 불가 | 전용 검증 에이전트 |
| 10시간+ 연속 테스트 | 불가 | 에이전트 루프 (300 이터레이션 x N 에이전트) |
| 컨텍스트 무한 확장 | 불가 | LLM 메모리 압축 |
| CVSS 점수 자동 계산 | 불가 | cvss 라이브러리 통합 |
| 실시간 최신 페이로드 검색 | 불가 | Web Search (Perplexity) 도구 |
| 중복 발견 자동 제거 | 불가 | LLM 기반 deduplication |
| 취약점 자동 수정 (White-box) | 불가 | 코드 수정 에이전트 |

---

## 14. 결론

Strix가 기본 LLM 대비 모의침투를 잘 수행하는 이유는 단일 요인이 아니라
**다층적 시스템 설계의 결합**에 있다:

1. **프롬프트 계층**: 자율성, 지속성, 효율성을 강제하는 정교한 시스템 프롬프트
   — [`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja)
2. **도구 계층**: 15+ 전용 도구로 실제 명령 실행, 브라우저 자동화, 트래픽 분석
   — [`tools/`](https://github.com/usestrix/strix/tree/5a76fab/strix/tools), [`registry.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/registry.py)
3. **지식 계층**: 17종 이상의 취약점별 전문 스킬 동적 주입
   — [`skills/`](https://github.com/usestrix/strix/tree/5a76fab/strix/skills)
4. **실행 계층**: Docker 샌드박스에서의 격리된 실행 환경 (Kali Linux + 전체 보안 도구)
   — [`docker_runtime.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py)
5. **협업 계층**: 계층적 멀티에이전트 시스템으로 분할 정복
   — [`agents_graph_actions.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py), [`base_agent.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/base_agent.py)
6. **기억 계층**: LLM 기반 메모리 압축으로 장기 컨텍스트 유지
   — [`memory_compressor.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py)
7. **품질 계층**: LLM 기반 중복 제거, 자동 검증, CVSS 채점
   — [`dedupe.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/dedupe.py)

기본 LLM은 "텍스트 생성기"이고, Strix는 그 위에 **에이전트 프레임워크**,
**실행 환경**, **전문 지식**, **협업 시스템**을 쌓아올린 **특화된 보안 테스트 플랫폼**이다.
