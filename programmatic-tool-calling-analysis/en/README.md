# Programmatic Tool Calling vs Traditional Tool Calling: Who Receives the Result?

> The cost of Tool Calling lies in "where the result goes"

---

## 1. Background: The Evolution of Tool Calling

### 1.1 Traditional Tool Calling

The most basic pattern for LLMs to use tools:

```text
User → LLM → {"name": "query_db", "input": {"sql": "..."}}
                                    ↓
                              Tool Execution
                                    ↓
                                 Result → LLM → Final Response
```

The LLM generates JSON-formatted arguments, the tool executes on the developer's server, and the result is fed back into the LLM's context.

### 1.2 Parallel Tool Calling

Multiple tools are called in parallel to reduce the **number of round-trips**:

```text
User → LLM → [tool_use A, tool_use B, tool_use C]  (parallel calls in one shot)
                        ↓
             [Tool Execution A, B, C]
                        ↓
             [result A, result B, result C] → LLM → Final Response
```

While round-trips are reduced, **the structure where all results enter the LLM context remains the same**.

### 1.3 CodeAct (Wang et al., 2024, ICML)

Traditional LLM Agents generate actions in JSON or predefined text formats. CodeAct addresses the fundamental limitations of this approach (constrained action space & restricted flexibility) by **using executable Python code as a unified action space**:

```text
Traditional:  LLM → {"tool": "search", "query": "..."}     (JSON action)
CodeAct:      LLM → result = search("..."); print(len(result))  (Code action)
```

In experiments across 17 LLMs, CodeAct achieved higher success rates than traditional JSON/Text approaches. The best-performing model, GPT-4-1106-preview, achieved a **success rate approximately 20 percentage points higher** than Text (74.4% vs 53.7%). Section 6 provides a detailed analysis of where this performance improvement comes from.

### 1.4 Programmatic Tool Calling (PTC)

Anthropic introduced PTC, which embodies the concepts of CodeAct, into Claude.

```text
User → LLM → code_execution("""
    result = await query_db(...)
    total = sum(r['revenue'] for r in result)
    print(f"Total revenue: {total}")
""")
```

The basic architecture is the same as CodeAct. Code is generated, executed in a Python runtime, and the code execution result (stdout/stderr) is returned to the LLM.

What PTC adds as a product are **engineering elements** like `allowed_callers` (tool invocation permission control), container lifecycle management, and API design. It can be seen as the productization of CodeAct.

---

## 2. The Key Question: Is PTC Really Advantageous?

The commonly cited advantages of PTC include:

- Repeated calls via for loops → token savings
- Fewer round-trips → reduced latency

But this raises a question:

> "For complex tools, traditional tool calling only needs to pass arguments, whereas PTC has to generate invocation code every time — isn't that less reliable?"

This concern is valid. And to resolve it, you need to understand precisely **where the tool result goes**.

---

## 3. Who Receives the Result?

### 3.1 A Common Misconception

What Claude generates in PTC is **not the tool's implementation code but the invocation code**. The actual tool implementation still runs deterministically on the developer's server. This is the same as traditional tool calling.

```text
Traditional:  LLM → {"sql": "SELECT ..."}  → Server Execution → Result
PTC:          LLM → result = await query_db("SELECT ...") → Server Execution → Result
```

**Up through tool execution, they are identical. The difference lies in what happens after.**

### 3.2 The Decisive Difference: Who Receives the Result

| | Traditional Tool Calling | PTC |
|---|---|---|
| **Who receives the tool result** | LLM (inference engine) | Python runtime (container) |
| **What the LLM sees** | The entire raw result | Only the code execution result (stdout/stderr) |
| **How the result is processed** | LLM reads and reasons token by token | Python processes it as variables |

Illustrated:

**Traditional Tool Calling:**

```text
func1(a) execution → 10,000-row result ──────────────► LLM context window
                                                        (reads 10,000 rows as tokens)
```

**PTC:**

```text
func1(a) execution → 10,000-row result → Python variable (result)
                                              │
                                              ▼
                                        Python processes it
                                        (sort, filter, sum...)
                                              │
                                              ▼
                                  Code execution result: "Total revenue: $2.95M"
                                              │
                                              ▼
                                      LLM context window
                                      (reads only the processed result)
```

> Anthropic official documentation: "Tool results from programmatic invocations do not count toward your input/output token usage. Only the final code execution result and Claude's response count."

---

## 4. Comparison with Parallel Tool Calling

The natural follow-up question is: "How is this different from Parallel Tool Calling?" Both reduce round-trips, but the key difference is **how results accumulate in the context**.

### 4.1 Flow of Parallel Tool Calling

```text
LLM: [tool_use A, tool_use B, tool_use C]
       ↓            ↓            ↓
  Execution A   Execution B   Execution C
       ↓            ↓            ↓
  result A      result B     result C
       └────────────┴────────────┘
                    ↓
            All go into LLM context ✅
                    ↓
              LLM: Final Response
```

