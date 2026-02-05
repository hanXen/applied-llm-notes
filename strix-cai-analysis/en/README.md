# LLM-Based Penetration Testing Automation: In-Depth Analysis of Core Techniques

> Design techniques and their rationale for improving penetration testing effectiveness over vanilla LLMs, derived from source code analysis of Strix and CAI
>
> **Repositories (commit references):**
> - Strix: https://github.com/usestrix/strix @ [`5a76fab`](https://github.com/usestrix/strix/tree/5a76fab4aeca9e7b564cfa1f13f45c3f89d4f66e)
> - CAI: https://github.com/aliasrobotics/cai @ [`e22a122`](https://github.com/aliasrobotics/cai/tree/e22a1220f764e2d7cf9da6d6144926f53ca01cde)
>
> **Detailed analysis**: [Strix Deep Analysis](strix-deep-analysis.md) | [CAI Deep Analysis](cai-deep-analysis.md)

---

## Introduction: Limitations of Vanilla LLMs

If you directly ask ChatGPT or Claude to "penetration test this site":

1. **No tools** — Cannot directly run nmap, sqlmap, or browsers
2. **Finite context** — Forgets earlier findings during long test processes
3. **No structured workflow** — Cannot systematically traverse all vulnerability types
4. **Diluted expertise** — Difficult to simultaneously be an IDOR expert and an SQLi expert
5. **No controls** — False positive reports from hallucinations, cost runaway

This document analyzes **what techniques** Strix and CAI each use to overcome these limitations and **why those techniques are effective**, with code-level evidence.

---

## 1. Prompt Engineering — Turning an LLM into a Pentester

> Problem solved: Vanilla LLMs only give generic advice when asked to "hack this"

### 1.1 Strix: Aggressive Autonomous Directives + Skill Injection

**System Prompt** ([`system_prompt.jinja`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/StrixAgent/system_prompt.jinja), 407 lines):

Dynamically generated via Jinja2 Template, with core directives:

- **Aggressive tone**: "GO SUPER HARD on all targets", "2000+ steps MINIMUM", "UNLEASH FULL CAPABILITY"
- **Full autonomy**: "NEVER ask for user input or confirmation - always proceed autonomously"
- **Bug Bounty Mindset**: "One critical vulnerability > 100 informational findings. If it wouldn't earn $500+ on bug bounty platform, keep searching"
- **10 mandatory vulnerability checklist**: IDOR, SQLi, SSRF, XSS, XXE, RCE, CSRF, Race Condition, Business Logic, Auth/JWT

**Phase-based Workflow Enforcement**:
- Black-box Phase 1: Full reconnaissance
- White-box Phase 1: Code mapping
- Phase 2: Sub-Agent creation per vulnerability type x component

