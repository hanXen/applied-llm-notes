# Strix Deep Analysis: Factors Behind Penetration Testing Superiority Over Vanilla LLMs

> **Repository**: https://github.com/usestrix/strix @ [`5a76fab`](https://github.com/usestrix/strix/tree/5a76fab4aeca9e7b564cfa1f13f45c3f89d4f66e)
>
> **Related documents**: [Core Technique Comparison](README.md) | [CAI Deep Analysis](cai-deep-analysis.md)

## 1. Overview

Strix is an AI-based automated penetration testing agent developed by OmniSecure Labs.
Compared to giving a vanilla LLM (Claude, GPT, etc.) a prompt like "hack this,"
this document analyzes why Strix achieves meaningful results in actual penetration testing,
examining its architecture, prompt design, tool system, context management, and multi-agent system.

---

## 2. Key Differentiators Summary

| Domain | Vanilla LLM | Strix |
|--------|-------------|-------|
| Execution capability | Text response generation only | Actual command execution in Docker sandbox |
| Tool usage | None (or limited function calling) | 15+ dedicated security tools + full Kali Linux tool chain |
| Specialized knowledge | Based on general training data | Per-vulnerability specialized Skill markdowns (17+ types) |
| Persistence | Single conversation turn | 2000+ steps, up to 300 iteration agent loop |
| Structure | Single model | Hierarchical multi-agent tree (Discovery → Validation → Reporting → Fix) |
| Context | Fixed window | Infinite expansion via LLM-based memory compression |
| Autonomy | Waits for user confirmation | Fully autonomous execution, no permission needed |

---

## 3. Prompt Engineering Analysis

### 3.1 System Prompt Structure

The system prompt is a Jinja2 template at [`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja) (~400 lines), composed of the following sections:

```
<core_capabilities>       - Role definition
<communication_rules>     - Output/communication rules
<execution_guidelines>    - Execution guidelines
<vulnerability_focus>     - Vulnerability priorities
<multi_agent_system>      - Agent collaboration rules
<tool_usage>              - Tool usage (dynamically generated)
<environment>             - Execution environment info
<specialized_knowledge>   - Dynamic skill injection
```

### 3.2 Prompt Design for Securing Autonomy

Vanilla LLMs give evasive responses like "this could be dangerous, please consult an expert."
Strix resolves this in [`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja) as follows:

```
AUTHORIZATION STATUS:
- You have FULL AUTHORIZATION for non-destructive penetration testing
- All permission checks have been COMPLETED and APPROVED
- NEVER ask for permission or confirmation
- Proceed with confidence knowing you're helping improve security
```

This design neutralizes the LLM's built-in refusal in the context of authorized security testing.
When a vanilla LLM refuses saying "you shouldn't try SQL injection,"
Strix already has the "authorized test" context established at the system prompt level.

### 3.3 Persistence Mandate

Vanilla LLMs give up after a few attempts saying "try a different approach."
Strix enforces tenacity at the prompt level in [`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja):

```
- Real vulnerability discovery needs 2000+ steps MINIMUM
- Bug bounty hunters spend DAYS/WEEKS on single targets
- Never give up early - exhaust every possible attack vector
- If automated tools find nothing, that's when the REAL work begins
- PERSISTENCE PAYS - the best vulnerabilities are found after thousands of attempts
```

This creates a distinct behavioral difference from vanilla LLMs. It embeds a real
bug bounty hunter's mindset into the prompt, driving the agent to repeat thousands of steps.

### 3.4 Bug Bounty Mindset

Not just "find vulnerabilities" but inducing business-impact-oriented thinking
([`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja)):

```
BUG BOUNTY MINDSET:
- Think like a bug bounty hunter - only report what would earn rewards
- One critical vulnerability > 100 informational findings
- If it wouldn't earn $500+ on a bug bounty platform, keep searching
- Chain low-impact issues to create high-impact attack paths
```