### 4.2 Flow of PTC

```text
LLM: code_execution("""
    result_a = await tool_a(...)
    result_b = await tool_b(...)
    result_c = await tool_c(...)
    print(summary)
""")
       ↓            ↓            ↓
  Execution A   Execution B   Execution C
       ↓            ↓            ↓
  Python var    Python var   Python var    ← LLM cannot see these ❌
       └────────────┴────────────┘
                    ↓
          Python processes, then print()
                    ↓
        Only code execution result enters LLM context ✅
                    ↓
              LLM: Final Response
```

The actual execution mechanism is more complex than the diagram above. At each `await`, code execution pauses and the API returns a `tool_use` block to the developer. Once the developer executes the tool and sends the result back to the API, code execution resumes. But the core point remains the same: **tool results go into Python variables, and the LLM only sees the final code execution output.**

### 4.3 Concrete Cost Comparison

Consider calling a tool that returns 10,000 rows from a DB for 3 regions:

| | Traditional (Sequential) | Parallel | PTC |
|---|---|---|---|
| Round-trips | 3 | 1 | 1 |
| Data entering LLM context | 30,000 rows (cumulative) | 30,000 rows | Only code execution result |
| Intermediate re-reasoning needed | Every call | None | None |
| Input token cost | Very high | High | Low |

**The decisive difference between Parallel and PTC is not the round-trips — it's whether tool results enter the LLM context.**

---

## 5. "Can't You Just Design Better Tools?"

Another reasonable question arises here:

> "If you design the tool to return only filtered results from the start, can't you get the same effect without PTC?"

Yes. Carefully designing tool responses can reduce the context burden even with the traditional approach. But there are practical limitations:

### 5.1 Tools Are Designed to Be General-Purpose

```text
query_db("SELECT * FROM sales WHERE region = 'West'")
```

How the result of this tool should be processed **differs with every call**:

| Scenario | Required Processing |
|------|------------|
| "Tell me total revenue" | `sum(revenue)` |
| "Top 5 customers" | `sorted(...)[:5]` |
| "Monthly trend" | `groupby(month)` |
| "Outlier detection" | `[r for r in result if r['revenue'] > threshold]` |

If you embed all this processing logic into the tool itself, the tool's parameters become complex, and you need to modify the tool every time a new requirement arises.

### 5.2 The Value of PTC: LLMs Flexibly Generate Processing Logic

In PTC, **"how to process" is decided by the LLM as code each time**:

```python
# Code generated by the LLM to fit the situation
result = await query_db("SELECT * FROM sales WHERE region = 'West'")

# This time, total revenue is needed:
total = sum(r['revenue'] for r in result)
print(f"Total revenue: ${total:,.0f}")
```

```python
# Next time, top 5 are needed:
result = await query_db("SELECT * FROM sales WHERE region = 'West'")
top5 = sorted(result, key=lambda x: x['revenue'], reverse=True)[:5]
for r in top5:
    print(f"{r['customer']}: ${r['revenue']:,.0f}")
```

**Keep the tool general-purpose and let the LLM flexibly handle processing through code.**

---

## 6. What Drives the Performance Improvement?

The CodeAct paper presents four advantages of code actions:

| Advantage | Description |
|------|------|
| **(1) Dynamic adjustment** | Observe execution results through multi-turn interaction with the Python interpreter, then modify previous actions or generate new ones |
| **(2) Existing software packages** | Directly leverage existing PyPI packages to expand the action space |
| **(3) Code familiarity** | LLMs are familiar with code generation because they encountered large amounts of code during pretraining |
| **(4) Control and data flow** | Combine multiple tools and reuse intermediate results through loops, conditionals, and variable storage |

### 6.1 The Paper's Experimental Structure

The paper uses two experiments to confirm the contribution of these advantages.

**§2.2 (API-Bank)** — Tests only atomic tool use (single tool calls). The primary goal is to verify RQ1: "Does code familiarity (advantage 3) help?" Because only single calls are tested, the effect of control and data flow (advantage 4) is naturally eliminated. The paper explains this in the Introduction:

> "To demonstrate benefit (3), our first experiment (§2.2) compares CodeAct to baselines on basic tasks involving atomic tool use (i.e., only one tool is used per action), ablating the control and data flow advantage offered by CodeAct."

Result: According to the paper, performance was at a **comparable** level. This shows that code familiarity alone does not create a meaningful difference.

**§2.3 (M3ToolEval)** — Tests complex multi-tool composition tasks. Control and data flow (advantage 4) is included. Result: Approximately 20 percentage points higher success rate than Text for GPT-4-1106-preview, with an average of 2.2 fewer turns of interaction.

Contrasting the two experiments:

| Experiment | Control and data flow | Result |
|------|----------------------|------|
| §2.2 (API-Bank) | Ablated (single calls) | Comparable |
| §2.3 (M3ToolEval) | Included (multi-tool) | Large performance improvement |