**Skill System** ([`skills/`](https://github.com/usestrix/strix/tree/5a76fab/strix/skills), 26 Skill files):
- Composed of markdown files with YAML frontmatter
- Up to 5 Skills dynamically injected into system prompt per Agent — `{{ get_skill(skill_name) }}`
- Auto-added per scan mode — [`quick.md`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/scan_modes/quick.md) / [`standard.md`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/scan_modes/standard.md) / [`deep.md`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/scan_modes/deep.md)
- Example — IDOR Skill ([`idor.md`](https://github.com/usestrix/strix/blob/5a76fab/strix/skills/vulnerabilities/idor.md), 213 lines): covers attack surface identification, high-value targets, reconnaissance methods, bypass techniques, and chaining strategies

**Prompt Composition Flow**:
```
System Prompt (Jinja2, 407 lines)
  ├── Core Instructions (aggressive autonomous behavior directives)
  ├── Scan mode guide (quick/standard/deep Skill)
  ├── Vulnerability Skills (up to 5, dynamically injected)
  ├── Tool Schema (XML, auto-generated from Registry)
  ├── Environment info (Kali Linux Container tool list)
  └── Agent Identity Metadata (name, ID)
```

### 1.2 CAI: Systematic Methodology + Memory Injection

**Master Template** ([`system_master_template.md`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/core/system_master_template.md), 190 lines):

Dynamically generated via Mako Template, with core directives:

- **Methodology-driven**: Systematic step-by-step approach (Reconnaissance → Threat Modeling → Testing → Exploitation → Report)
- **Safety-first**: "Prefer safe, read-only tests first", "Do not attempt data deletion or service disruption"
- **Auto-injected environment context** — [L128-157](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/core/system_master_template.md#L128-L157): OS info, IP, hostname, wordlist paths, etc. **collected at runtime**
- **Memory injection** — [L98-105](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/core/system_master_template.md#L98-L105): Episodic/Semantic memories from previous sessions integrated into the prompt

**Agent-specific Specialized Prompts**:
- [`system_red_team_agent.md`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/system_red_team_agent.md) — "gain root access and find flags", non-interactive command enforcement ([L22-31](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/system_red_team_agent.md#L22-L31)), shell session management ([L38-60](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/system_red_team_agent.md#L38-L60))
- [`system_web_pentester.md`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/system_web_pentester.md) — 7-step methodology ([L48-154](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/system_web_pentester.md#L48-L154)), "10-15 custom test cases" (Business-Logic Abuse Backlog, [L95](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/prompts/system_web_pentester.md#L95))

**Prompt Composition Flow**:
```
Master Template (Mako, 190 lines)
  ├── Agent Instructions (per-agent .md file)
  ├── Compressed previous conversation context
  ├── Memory injection (Episodic/Semantic from Qdrant)
  ├── Reasoning block (reasoning model output)
  └── Environment context (OS, IP, wordlists, etc. collected at runtime)
```

### 1.3 Insight: Why These Techniques Are Effective

| Aspect | Strix | CAI |
|--------|-------|-----|
| **Tone** | Aggressive/unlimited ("GO SUPER HARD") | Systematic/safety-first ("safe, read-only first") |
| **Autonomy** | Fully autonomous ("never ask the user") | Semi-autonomous (HITL, Ctrl+C interrupt) |
| **Knowledge injection** | Skill files (markdown, 5 per agent) | Memory system (Vector DB, learning from prior sessions) |
| **Tool Schema** | XML-based custom format | OpenAI function calling JSON Schema |
| **Environment info** | Fixed within container (Kali tool list) | Dynamically collected at runtime (OS, IP, wordlist discovery) |
| **Methodology** | Vulnerability-type based (10 mandatory checks) | Kill Chain Phase based (Recon → Exploit → C2) |
| **Workflow enforcement** | Enforced via prompt phases | Determined by agent selection |
| **Scan modes** | 3 tiers (quick/standard/deep) → linked to reasoning effort | Environment variable based (MAX_TURNS, PRICE_LIMIT) |

**Key Insights**:

1. **Aggressiveness vs methodology is determined by the coverage domain.**
- Strix's "GO SUPER HARD" stems from web app CI/CD automation — in a pipeline with no human present, the agent must not give up early, so persistence is enforced to an extreme degree.
- CAI's "safe, read-only first" exists because it covers the entire Kill Chain (Recon → C2) — aggressive unordered exploration is actually inefficient in multi-stage operations.
- **Trade-off: Aggressive tone is advantageous for uncovering hidden edge cases in web apps, but systematic methodology is more stable for operations where sequence matters, like Kill Chains.**

2. **Static Skills vs dynamic memory is a choice based on target predictability.**
- Why Strix uses markdown Skill files: web vulnerabilities are well-classified by OWASP Top 10, so a structured 213-line IDOR Playbook is more reliable than experiential learning.
- Why CAI uses Vector DB memory: across a scope covering network/IoT/forensics, static Playbooks cannot cover all scenarios, so searching similar cases from past experience is more scalable.
- **Trade-off: Static Skills guarantee consistent quality even on first-seen targets but require manual updates, while dynamic memory improves with repeated use but has lower first-run quality.**

---

## 2. Multi-Agent Architecture — Building an Expert Team

> Problem solved: Assigning everything to a single LLM dilutes expertise

### 2.1 Strix: Dynamic Agent Creation (Tree Structure)

**Agent System** ([`agents/StrixAgent/`](https://github.com/usestrix/strix/tree/5a76fab/strix/agents/StrixAgent)):
- **Root Agent (StrixAgent)**: Main coordinator orchestrating the entire penetration test
- **Sub-agents**: Specialized agents **dynamically created at runtime** for specific tasks (IDOR, SQLi, XSS, etc.) — [`agents_graph_actions.py:188-281`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py#L188-L281)
- **Agent Graph**: Directed graph (`_agent_graph`) tracking agent relationships (nodes: ID/name/task/status, edges: delegation)
- **State management**: Each agent maintains independent `AgentState` — [`state.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/state.py)

**Agent Loop** ([`base_agent.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/base_agent.py)):
```
while iteration < 300:
  1. Check state (stop/waiting/failed)
  2. iteration++
  3. Warning injected at 85%, forced wrap-up at 97%
  4. LLM.generate() → Streaming response
  5. Streaming halted at first </function> tag ← Key token-saving optimization
  6. Tool call parsing (XML)
  7. Sandbox/local branch execution
  8. Check for finish signal → Exit loop
```
- Exactly **1 tool call per iteration** (sequential)
- `max_iterations = 300` — [`base_agent.py:50`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/base_agent.py#L50)
- 85%/97% two-stage warning — [`base_agent.py:183-197`](https://github.com/usestrix/strix/blob/5a76fab/strix/agents/base_agent.py#L183-L197)
- **Early streaming cutoff**: Halts at first `</function>` tag to save unnecessary output tokens — [`llm.py:146-152`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py#L146-L152)

**Tool isolation**: `contextvars`-based isolation of Browser/Terminal/Python instances per Agent ID — [`tools/__init__.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/__init__.py)

### 2.2 CAI: Pre-defined Agent Pool + Pattern Composition

**Agent System** (~20 agents) — [`src/cai/agents/`](https://github.com/aliasrobotics/cai/tree/e22a122/src/cai/agents), [`factory.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/factory.py):

| Category | Agent | Purpose |
|----------|-------|---------|
| **Offensive** | [`red_teamer.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/red_teamer.py), [`web_pentester.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/web_pentester.py), [`bug_bounter.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/bug_bounter.py) | Penetration testing |
| **Defensive** | [`blue_teamer.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/blue_teamer.py), [`wifi_security_tester.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/wifi_security_tester.py) | Defense and hardening |
| **Investigation** | [`dfir.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/dfir.py), [`memory_analysis_agent.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/memory_analysis_agent.py), [`reverse_engineering_agent.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/reverse_engineering_agent.py) | Forensics/Analysis |
| **Mobile/IoT** | [`android_sast_agent.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/android_sast_agent.py), [`subghz_sdr_agent.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/subghz_sdr_agent.py) | Mobile/IoT security |
| **Specialized** | [`codeagent.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/codeagent.py), [`thought.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/thought.py), [`flag_discriminator.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/flag_discriminator.py), [`reporter.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/reporter.py) | Utility/Orchestration |
| **Game Theory** | redteam_gctr, blueteam_gctr, purple_team_agents | CPT analysis |

**5 Orchestration Patterns** (PatternType Enum) — [`patterns/`](https://github.com/aliasrobotics/cai/tree/e22a122/src/cai/agents/patterns), [`pattern.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/patterns/pattern.py):
- **Swarm**: Decentralized P2P + dynamic Handoff — [`red_team.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/patterns/red_team.py)
- **Hierarchical**: Top-down task assignment (PlannerAgent → specialists)
- **Sequential**: Sequential pipeline (A → B → C)
- **Conditional**: Conditional branching
- **Parallel**: Parallel execution (`CAI_PARALLEL=N`) — [`parallel_offensive_patterns.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/patterns/parallel_offensive_patterns.py)

**Agent Loop** ([`run.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/sdk/agents/run.py)):
```
while current_turn < max_turns:
  1. Dynamic system prompt rendering (memory/environment injection)
  2. Input guardrail execution (first turn only, async parallel)
  3. LLM call (including tools + handoffs)
  4. Response processing → tool calls + handoff + computer actions classification
  5. Tools executed in parallel via asyncio.gather() ← Key difference
  6. Output guardrail execution
  7. Next step decision: FinalOutput / Handoff / RunAgain
```
- **Multiple tools executed in parallel** per turn
- Built-in guardrail pipeline
- Integrated handoff routing

### 2.3 Concurrency Model

**Strix** — Wide parallelism:

Sub-Agents each run as independent threads (daemon). With 10-20 Sub-Agents each independently loading a 407-line system prompt + 5 Skills, prompt duplication costs arise. `cache_control`-based Prompt Caching mitigates repetition costs — [`llm.py:103-108`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py#L103-L108).

**CAI** — Deep sequential:

A single active agent transitions sequentially, with the full history accumulating across agents. `/compact` compression mitigates history growth costs. However, parallel execution is also available via the `CAI_PARALLEL=N` environment variable — [`parallel_offensive_patterns.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/patterns/parallel_offensive_patterns.py).

| Aspect | Strix | CAI |
|--------|-------|-----|
| **Model** | Wide parallelism (Multi-Thread) | Deep sequential (sequential switching) |
| **Active agents** | 10-20 simultaneously | 1 at a time (default) |
| **Cost pattern** | Prompt duplication (system prompt × N) | History accumulation (gradual increase) |
| **Cost mitigation** | Prompt Caching (`cache_control`) | `/compact` compression |
| **Parallel option** | Default (Sub-Agent = Thread) | Activated with `CAI_PARALLEL=N` |

### 2.4 Insight: Why Multi-Agent Is Effective

| Aspect | Strix | CAI |
|--------|-------|-----|
| **Agent creation** | Dynamic (at runtime as needed) | Static (~20 pre-defined) |
| **Structure** | Root + N Sub-Agents (tree) | Agent pool + 5 patterns |
| **Concurrency** | Multi-Thread parallel | Sequential switching (only one active at a time) |
| **Agent Loop** | 1 tool per iteration, 300 limit, early streaming cutoff | N tools parallel per turn, Guardrail pipeline |

**Key Insights**:

1. **Role separation improves performance.**
- The IDOR-finding agent and the SQLi-finding agent load different specialized knowledge (Skill files). Putting all knowledge into a single vanilla LLM session dilutes the context.

2. **Dynamic creation vs pre-definition is a choice based on target predictability.**
- Why Strix creates agents at runtime: a web app's structure (endpoints, auth methods, tech stack) is only known after reconnaissance, so the needed specialists cannot be pre-defined.
- Why CAI pre-defines ~20: Kill Chain phases (Recon → Exploit → Privilege Escalation → C2) are consistent regardless of target, making role classification possible in advance.
- **Trade-off: Dynamic creation is target-tailored but incurs agent spawn costs, while pre-definition enables immediate deployment but may include unnecessary agents.**

3. **"1 tool per iteration" vs "N tools per turn" reflects different reasoning patterns.**
- Strix's 1-tool approach forces deliberate reasoning where each result is analyzed before deciding the next action — suitable for web apps where one response determines the next probe.
- CAI's N-tool parallel approach batch-processes independent reconnaissance commands (port scanning, service enumeration, etc.) — suitable for network reconnaissance where inter-probe dependencies are low.
- **Trade-off: The 1-tool approach is adaptive but slow, while N-tool parallel is fast but makes mid-course corrections based on intermediate results difficult.**

---

## 3. Tool Integration — Giving the LLM Hands and Feet

> Problem solved: Vanilla LLMs can only generate text and cannot actually execute tools

### 3.1 Strix Toolkit (13+ tools)

| Tool | Description | Key Contribution |
|------|-------------|------------------|
| **browser** | Playwright-based multi-tab browser automation (goto, click, type, scroll, JS execution) | JS-rendered SPA testing |
| **proxy** | HTTP Proxy - request/response manipulation, HTTPQL filtering, scope rules | Traffic analysis/tampering |
| **terminal** | tmux-based shell - PS1 pattern (`[STRIX_$?]$`) for exit code extraction, multi-session | Arbitrary command execution |
| **python** | IPython shell - Proxy functions auto-injected, stdout/stderr capture | Custom scripting |
| **agents_graph** | Sub-Agent creation, messaging, status monitoring | Multi-Agent orchestration |
| **file_edit** | Code/config file editing (openhands_aci integration) | Automated patching |
| **reporting** | CVSS v3.1 calculation, PoC code inclusion, remediation steps | Structured reports |
| **thinking** | Claude extended thinking integration | Extended reasoning |
| **web_search** | Perplexity API-based OSINT | External information gathering |

- **Tool registration**: Decorator-based Registry, XML Schema, DefusedXML security parsing (XXE prevention) — [`registry.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/registry.py)
- **Tool call format**: `<function=tool_name><parameter=name>value</parameter></function>` (XML)
- **Execution**: 1 per iteration, sequential

### 3.2 CAI Toolkit (24+ tools, Kill Chain based)

| Category | Key Tools | Key Contribution |
|----------|-----------|------------------|
| **Recon** | [`nmap.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/reconnaissance/nmap.py), [`shodan.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/reconnaissance/shodan.py), [`google_search.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/web/google_search.py) | Network/OSINT recon |
| **Execution** | [`generic_linux_command.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/reconnaissance/generic_linux_command.py) | Shell session management + command execution |
| **Analysis** | [`capture_traffic.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/network/capture_traffic.py), [`code_interpreter.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/misc/code_interpreter.py) | Traffic analysis/code execution |
| **Reasoning** | [`reasoning.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/misc/reasoning.py), [`rag.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/misc/rag.py) | Strategic analysis/memory integration |
| **C2** | [`command_and_control/`](https://github.com/aliasrobotics/cai/tree/e22a122/src/cai/tools/command_and_control) | Command and control channels |
| **Web** | [`js_surface_mapper.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/web/js_surface_mapper.py), [`webshell_suit.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/web/webshell_suit.py) | JS asset recon, Webshell |

- **Tool call format**: OpenAI function calling JSON Schema
- **Execution**: N per turn, parallel (`asyncio.gather`)
- Warning: 4 Kill Chain categories (`exploitation/`, `lateral_movement/`, `privilege_scalation/`, `data_exfiltration/`) have directories only, no implementations

### 3.3 Sandboxing: Tool Execution Environment

**Strix**: Docker container required — [`docker_runtime.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py)
- Kali Linux image — [L107](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py#L107)
- In-container Tool Server, port 48081 — [L24](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py#L24)
- Token auth `secrets.token_urlsafe(32)` — [L124](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py#L124)
- `NET_ADMIN`, `NET_RAW` capabilities — [L134](https://github.com/usestrix/strix/blob/5a76fab/strix/runtime/docker_runtime.py#L134)
- `pentester` user (not root)
- **Tool execution path**: Host (Agent logic) → HTTP POST → Container (tool execution) → JSON response

**CAI**: Docker optional — [`common.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/common.py) (1600+ line common execution engine)
- 3-way execution routing: Docker (`docker exec` + TTY), CTF (`ctf.get_shell()`), Host (PTY + `os.setsid()`)
- Remote execution via SSH (`paramiko`) also supported
- Shell session management (persistent sessions: S1, S2, ...) — [`generic_linux_command.py:70-115`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/tools/reconnaissance/generic_linux_command.py#L70-L115)
- Auto-kill process after 10 seconds of no output (idle detection)
- **Tool execution path**: Direct execution within the agent process (in-process)

### 3.4 Insight: Why Tool Integration Is Effective

| Aspect | Strix | CAI |
|--------|-------|-----|
| **Tool count** | 13+ | 24+ |
| **Tool Schema** | XML custom format | OpenAI function calling JSON |
| **Execution** | 1 per iteration, sequential | N per turn, parallel |
| **Sandbox** | Docker required (forced isolation) | Docker optional (flexibility first) |
| **Key differentiating tools** | Playwright browser, HTTP Proxy | Kill Chain tools (C2, SDR, PCAP) |

**Domain-specific Tool Comparison**:

| Domain | Strix | CAI |
|--------|-------|-----|
| Shell execution | tmux terminal (persistent session, PS1 exit code) | generic_linux_command + session management (S1, S2, ...) |
| Browser | Playwright multi-tab (screenshots → LLM vision) | None (curl, HTTP-level tools) |
| HTTP Proxy | Caido-based (HTTPQL filtering, request tampering) | None |
| Code execution | Python (IPython, Proxy functions auto-injected) | 14 languages + CodeAgent (CodeAct pattern) |
| Network recon | nmap (in container) | nmap + Shodan + Google Dorking |
| Webshell | None | webshell_suit |
| C2 | None | command_and_control |
| OSINT | Perplexity API | Perplexity + Google Custom Search |
| Reasoning | think (no side effects) | think + write_key_findings (persistent file storage) |
| Reporting | CVSS calculation + LLM deduplication check | Reporter Agent (HTML reports) |

**Key Insights**:

1. **PoC execution reduces false positives.**
- Instead of "possible SQLi", it shows data actually extracted with sqlmap. Vanilla LLMs can only speculate.

2. **Tool Schema controls LLM hallucination.**
- Whether XML or JSON, a structured Schema forces the LLM to make only "correctly formatted tool calls." Vanilla LLMs just explain in free text "you could do this."

3. **Tool scope determines coverage.**
- Strix secures web app depth with Playwright/Proxy, while CAI secures breadth across the entire Kill Chain (Recon → C2) with its tooling.

---

## 4. Handoff — Inter-Agent Collaboration Structure

> Problem solved: Recon results must be accurately passed to the exploitation agent

### 4.1 Strix: Hierarchical Delegation

**Mechanism**: Root Agent dynamically creates Sub-Agents as **daemon threads**

```
Root Agent (StrixAgent)
  ├── create_agent() → SQLi Validation Agent [Thread]
  ├── create_agent() → XSS Testing Agent [Thread]
  └── create_agent() → IDOR Discovery Agent [Thread]
       └── create_agent() → IDOR Reporting Agent [Thread]
```

**Core Implementation** ([`agents_graph_actions.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py)):
- `create_agent()` — [L188-281](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py#L188-L281): Creates Sub-Agent in a new thread
- `send_message_to_agent()` — [L285-352](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py#L285-L352): Async message queue (type: query/instruction/information, priority: low~urgent)
- `agent_finish()` — [L356-384](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py#L356-L384): Sends XML Report to parent upon completion
- **Context inheritance**: `inherit_context=True` ([L192](https://github.com/usestrix/strix/blob/5a76fab/strix/tools/agents_graph/agents_graph_actions.py#L192)) passes context via `<inherited_context_from_parent>` tag
- **Identity isolation**: "You are NOT your parent agent. You are a NEW, SEPARATE sub-agent" injected
- **Shared resources**: All agents share the same `/workspace` and Proxy history
- **Message format**: XML structure (`<inter_agent_message>`, `<agent_completion_report>`)

### 4.2 CAI: Swarm-Based Transfer (Peer-to-Peer)

**Mechanism**: **Bidirectional handoff** for control transfer between agents

```
Thought Agent ←→ Red Team Agent ←→ DNS/SMTP Agent
                    ↕
              Bug Bounty Agent
```

**Core Implementation** ([`handoffs.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/sdk/agents/handoffs.py)):
- `handoff()` — [L150-236](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/sdk/agents/handoffs.py#L150-L236): Registers handoff as a tool (e.g., `transfer_to_redteam_agent`)
- `Handoff` class — [L53-115](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/sdk/agents/handoffs.py#L53-L115): Handoff definition
- When the LLM calls a handoff tool, control transfers to the target agent
- **Message history sharing**: Entire conversation history is passed to the new agent on handoff
- **Agent cloning**: `agent.clone()` creates independent copies — [`red_team.py:17-19`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/patterns/red_team.py#L17-L19)

**Pattern Example** ([`red_team.py:44-46`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/patterns/red_team.py#L44-L46)):
```python
# Bidirectional edge construction
redteam_agent.handoffs.append(dns_smtp_handoff)    # Red → DNS
dns_smtp_agent.handoffs.append(redteam_handoff)    # DNS → Red
thought_agent.handoffs.append(redteam_handoff)     # Thought → Red
```

### 4.3 Insight: Why Handoff Is Effective

| Aspect | Strix | CAI |
|--------|-------|-----|
| **Structure** | Tree (parent → child, unidirectional) | Graph (P2P, bidirectional) |
| **Creation method** | Dynamic (created at runtime as needed) | Static (pre-defined agent pool) |
| **Concurrency** | Multi-thread parallel execution | Sequential switching (only one active at a time) |
| **Communication** | Async message queue (XML) | Full conversation history transfer |
| **Context** | Selective inheritance + identity isolation | Full history sharing |
| **Control authority** | Parent agent decides | LLM decides autonomously (tool selection) |
| **Return** | `agent_finish()` → report to parent | Reverse handoff to return |
| **Scalability** | Dozens of Sub-Agents can run in parallel | Agent count = number of pre-registered handoffs |

**Key Insights**:

1. **Handoff = structured information transfer.**
- In vanilla LLM conversations, referencing "from the earlier nmap results..." loses accuracy. Strix prevents information loss via XML Reports, CAI via full history transfer.

2. **Tree vs graph solves different problems.**
- Strix's tree structure is optimized for "attacking a single web app from multiple angles simultaneously," while CAI's graph structure is optimized for "organic transitions between Kill Chain stages."

3. **Context transfer method determines cost.**
- Strix's "selective inheritance" saves tokens by passing only needed information, while CAI's "full history sharing" transfers without information loss but accumulates tokens.

---

## 5. Memory and Learning — Accumulating Experience

> Problem solved: LLM context windows are finite, and everything is forgotten when a session ends

### 5.1 Strix: Security-Aware Conversation Compression

**MemoryCompressor** — [`memory_compressor.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py):
- Summarizes early messages when conversation history **exceeds 100K tokens** — [L12](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py#L12) (`MAX_TOTAL_TOKENS = 100_000`)
- LLM-based summarization (10-message chunks → `<context_summary>` tags) — [L172-225](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py#L172-L225)
- Most recent 15 messages kept as-is — [L13](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/memory_compressor.py#L13) (`MIN_RECENT_MESSAGES = 15`)
- **Security-aware compression**: Vulnerabilities, credentials, system architecture, and failed attempts are **prioritized for preservation**
- Image handling: only 3 most recent kept, rest replaced with text references
- **No cross-session learning** — Each scan is independent

### 5.2 CAI: Vector DB-Based Cross-Session Learning

**Episodic + Semantic + RAG** — [`memory.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/memory.py):
- **Episodic**: Chronological per-target records stored in Qdrant Vector DB — [L14-18](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/memory.py#L14-L18)
- **Semantic**: All target data vectorized in a single global collection — [L20-24](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/memory.py#L20-L24)
- **RAG Pipeline**: LLM summarization → Vector embedding → Similarity search
- **Cross-session learning**: Leverages previous penetration test experience in the next session — [L36-40](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/memory.py#L36-L40) (Online Learning)
- **Online/Offline modes**: Real-time updates or batch learning
- Warning: **Code issues**: Undefined variable `model_name` in `memory.py` (L200, 215, 229), `query_agent` has no tools defined but `tool_choice="required"` is set

### 5.3 Vulnerability Deduplication

**Strix**: LLM-based automatic deduplication — [`dedupe.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/dedupe.py)
- `check_duplicate()` — [L141-217](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/dedupe.py#L141-L217): Compares new vulnerability against existing reports via LLM
- Same root cause, same component, same attack vector = duplicate
- Same type on different endpoints = not duplicate

**CAI**: No automatic deduplication
- Reporter Agent handles manual cleanup
- Only flag discriminator for CTF flag identification exists

### 5.4 Insight: Why Memory Is Effective

| Aspect | Strix | CAI |
|--------|-------|-----|
| **Intra-session** | LLM compression when exceeding 100K tokens (security info prioritized) | Same principle |
| **Cross-session** | None (each scan independent) | Qdrant Vector DB (Episodic + Semantic) |
| **Deduplication** | LLM-based automatic | None (manual) |

**Key Insights**:

1. **"Security-aware" compression is key.**
- General conversation summarization compresses all content equally, but Strix prioritizes preserving vulnerabilities/credentials. This is the difference from simple truncation.

2. **Cross-session learning has great value in repeated testing.**
- When periodically scanning the same target, CAI remembers "last time we found this vulnerability on this port."
- Strix starts from scratch every time.

3. **Deduplication determines report quality.**
- Strix's LLM-based deduplication check distinguishes "same SQLi found on different parameters."
- CAI lacks this feature and requires manual cleanup.

---

## 6. Safety Controls — Managing Hallucination, False Positives, and Cost

> Problem solved: LLMs hallucinate, costs can spiral, and dangerous actions may occur

### 6.1 Guardrails

**Strix**: No guardrails
- No Prompt Injection defense
- If the target web app returns malicious responses, the agent may be affected

**CAI**: 3-layer defense — [`guardrails.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/guardrails.py), [`sdk/guardrail.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/sdk/agents/guardrail.py)
1. **Input Guardrail**: Pattern matching, Unicode Homoglyph detection ([L81-131](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/guardrails.py#L81-L131)), Base64/32 decoding
2. **Output Guardrail**: Dangerous command detection (reverse shells, fork bombs), data exfiltration prevention
3. **Detection Patterns** (~15 core regex + Unicode/Base64/32 checks) — [L40-78](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/agents/guardrails.py#L40-L78): Instruction override, hidden XML tags, encoding tricks, role manipulation
4. Configuration: `CAI_GUARDRAILS=true/false`, Tripwire — [`guardrail.py:29-32`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/sdk/agents/guardrail.py#L29-L32)

### 6.2 Cost Control

**Strix** — [`llm.py`](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py):
- `litellm.completion_cost()` call — [L252](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py#L252)
- `RequestStats` class (input/output tokens, cost) — [L42-57](https://github.com/usestrix/strix/blob/5a76fab/strix/llm/llm.py#L42-L57)
- **No cost limit** — User must monitor manually
- Streaming halts at first `</function>` tag → saves output tokens

**CAI** — [`util.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/util.py):
- `CostTracker` class: 3-tier cost tracking (Session/Agent/Interaction) — [L392-440](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/util.py#L392-L440)
- `CAI_PRICE_LIMIT` — [L422-426](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/util.py#L422-L426): **Auto-stop** when dollar limit exceeded
- `GlobalUsageTracker` — [`global_usage_tracker.py`](https://github.com/aliasrobotics/cai/blob/e22a122/src/cai/sdk/agents/global_usage_tracker.py): File-lock based global usage tracking (`~/.cai/usage.json`)

**Theoretical Cost Structure Comparison**:

| Cost Factor | Strix | CAI |
|-------------|-------|-----|
| **Per-agent overhead** | High (407-line system prompt + 5 skills injected each time) | Medium (61-191 line per-agent prompt) |
| **Tool call efficiency** | 1 per iteration → LLM call per tool result | N per turn → multiple results processed at once |
| **Sub-Agent** | Dozens parallel → each independent LLM session (cost × N) | Swarm → sequential switching (history accumulation) |
| **Cost pattern** | **Wide parallel cost** (system prompt repetition) | **Deep sequential cost** (gradual history increase) |

### 6.3 False Positives and Detection Quality

**Strix ("GO SUPER HARD" approach)**:
- Strengths: "2000+ steps", "$500+ Bug Bounty value" emphasis → deep probing, LLM deduplication + PoC verification
- Risks: No guardrails → vulnerable to Prompt Injection, aggressive payloads → possible target side effects

**CAI ("Safe First" approach)**:
- Strengths: 3-layer guardrails, "read-only first", HITL, memory learns from previous false positives
- Risks: Conservative → may miss deeply hidden vulnerabilities, guardrail over-detection possible

**Theoretical Detection Quality**:

| Aspect | Strix | CAI |
|--------|-------|-----|
| **True Positive** | High (aggressive + mandatory PoC verification) | Medium (systematic but conservative) |
| **False Positive** | Low (Validation Agent + LLM deduplication) | Medium (no auto-deduplication) |
| **False Negative** | Low (2000+ steps enforced) | Medium (safe-first may cause some false negatives) |
| **Target safety** | Medium (claims non-destructive, no guarantee) | High (read-only + HITL) |

> Warning: This is a theoretical analysis. Actual comparison requires experiments on identical targets such as DVWA.

### 6.4 Insight: Safety Control Trade-offs

**Key Insights**:

1. **Guardrail vs attack effectiveness is a trade-off.**
- CAI's guardrails defend against Prompt Injection, but may also block legitimate attack payloads.
- Strix maximizes attack effectiveness without guardrails, but is vulnerable to malicious responses from targets.

2. **PoC verification is an effective method for reducing false positives.**
- Strix's Validation Agent actually executes discovered vulnerabilities to confirm them.
- CAI lacks this automatic verification loop.

3. **Cost control is essential in production environments.**
- Strix can spawn Sub-Agents indefinitely with no cost limit.
- CAI's `CAI_PRICE_LIMIT` prevents accidental cost runaway.

---

## 7. Comprehensive Comparison

### Tool Overview

| Item | Strix (v0.7.0) | CAI (v0.5.10) |
|------|----------------|---------------|
| **Position** | DevSecOps web app automation | Comprehensive cybersecurity framework |
| **Development** | usestrix (community) | Alias Robotics (enterprise) |
| **License** | Apache-2.0 (commercial free) | Open Core (commercial €350/month) |
| **Python** | 3.12+ | 3.9+ |
| **SDK** | Custom-built | OpenAI Agents SDK fork |
| **LLM abstraction** | LiteLLM ~1.81.1 | OpenAI SDK 1.75.0 + LiteLLM |
| **GitHub Stars** | ~19.5K | ~6.9K |

### Architecture Comparison

| Domain | Strix | CAI |
|--------|-------|-----|
| **Template** | Jinja2 (~407 lines) | Mako (~190 lines) |
| **Tone** | Aggressive/fully autonomous | Systematic/safety-first |
| **Agent creation** | Dynamic (runtime) | Static (~20 pre-defined) |
| **Structure** | Tree (parent → child) | Graph (P2P bidirectional) |
| **Concurrency** | Multi-Thread parallel | Sequential switching (Parallel option) |
| **Agent Loop** | 1 tool/iteration, 300 limit | N tools/turn parallel, built-in guardrails |
| **Tool Schema** | XML custom | OpenAI function calling JSON |
| **Key tools** | Playwright, Caido Proxy | Session management, CodeAgent, C2 |
| **Sandbox** | Docker required | Docker optional (4-environment auto-routing) |
| **Cross-session memory** | None | Qdrant Vector DB |
| **Deduplication** | LLM automatic | None |
| **Guardrails** | None | 3-layer (Pattern + Unicode + AI) |
| **Cost limit** | None | `CAI_PRICE_LIMIT` auto-stop |
| **Model specification** | Unified across all | Per-agent individual specification |

### Security Testing Coverage

| Domain | Strix | CAI |
|--------|:-----:|:---:|
| Web App / API / SAST | ✅ | ✅ |
| Browser Automation (Playwright) | ✅ | ❌ |
| HTTP Proxy (HTTPQL) | ✅ | ❌ |
| Network Penetration Testing | ⚠️ (basic) | ✅ |
| Binary/Reversing | ❌ | ✅ |
| Mobile (Android) | ❌ | ✅ |
| IoT/Embedded/Wireless | ❌ | ✅ |
| Forensics/Memory/PCAP | ❌ | ✅ |
| Benchmarks / CTF Auto-solve | ✅ ([XBEN](https://github.com/usestrix/benchmarks/tree/main/XBEN) 104 challenges, 96%) | ✅ (CAIBench 100+ challenges) |

**Summary**: Strix = Web app depth, CAI = Security domain breadth

---

## Appendix A: License Essentials

- **Strix**: Apache-2.0 → **Free** for personal/commercial use
- **CAI**: Open Core → Free for research/learning, **CAI PRO €350/month** required for commercial use
- CAI SDK code (`src/cai/agents/`) is MIT, remaining Alias Robotics code is Research-Use Proprietary

## Appendix B: Selection Guide

| Use Case | Recommendation | Reason |
|----------|:--------------:|--------|
| Web app CI/CD security automation | Strix | GitHub Actions, headless, exit code |
| Automated code patching & PR | Strix | file_edit + verification loop |
| Commercial product integration | Strix | Apache-2.0 free |
| Comprehensive pentest (network + web) | CAI | Kill Chain based, rich network tools |
| CTF/Security education | CAI | CAIBench 100+ challenges |
| Forensics/Incident response | CAI | DFIR, Memory/PCAP analysis |
| IoT/Mobile security | CAI | Android SAST, Sub-GHz SDR |

## Appendix C: Future Research Directions

- [ ] Quantitative comparison of technique effectiveness on identical targets (e.g., DVWA)
- [ ] Performance analysis by LLM model (GPT-5, Claude, alias1)
- [ ] Cost efficiency comparison (tokens/cost to discover identical vulnerabilities)
- [ ] Hybrid design: Strix's parallel agents + auto-patching + CAI's memory/guardrails
- [ ] Measuring impact of CAI's cross-session learning on repeated scan efficiency
- [ ] Risk assessment of Strix's lack of guardrails in real attack scenarios