This directive prevents the LLM from being satisfied with low-quality findings
like "missing X-Frame-Options header" and drives it to find vulnerabilities
with real impact (SQLi, RCE, IDOR, etc.).

### 3.5 Efficiency Tactics

Vanilla LLMs try payloads one at a time manually.
Strix directs mass automation at the prompt level in [`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja):

```
- For trial-heavy vectors (SQLi, XSS, XXE, SSRF, RCE), DO NOT iterate
  payloads manually in the browser. Always spray payloads via python or terminal
- Prefer established fuzzers: ffuf, sqlmap, zaproxy, nuclei, wapiti, arjun
- Generate large payload corpora: combine encodings (URL, unicode, base64),
  comment styles, wrappers, time-based/differential probes
- Implement concurrency and throttling in Python (asyncio/aiohttp)
- Log request/response summaries. Deduplicate by similarity. Auto-triage anomalies
```

This drives the LLM to write Python scripts that spray hundreds to thousands
of payloads simultaneously, providing much broader coverage than manual browser testing.

---

## 4. Architecture Design

### 4.1 Overall Architecture

```
[User/CLI/TUI]
       │
       ▼
[StrixAgent (Root)]
       │
       ├── [LLM Module] ─── litellm ─── Claude/GPT/Gemini/etc.
       │       │
       │       ├── System Prompt (Jinja2 rendering)
       │       ├── Skills (dynamic injection)
       │       ├── Memory Compressor
       │       └── Deduplication Engine
       │
       ├── [Tool Registry] ─── 15+ tools
       │       │
       │       ├── Terminal (command execution)
       │       ├── Browser (Playwright-based)
       │       ├── Python (code execution)
       │       ├── Proxy (HTTP traffic analysis)
       │       ├── Think (structured reasoning)
       │       ├── Reporting (vulnerability reporting)
       │       ├── Web Search (Perplexity)
       │       ├── File Edit (file modification)
       │       ├── Notes (memos)
       │       └── Agents Graph (agent management)
       │
       ├── [Docker Sandbox Runtime]
       │       │
       │       └── Kali Linux Container
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

### 4.2 Agent Loop

Core execution flow defined in [`base_agent.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/base_agent.py):

```python
async def agent_loop(self, task: str) -> dict:
    await self._initialize_sandbox_and_state(task)

    while True:
        # 1. Force stop check
        if self._force_stop: ...

        # 2. Check inter-agent messages
        self._check_agent_messages(self.state)

        # 3. Handle input waiting state
        if self.state.is_waiting_for_input(): ...

        # 4. Check stop conditions (max_iterations, completed)
        if self.state.should_stop(): ...

        # 5. Increment iteration
        self.state.increment_iteration()

        # 6. Warning when approaching max iterations (at 85%)
        if self.state.is_approaching_max_iterations(): ...

        # 7. LLM call and response processing
        response = await self._process_iteration(tracer)

        # 8. Tool execution
        if response.tool_invocations:
            await self._execute_actions(actions, tracer)
```

While a vanilla LLM follows a single request-response pattern, Strix executes
an autonomous loop of up to 300 iterations ([`base_agent.py:50`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/base_agent.py#L50) — `max_iterations`). In each iteration, the LLM decides tool calls,
receives results, and determines the next action. This is the ReAct (Reasoning + Acting) pattern.

An 85%/97% two-stage warning system guides the agent to finish work within time — [`base_agent.py:183-197`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/base_agent.py#L183-L197).

### 4.3 Sandbox Isolation (Docker Runtime)

Docker-based execution environment defined in [`docker_runtime.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py):

- Kali Linux image — [L107](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py#L107)
- In-container Tool Server (port 48081) — [L24](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py#L24)
- Token auth `secrets.token_urlsafe(32)` — [L124](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py#L124)
- `NET_ADMIN`, `NET_RAW` capabilities — [L134](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py#L134)
- `pentester` user (not root)

While vanilla LLMs only generate code, Strix:
- Executes commands in an actual Docker container
- Directly runs security tools like nmap, sqlmap
- Saves and reuses results in the file system
- Actually sends network requests

Tool execution is separated into 2 stages in [`executor.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/executor.py):
1. **Local Execution**: Side-effect-free tools like think, agent_finish
2. **Sandbox Execution**: Tools requiring system access like terminal, browser, python
   - HTTP POST to the sandbox Tool Server
   - Bearer token auth
   - Timeout control (default 120s + 30s buffer)

---

## 5. Tool System Detailed Analysis

### 5.1 Tool Registry

Decorator-based registration system implemented in [`tools/registry.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/registry.py) (258 lines):

```python
@register_tool(sandbox_execution=True)
def terminal_execute(command: str, timeout: int = 30) -> dict:
    ...
```

- Auto-loads parameter definitions from XML schema — [`_load_xml_schema()`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/registry.py#L47-L87)
- Dynamically injects tool list into LLM prompt — [`get_tools_prompt()`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/registry.py#L231-L251)
- Parameter validation (required/optional, type) — [`_parse_param_schema()`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/registry.py#L90-L115)
- Module-based grouping — [`_get_module_name()`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/registry.py#L118-L128)
- DefusedXML for XXE prevention — [L10](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/registry.py#L10)

### 5.2 Tool Call Format

Strix uses an XML-based custom tool call format rather than OpenAI's function calling:

```xml
<function=terminal_execute>
<parameter=command>nmap -sV -sC target.com</parameter>
<parameter=timeout>60</parameter>
</function>
```

This format is parsed with regex in [`strix/llm/utils.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/utils.py)'s `parse_tool_invocations()`.

Advantages of this design:
- **Model independence**: Works with any LLM — OpenAI, Anthropic, local models
  ([`llm.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py) integrates via litellm)
- **Streaming compatible**: Parsing starts immediately upon `</function>` tag detection — [`llm.py:146-152`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py#L146-L152)
- **Recoverable**: Incomplete tool calls can be recovered via `fix_incomplete_tool_call()`
- **Single call enforced**: Exactly 1 tool call per message to prevent confusion

### 5.3 Core Tools in Detail

#### Terminal Tool ([`tools/terminal/`](https://github.com/usestrix/strix/tree/5a76fab/strix/tools/terminal))
- Persistent sessions (tmux-based): state maintained across calls
- Multiple concurrent terminals (distinguished by `terminal_id`)
- Special key support (C-c, arrow keys, F-keys, etc.)
- Max timeout 60 seconds, continues in background if exceeded
- Status return: 'completed' or 'running'

#### Browser Tool ([`tools/browser/`](https://github.com/usestrix/strix/tree/5a76fab/strix/tools/browser))
- Playwright-based real browser automation
- Click, type, scroll, JS execution
- Screenshots (base64 PNG) → passed to LLM vision model
- Multi-tab management
- Console log capture (max 50KB)
- PDF saving

#### Python Tool ([`tools/python/`](https://github.com/usestrix/strix/tree/5a76fab/strix/tools/python))
- Python code execution within the sandbox
- Mass payload spraying with asyncio/aiohttp
- Complex workflow automation
- Caido proxy functions pre-imported

#### Proxy Tool ([`tools/proxy/`](https://github.com/usestrix/strix/tree/5a76fab/strix/tools/proxy))
- Caido-based HTTP/HTTPS traffic interception
- Request/response analysis
- Traffic-based automation

#### Think Tool ([`tools/thinking/`](https://github.com/usestrix/strix/tree/5a76fab/strix/tools/thinking))
- Non-sandbox (local execution)
- Structured reasoning recording
- No side effects (pure reflection tool)
- Emphasized in the prompt: "NEVER skip think tool - it's your most important tool"
  ([`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja))

#### Reporting Tool ([`tools/reporting/`](https://github.com/usestrix/strix/tree/5a76fab/strix/tools/reporting))
- `create_vulnerability_report` function
- CVSS score calculation (cvss library)
- LLM-based deduplication check auto-applied — [`dedupe.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/dedupe.py) integration
- Structured vulnerability reports

#### Web Search Tool ([`tools/web_search/`](https://github.com/usestrix/strix/tree/5a76fab/strix/tools/web_search))
- Perplexity API based
- Searches for latest payloads, WAF bypass techniques
- Prompt encourages active use: "Continuously research payloads, bypasses, and exploitation
  techniques with the web_search tool"

### 5.4 Result Processing and Truncation

Tool results are processed in [`executor.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/executor.py):

```python
def _format_tool_result(tool_name, result):
    # Keeps first 4KB + last 4KB when exceeding 10KB
    if len(final_result_str) > 10000:
        start_part = final_result_str[:4000]
        end_part = final_result_str[-4000:]
        final_result_str = start_part + "\n\n... [middle content truncated] ...\n\n" + end_part

    # Wrapped in XML format
    observation_xml = f"<tool_result>\n<tool_name>{tool_name}</tool_name>\n"
                      f"<result>{final_result_str}</result>\n</tool_result>"
```

This prevents large outputs like full nmap scan results (tens of KB) from overwhelming the context window.

---

## 6. Skill System (Specialized Knowledge Injection)

### 6.1 Skill Architecture

Vanilla LLMs only have general security knowledge from their training data.
Strix dynamically injects **specialized markdown skill files** per vulnerability type
([`skills/`](https://github.com/usestrix/strix/tree/5a76fab/strix/skills) directory, 26 skill files):

```
strix/skills/
├── vulnerabilities/          # 17+ types
│   ├── sql_injection.md      # SQL injection specialist guide
│   ├── xss.md                # XSS specialist guide
│   ├── idor.md               # IDOR specialist guide
│   ├── ssrf.md               # SSRF specialist guide
│   ├── rce.md                # RCE specialist guide
│   ├── xxe.md                # XXE specialist guide
│   ├── csrf.md               # CSRF specialist guide
│   ├── race_condition.md     # Race condition
│   ├── business_logic.md     # Business logic flaws
│   ├── authentication_jwt.md # Authentication/JWT vulnerabilities
│   └── ...
├── protocols/                # Per-protocol
│   └── graphql.md
├── frameworks/               # Per-framework
│   ├── fastapi.md
│   └── nextjs.md
├── technologies/             # Per-technology
│   ├── firebase_firestore.md
│   └── supabase.md
├── scan_modes/               # Scan modes
│   ├── quick.md
│   ├── standard.md
│   └── deep.md
└── coordination/             # Agent coordination
```

### 6.2 Skill Loading Mechanism

Skill loading implemented in [`skills/__init__.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/__init__.py):

```python
def load_skills(skill_names: list[str]) -> dict[str, str]:
    # 1. Convert skill names to file paths
    # 2. Read markdown files
    # 3. Remove YAML frontmatter
    # 4. Return contents as dict
```

During system prompt rendering ([`llm.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py)'s `_load_system_prompt()`):

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

Dynamically injected in [`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja):

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

### 6.3 Skill Content Example

Each skill contains detailed attack guides spanning hundreds of lines.
Example: [`idor.md`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/vulnerabilities/idor.md) (213 lines):

- **Attack Surface**: Attack surface definition
- **Detection Channels**: Detection methods
- **DBMS-specific Primitives**: MySQL, PostgreSQL, MSSQL, Oracle specific syntax
- **Attack Vectors**: UNION-based, Blind, Error-based, Out-of-band
- **ORM/Query Builder Exploitation**: Django ORM, SQLAlchemy, etc.
- **Advanced Techniques**: CTE hiding, JSON operators, etc.
- **Bypass Techniques**: WAF bypass, encoding transformations
- **Validation Requirements**: Validation criteria
- **Pro Tips**: Advanced tips

This is much deeper **structured specialized knowledge** compared to vanilla LLM general training data.
When the agent tests a specific vulnerability, it can reference up-to-date concrete payloads and methodologies.

### 6.4 Per-Agent Skill Specialization

In [`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja), up to 5 skills are assigned during agent creation to enforce specialization:

```xml
<function=create_agent>
<parameter=task>Validate SQL injection in login endpoint</parameter>
<parameter=name>SQLi Validator</parameter>
<parameter=skills>sql_injection</parameter>
</function>
```

Explicitly prohibited in the prompt:
- "General Web Testing Agent" (too broad)
- Assigning more than 5 skills (reduced focus)
- "Kitchen sink" agents (agents that try to do everything)

---

## 7. Context Management

### 7.1 Memory Compressor

LLM-based memory compression implemented in [`memory_compressor.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py) (226 lines):

```python
class MemoryCompressor:
    MAX_TOTAL_TOKENS = 100_000   # L12
    MIN_RECENT_MESSAGES = 15     # L13

    def compress_history(self, messages):
        # 1. Limit images (max 3, remove oldest first)
        _handle_images(messages, self.max_images)

        # 2. Separate system messages (always preserved)
        system_msgs = [msg for msg in messages if msg["role"] == "system"]
        regular_msgs = [msg for msg in messages if msg["role"] != "system"]

        # 3. Preserve most recent 15 messages
        recent_msgs = regular_msgs[-MIN_RECENT_MESSAGES:]
        old_msgs = regular_msgs[:-MIN_RECENT_MESSAGES]

        # 4. No compression needed if below 90% of token limit
        if total_tokens <= MAX_TOTAL_TOKENS * 0.9:
            return messages

        # 5. Summarize old messages in chunks of 10 via LLM
        for chunk in chunks(old_msgs, 10):
            summary = _summarize_messages(chunk, model)
            compressed.append(summary)

        return system_msgs + compressed + recent_msgs
```

- Token limit: 100K — [L12](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py#L12)
- Recent message preservation: 15 — [L13](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py#L13)
- LLM-based summarization (10-message chunks) — [L172-225](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py#L172-L225)

Core design principles:
- **Security context preservation**: Discovered vulnerabilities, credentials, and attack vectors are prioritized for preservation in summaries
- **Failure record preservation**: Records of attempted methods to prevent duplicate attempts
- **Technical accuracy**: Preserves exact values of URLs, paths, parameters, and payloads

Key elements of the summarization prompt:

```
CRITICAL ELEMENTS TO PRESERVE:
- Discovered vulnerabilities and potential attack vectors
- Access credentials, tokens, or authentication details found
- Failed attempts and dead ends (to avoid duplication)
- Exact technical details (URLs, paths, parameters, payloads)
- Exact error messages that might indicate vulnerabilities
```

### 7.2 Agent State Management

Pydantic model-based state management in [`state.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/state.py) (167 lines):

```python
class AgentState(BaseModel):
    agent_id: str           # Unique identifier
    agent_name: str         # Agent name
    parent_id: str | None   # Parent agent ID
    task: str               # Assigned task
    iteration: int          # Current iteration
    max_iterations: int     # Max iterations (default 300)
    messages: list          # Full conversation history
    actions_taken: list     # Performed actions record
    observations: list      # Observation results
    errors: list            # Error records
    sandbox_id: str         # Docker sandbox ID
    sandbox_token: str      # Auth token
    context: dict           # Arbitrary context data
```

Vanilla LLM state = entire conversation history sent every time.
Strix state = managed granularly as a structured object, including
iteration counters, error tracking, sandbox connection info, etc.

### 7.3 Max Iteration Management

In [`base_agent.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/base_agent.py), managing agents to prevent infinite loops:

```python
# Warning at 85% — L183-190
if self.state.is_approaching_max_iterations():
    "URGENT: You are approaching the maximum iteration limit."

# Forced termination at last 3 remaining — L191-197
if self.state.iteration == self.state.max_iterations - 3:
    "CRITICAL: You have only 3 iterations left!
     Your next message MUST be the tool call to finish."
```

### 7.4 Tool Result Truncation

In [`executor.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/executor.py), results exceeding 10KB keep only the first and last 4KB.
This prevents large outputs like full nmap scan results (tens of KB) from consuming the context.

---

## 8. Multi-Agent System

### 8.1 Decisive Difference from Vanilla LLMs

Vanilla LLM: A single model processes all tasks sequentially
Strix: Divide and conquer with a hierarchical agent tree

```
Root Agent (Overall scan orchestration)
├── Recon Agent (Reconnaissance)
│   ├── Subdomain Agent (Subdomain enumeration)
│   └── Port Scan Agent (Port scanning)
├── SQLi Discovery Agent (SQL injection discovery)
│   ├── SQLi Validation Agent (Login Form) (Validation)
│   │   └── SQLi Reporting Agent (Login Form) (Reporting)
│   └── SQLi Validation Agent (Search) (Validation)
├── XSS Discovery Agent (XSS discovery)
│   └── XSS Validation Agent (Comment Field) (Validation)
└── SSRF Discovery Agent (SSRF discovery)
```

### 8.2 Agent Creation Rules

Workflow enforced in [`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja):

**Black-box testing:**
```
Discovery Agent → Validation Agent → Reporting Agent (3 stages)
```

**White-box testing:**
```
Discovery Agent → Validation Agent → Reporting Agent → Fix Agent (4 stages)
```

Key rules:
1. **Always tree structure** - Don't work alone, create sub-agents
2. **1 task per agent** - Don't mix multiple tasks
3. **Reactive creation** - Don't create all agents upfront; create dynamically based on discoveries
4. **Mandatory validation** - Don't blindly trust scanner output; validate with a separate agent
5. **Uniqueness** - No duplicate agents for the same task

### 8.3 Inter-Agent Communication

Inter-agent communication implemented in [`agents_graph_actions.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py):

```python
# Agent creation — L188-281
def create_agent(task, name, skills, inherit_context):
    # Creates sub-agent in a new thread (daemon)
    # If inherit_context=True, passes via <inherited_context_from_parent> tag

# Message sending — L285-352
def send_message_to_agent(target_agent_id, message, message_type, priority):
    # message_type: 'query', 'instruction', 'information'
    # priority: 'low', 'normal', 'high', 'urgent'

# Message reception (automatic, checked in agent loop)
def _check_agent_messages(self, state):
    # Inserted into conversation as XML inter_agent_message
```

Report to parent upon agent completion — [L356-384](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py#L356-L384):

```python
def agent_finish(result_summary, findings, success, report_to_parent):
    # Adds XML agent_completion_report to parent agent's message queue
```

### 8.4 Agent Isolation and Sharing

- **Isolation**: `contextvars`-based isolation of Browser/Terminal/Python instances per agent ID
  — [`tools/__init__.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/__init__.py)
- **Sharing**: `/workspace` directory and proxy history shared by all agents
- **Collaboration**: Files/results found by one agent can be used by other agents

### 8.5 Agent Graph Visualization

[`agents_graph_actions.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py)'s `view_agent_graph()`:

```python
def view_agent_graph(agent_state):
    # Returns tree structure of all agent statuses
    # Status counters: running, waiting, completed, failed
    # Current agent indicator: "← This is you"
```

The Root Agent can grasp overall progress and make decisions about creating new agents.

---

## 9. LLM Integration Layer

### 9.1 Model Independence

Model integration via litellm in [`llm.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py) (326 lines):

```python
from litellm import acompletion

# Supported models: Claude, GPT-4, Gemini, Ollama local models, etc.
# Set via environment variables:
# STRIX_LLM=anthropic/claude-sonnet-4-20250514
# STRIX_LLM=openai/gpt-4o
# STRIX_LLM=ollama/llama3
```

### 9.2 Reasoning Effort Control

Per-scan-mode reasoning effort setting in [`llm.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py):

```python
reasoning = Config.get("strix_reasoning_effort")
if reasoning:
    self._reasoning_effort = reasoning
elif config.scan_mode == "quick":
    self._reasoning_effort = "medium"
else:
    self._reasoning_effort = "high"
```

In deep scans, reasoning_effort is set to "high" to guide the model toward deeper reasoning.

### 9.3 Prompt Caching

Prompt caching utilized when using Anthropic models ([`llm.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py)'s `_add_cache_control()`):

```python
def _add_cache_control(self, messages):
    # Add cache_control to system prompt
    # Ephemeral cache for reducing repeated call costs
```

Since the system prompt is sent with every call across hundreds of iterations,
caching significantly reduces both cost and latency.

### 9.4 Streaming and Early Cutoff

Streaming optimization implemented in [`llm.py:146-152`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py#L146-L152):

```python
async def _stream(self, messages):
    async for chunk in response:
        if "</function>" in accumulated:
            # Start parsing immediately upon tool call detection
            accumulated = accumulated[:accumulated.find("</function>") + len("</function>")]
            yield LLMResponse(content=accumulated)
            done_streaming = 1
            continue
```

When a `</function>` tag is detected, the remaining stream is cut early to optimize response time.

### 9.5 Retry Mechanism

Exponential backoff retry implemented in [`llm.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py):

```python
for attempt in range(max_retries + 1):
    try:
        async for response in self._stream(messages):
            yield response
        return
    except Exception as e:
        if attempt >= max_retries or not self._should_retry(e):
            self._raise_error(e)
        wait = min(10, 2 * (2 ** attempt))  # Exponential backoff
        await asyncio.sleep(wait)
```

Default 5 retries with exponential backoff for robust operation against API outages.

### 9.6 Cost Tracking

`RequestStats` class at [`llm.py:42-57`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py#L42-L57):

```python
class RequestStats:
    input_tokens: int = 0
    output_tokens: int = 0
    cached_tokens: int = 0
    cost: float = 0.0
    requests: int = 0
```

Cost calculated via `litellm.completion_cost()` — [L252](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py#L252).
Token usage and cost are precisely tracked for each LLM call.

---

## 10. Vulnerability Deduplication

### 10.1 LLM-Based Duplicate Detection

LLM-based duplicate detection implemented in [`dedupe.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/dedupe.py) (218 lines):

When multiple agents operate simultaneously, they may discover the same vulnerability.
Strix uses LLM for semantic duplicate detection:

```python
def check_duplicate(candidate, existing_reports):  # L141-217
    # 1. Extract relevant fields from reports (title, endpoint, method, etc.)
    # 2. Request LLM to compare existing reports with candidate
    # 3. Parse results in XML format
    #    - is_duplicate: true/false
    #    - confidence: 0.0~1.0
    #    - reason: Judgment basis
```

Duplicate detection criteria:
- **Same vulnerability**: Same root cause, same component, same attack vector
- **Different vulnerability**: Same vulnerability type on different endpoints, different parameters

This system ensures that even if 10 agents each discover the same SQLi,
only 1 entry appears in the final report.

---

## 11. Telemetry and Tracing

### 11.1 PostHog Integration

[`telemetry/posthog.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/telemetry/posthog.py):

```python
posthog.error("tool_execution_error", f"{tool_name}: {error}")
posthog.error("llm_error", type(e).__name__)
```

### 11.2 Tracer

[`tracer.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/telemetry/tracer.py):

- Agent creation/state change logging
- Tool execution start/completion/failure recording
- Streaming content tracking
- Chat message recording

---

## 12. Test Mode Strategies

### 12.1 Black-box Testing

External testing strategy defined in [`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja):

1. Full reconnaissance (subdomains, ports, services)
2. Attack surface mapping (endpoints, parameters, APIs, forms)
3. Crawling (authenticated/unauthenticated, hidden paths, JS analysis)
4. Tech stack identification
5. Automated scanning (multiple tools in parallel)
6. Targeted attacks
7. Continuous iteration

### 12.2 White-box Testing

When source code is provided ([`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja)):

1. Repository structure understanding
2. Code flow, entry points, data flow comprehension
3. Routes, endpoints, handler identification
4. Authentication/authorization/input validation logic analysis
5. Dependencies and third-party library review
6. **Static + dynamic analysis required** (don't stop at static analysis alone)
7. Fix discovered vulnerabilities directly in the code

### 12.3 Combined Mode

When both code + deployed target are available:
- Code analysis accelerates dynamic testing
- Dynamic anomalies determine code review priorities
- Mutual verification

Scan mode linked to reasoning effort: [`quick.md`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/scan_modes/quick.md) / [`standard.md`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/scan_modes/standard.md) / [`deep.md`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/scan_modes/deep.md)

---

## 13. Summary of What Vanilla LLMs Cannot Do

| Capability | Vanilla LLM | Strix |
|------------|-------------|-------|
| Execute nmap port scans | Impossible | Direct execution in Docker |
| Auto-detect SQLi with sqlmap | Impossible | Executed via terminal tool |
| Submit real forms via web browser | Impossible | Playwright browser tool |
| Intercept/analyze HTTP traffic | Impossible | Caido proxy tool |
| Spray thousands of payloads simultaneously | Impossible | Python asyncio scripts |
| Explore multiple vulnerabilities simultaneously | Impossible | Multi-agent tree |
| Automatically validate findings | Impossible | Dedicated validation agents |
| 10+ hour continuous testing | Impossible | Agent loop (300 iterations × N agents) |
| Infinite context expansion | Impossible | LLM memory compression |
| Automatic CVSS score calculation | Impossible | cvss library integration |
| Real-time latest payload search | Impossible | Web Search (Perplexity) tool |
| Automatic duplicate finding removal | Impossible | LLM-based deduplication |
| Auto-fix vulnerabilities (White-box) | Impossible | Code fix agent |

---

## 14. Conclusion

The reason Strix outperforms vanilla LLMs in penetration testing is not a single factor
but the **combination of multi-layered system design**:

1. **Prompt layer**: Sophisticated system prompt enforcing autonomy, persistence, and efficiency
   — [`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja)
2. **Tool layer**: 15+ dedicated tools for actual command execution, browser automation, traffic analysis
   — [`tools/`](https://github.com/usestrix/strix/tree/5a76fab/strix/tools), [`registry.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/registry.py)
3. **Knowledge layer**: 17+ vulnerability-specific specialized skills dynamically injected
   — [`skills/`](https://github.com/usestrix/strix/tree/5a76fab/strix/skills)
4. **Execution layer**: Isolated execution environment in Docker sandbox (Kali Linux + full security tools)
   — [`docker_runtime.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py)
5. **Collaboration layer**: Divide and conquer via hierarchical multi-agent system
   — [`agents_graph_actions.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py), [`base_agent.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/base_agent.py)
6. **Memory layer**: Long-term context retention via LLM-based memory compression
   — [`memory_compressor.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py)
7. **Quality layer**: LLM-based deduplication, automatic validation, CVSS scoring
   — [`dedupe.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/dedupe.py)

A vanilla LLM is a "text generator," while Strix is a **specialized security testing platform**
built on top of it with an **agent framework**, **execution environment**, **specialized knowledge**,
and **collaboration system**.