What gets added going from §2.2 to §2.3 is control and data flow, and that is where the performance gap widens significantly. **The paper demonstrates through this that control and data flow is the key factor behind CodeAct's performance improvement.**

### 6.2 From a Context Efficiency Perspective

> From this point on, this is **my interpretation**, not a claim from the paper.

The paper describes the benefit of control and data flow as "the ability to combine multiple tools and reuse intermediate results by storing them in variables." Terms like "context efficiency" or "context engineering" do not appear in the paper.

However, when we re-examine this mechanism from a context perspective:

```text
Control flow (loops, conditionals):
  JSON:    5 round-trips → previous results accumulate in context each turn
  CodeAct: Processed in 1 turn with a for loop → no accumulation at all

Data flow (variable storage, reuse):
  JSON:    Entire tool result enters LLM context
  CodeAct: Stored in variables, only code execution result is passed
```

**Ultimately, what control and data flow does is reduce the information entering the LLM's context.** However, in early 2024, the "context engineering" perspective was not as widespread as it is today, which is why the word "context" is hard to find in the paper. The concept existed, but it didn't yet have a name.

---

## 7. (My Thoughts) Trade-offs and Implications

### 7.1 When PTC Is Advantageous

| Condition | Reason |
|------|------|
| **Multiple tool calls (3+)** | Eliminates round-trips + prevents context accumulation |
| **Many independent parallel tasks** | e.g., checking 50 endpoints — the more calls, the greater the round-trip savings |
| **Large tool results** | Filter large datasets with code and pass only a summary |
| **Conditional logic** | Determine the next call based on intermediate results |
| **Early termination** | Stop immediately when a condition is met |

### 7.2 When Traditional Tool Calling Is Advantageous

| Condition | Reason |
|------|------|
| **Single tool call** | Only adds code execution environment overhead |
| **Small, simple tool results** | Minimal context savings |
| **Immediate user feedback needed** | PTC responds only after code completes |
| **Models with weak code generation** | Code quality directly affects reliability |

Note that most current open/closed models are fine-tuned for each provider's tool calling format. While models optimized for the PTC approach may emerge in the future, this is a factor to consider at the present time. In particular, for smaller-parameter models, the PTC approach may be less reliable than traditional tool calling.

### 7.3 Considerations on Reliability

There may be concerns that "generated code has a higher failure rate than JSON arguments":

```text
Traditional: {"sql": "SELECT ..."}          ← Simple structure, fewer failures
PTC:         result = await query_db(...)    ← The entire code must be correct
             filtered = [r for r in result if ...]
             print(summary)
```

However, the code generated in PTC mostly consists of simple Python patterns like for loops, list comprehensions, and conditionals, and errors are returned via stderr allowing retries. As models continue to improve, code reliability is unlikely to remain a significant barrier.

### 7.4 Parallels with ACE-FCA

ACE-FCA, analyzed in a [previous article](../../ace-fca-agentflow-analysis/en/README.md), follows the same principle:

```text
ACE-FCA:      [Entire research process] → Bottleneck → research.md (compressed result)
CodeAct/PTC:  [Entire tool result]      → Bottleneck → Code execution result (compressed result)
```

**Create a bottleneck in the information flow to discard noise and pass only the signal.** The level of application differs, but the principle is the same:

| | ACE-FCA | CodeAct/PTC |
|---|---|---|
| **What is compressed** | Entire phase context | Tool result |
| **Where compression happens** | Sub-agent → artifact | Python runtime → code execution result |
| **Who receives it** | LLM in the next phase | LLM in the same turn |
| **Level** | Macro (project) | Micro (single turn) |

The insight that runs through these approaches is:

> **"The LLM's context window is an expensive resource. Minimize the information that enters it."**

---

## 8. Conclusion

```text
Traditional Tool Calling:
  Tool result → LLM reads it directly
  → Context cost spikes when results are large

Parallel Tool Calling:
  Tool result → LLM reads it directly (only reduces round-trips)
  → Same context cost problem

CodeAct/PTC:
  Tool result → Python receives and processes it → Only code execution result passed to LLM
  → Dramatically reduced context cost
```

> The essence of PTC is **"changing the recipient of tool results from the LLM to the Python runtime."**

| Scenario | Recommended Approach |
|------|----------|
| Simple single call, small result | Traditional Tool Calling |
| Independent multiple calls, high volume | **PTC** |
| Multiple calls + large results + processing needed | **PTC** |
| Conditional logic / loops / early exit | **PTC** |

---

## References

- [CodeAct: Executable Code Actions Elicit Better LLM Agents (ICML 2024)](https://arxiv.org/abs/2402.01030)
- [CodeAct GitHub Repository](https://github.com/xingyaoww/code-act)
- [Anthropic: Programmatic Tool Calling](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling)
- [Anthropic: Code Execution Tool](https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool)
- [Anthropic: Tool Use Overview](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
