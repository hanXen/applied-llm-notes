# Context Management Strategies for AI Agents: Lessons from ACE-FCA and AgentFlow

> Analyzing two approaches for effectively utilizing AI Agents in large codebases

---

## 1. Introduction to Two Studies

### 1.1 ACE-FCA (Advanced Context Engineering for Coding Agents)

**Source**: HumanLayer (Practical Guide)
**Target**: AI Coding Agent utilization in large codebases (300k+ LOC)

**Background Problems**:
- AI Coding Agents don't work effectively in large existing codebases
- Context window depletes quickly when handling complex tasks in a single turn
- Research phase alone consumes 60-80% of context

**Core Solution: Frequent Intentional Compaction (FIC)**

Separate the project into three phases, compressing each phase's results into artifacts:

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Research   │───►│     Plan     │───►│  Implement   │
│    Phase     │    │    Phase     │    │    Phase     │
└──────┬───────┘    └──────┬───────┘    └──────┬───────┘
       │                   │                   │
       ▼                   ▼                   ▼
   research.md          plan.md              code
  (~200 lines)        (~200 lines)
```

**Role of Each Phase**:

| Phase | Purpose | Output |
|-------|---------|--------|
| Research | Understand codebase, identify relevant files | research.md |
| Plan | Establish concrete implementation plan | plan.md |
| Implement | Write code according to plan | code + tests |

**Sub-agent Utilization**:
- Spawn sub-agents for tool calling within each phase
- Sub-agents return only results after task completion, discarding context
- Prevents parent context pollution

---

### 1.2 AgentFlow (In-the-Flow Agentic System Optimization)

**Source**: Stanford (Academic Research, ICLR 2026)
**Target**: Improving reasoning quality of tool-augmented LLMs

**Background Problems**:
- Existing approaches use a single monolithic policy that interleaves thinking and tool calls in one context
- Difficult to scale on long horizon tasks, weak generalization to new scenarios

**Core Solution: Modular Agentic System**

Separate one task into four specialized modules:

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Planner  │───►│ Executor │───►│ Verifier │───►│Generator │
│          │    │          │    │          │    │          │
│ Decide   │    │ Execute  │    │ Is result│    │ Generate │
│ next step│    │ tool     │    │ enough?  │    │ answer   │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
      │               │               │               │
      └───────────────┴───────────────┴───────────────┘
                            │
                      Shared Memory
                 (State storage/sharing)
```

**Role of Each Module**:

| Module | Role | Characteristics |
|--------|------|-----------------|
| Planner | Decide next action (tool selection + context) | Training target (Flow-GRPO) |
| Executor | Execute tool | Frozen |
| Verifier | Verify results, determine STOP/CONTINUE | Frozen |
| Generator | Generate final answer | Frozen |

**Memory Structure** (Based on actual code):
```python
class Memory:
    def __init__(self):
        self.query = None           # Original question
        self.files = []             # Related files
        self.actions = {}           # Action records per turn

    def add_action(self, step_count, tool_name, sub_goal, command, result):
        self.actions[f"Action Step {step_count}"] = {
            'tool_name': tool_name,
            'sub_goal': sub_goal,
            'command': command,
            'result': result,
        }
```

**Execution Limits**:
- `max_steps = 10` (default)
- `max_time = 300 seconds` (5 minutes)

**Key Results**:
- 7B model + AgentFlow outperforms GPT-4o on some benchmarks
- Training only the Planner improves overall performance

---

## 2. Commonalities

### 2.1 Fundamental Principle

> **"Don't handle complex things all at once; decompose into specialized stages"**

Both approaches recognize the **limitations of monolithic processing** and solve it through **stage-wise decomposition**.

### 2.2 Specific Commonalities

| Aspect | ACE-FCA | AgentFlow |
|--------|---------|-----------|
| **Decomposition** | Project → Phases | Task → Modules |
| **Specialization** | Each Phase has unique role | Each Module has unique role |
| **Information Flow Control** | Compressed transfer via artifacts | Store only results in Memory |
| **Context Pollution Prevention** | Reset context between Phases | Return only sub-agent results |

### 2.3 Key Insight

The value of both approaches lies not in simple "decomposition" but in **creating bottlenecks in information flow**:

```
[Raw context of entire process] → [Bottleneck] → [Compressed result]

ACE-FCA:  Entire Research process → Bottleneck → research.md
AgentFlow: Tool execution process → Memory → Store only results
```

This bottleneck discards noise and transmits only signal.

---

## 3. Differences

### 3.1 Application Level

| | ACE-FCA | AgentFlow |
|---|---------|-----------|
| **Level** | Macro (Project) | Micro (Task) |
| **Scale** | Tens to hundreds of tool calls | Max 10 steps |
| **Duration** | Hours to days | Minutes |

### 3.2 Core Concerns

```
ACE-FCA:  "How to manage context?"
          └─► Goal: Prevent context explosion

AgentFlow: "How to execute tasks well?"
          └─► Goal: Improve execution quality
```

### 3.3 Information Transfer Method

| | ACE-FCA | AgentFlow |
|---|---------|-----------|
| **Medium** | Files (.md artifacts) | Objects (Memory) |
| **Persistence** | Permanent (saved as files) | Volatile (deleted on session end) |
| **Readability** | Human-readable and reviewable | Internal state |

### 3.4 Human Involvement

| | ACE-FCA | AgentFlow |
|---|---------|-----------|
| **Review Points** | Human reviews at Phase completion | None (automatic) |
| **Verification Agent** | Human | Verifier module |
| **Philosophy** | Human-in-the-loop | Fully automated |

### 3.5 Horizon Comparison

```
ACE-FCA Phase:
├── Task 1 (tens of tool calls)
├── Task 2 (tens of tool calls)
├── Task 3 (tens of tool calls)
└── ... (hundreds of tool calls possible)

AgentFlow:
├── Step 1
├── Step 2
├── ...
└── Step 10 (max)
```

**→ Different scales**

---

## 4. Integration Strategy (My Thoughts)

Here's my integration strategy based on reviewing both studies.

### 4.1 Integrated Structure

```
┌─────────────────────────────────────────────────────────────────┐
│  ACE-FCA (Macro Level) - Project Flow Management                │
│                                                                 │
│  ┌───────────┐      ┌───────────┐      ┌───────────┐            │
│  │  Research │ ──►  │   Plan    │ ──►  │ Implement │            │
│  └─────┬─────┘      └─────┬─────┘      └─────┬─────┘            │
│        │                  │                  │                  │
│        ▼                  ▼                  ▼                  │
│   research.md         plan.md             code                  │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Inside Each Phase                                              │
│                                                                 │
│  Task 1 ─► Sub-agent ─► Return only results (discard context)   │
│  Task 2 ─► Sub-agent ─► Return only results (discard context)   │
│  Task 3 ─► Sub-agent ─► Return only results (discard context)   │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Inside Sub-agent (Micro Level) - Choose based on conditions    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Option A: Strong model + Easy problem                   │    │
│  │                                                         │    │
│  │   Claude/GPT-4 ──► Direct processing ──► Return result  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ Option B: Weak model or Difficult problem               │    │
│  │                                                         │    │
│  │   AgentFlow structure:                                  │    │
│  │   Planner → Executor → Verifier → Generator             │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Role of Each Level

| Level | Responsibility | Application |
|-------|----------------|-------------|
| **Macro (ACE-FCA)** | Context management | Recommended |
| **Micro (AgentFlow)** | Task execution quality | Conditional |

### 4.3 Micro Level Application Criteria

| Condition | Sub-agent Approach | Reason |
|-----------|-------------------|--------|
| Strong model + Easy problem | Direct call | Avoid overhead |
| Strong model + Hard problem | AgentFlow | Quality improvement |
| Weak model + Easy problem | AgentFlow | Quality compensation |
| Weak model + Hard problem | AgentFlow (recommended) | Quality assurance |

**Key Insight**:
> AgentFlow is "a structuring method that makes weak models stronger, or enables solving difficult problems"

### 4.4 Concrete Example: Research Phase

```
Research Phase
│
├── Task 1: "Understand src/ folder structure"
│   └── Sub-agent (AgentFlow)
│       ├── Planner: "First check directory structure"
│       ├── Executor: Run tree command
│       ├── Verifier: "Structure understood, STOP"
│       └── Generator: Return structure summary
│       └── (Memory discarded)
│
├── Task 2: "Understand error handling patterns"
│   └── Sub-agent (AgentFlow)
│       ├── Planner: "Search for error-related files"
│       ├── Executor: Run grep
│       ├── Verifier: "Need more, CONTINUE"
│       ├── Planner: "Check key file contents"
│       ├── Executor: Read files
│       ├── Verifier: "Sufficient, STOP"
│       └── Generator: Return pattern summary
│       └── (Memory discarded)
│
├── Task 3: "Understand test structure"
│   └── ...
│
└── Synthesize results → Generate research.md
```

### 4.5 Why AgentFlow Approach is Recommended for Sub-tasks

**What seems like a "simple task"**:
```
"Read this file" → tool call → Return file contents
```

**What actually happens**:
```
Entire file contents (500 lines) enter parent context
→ Context pollution
→ Exactly the problem ACE-FCA tries to prevent
```

**What should actually be done**:
```
"Find and summarize only the error handling parts in this file"
→ Sub-agent reads → analyzes → returns summary
→ Already a multi-step task
```

**→ Most meaningful sub-tasks require the "execute → analyze → summarize" pattern, which naturally maps to the AgentFlow structure**

---

## 5. Trade-offs and Considerations (My Thoughts)

Here are the expected trade-offs when applying the integration strategy.

### 5.1 Main Trade-off: Overhead

```
When running AgentFlow for every sub-task:
- Increased LLM calls: Planner + Executor + Verifier + Generator
- Increased latency: Sequential execution
- Increased cost: API call costs
```

**Mitigation Strategies**:
- Apply full AgentFlow only to truly complex tasks
- Use lightweight versions for simple tasks (Executor → Summarizer only)
- Parallelize independent tasks

### 5.2 Application Scope

This integrated framework targets **large Brownfield codebases**:
- 300k+ LOC scale
- Requires understanding existing code
- Complex modifications/feature additions

**Not suitable for**:
- Small projects
- Greenfield development
- Simple bug fixes

### 5.3 Relationship with LLM Performance

| LLM Performance | Macro (ACE-FCA) | Micro (AgentFlow) |
|----------------|-----------------|-------------------|
| Strong | Still needed (context management) | Optional |
| Weak | Recommended | Recommended |

**Key Points**:
- **Context Management (Macro)**: Needed regardless of model performance
- **Task Quality (Micro)**: Need varies based on model performance

---

## 6. Conclusion

### 6.1 Key Summary

```
Macro (ACE-FCA)
  → Solves context management problem
  → Recommended for large projects regardless of model performance

Micro (AgentFlow)
  → Solves task execution quality problem
  → Apply selectively based on model performance / problem difficulty
```

### 6.2 Integration Framework in One Line

> "Manage context with ACE-FCA, and improve task execution quality with AgentFlow when needed"

### 6.3 Practical Application Guide

1. **When starting large projects**: Apply ACE-FCA's Research → Plan → Implement flow
2. **When executing sub-tasks within each Phase**: Spawn sub-agents to return only results
3. **Sub-agent implementation choice**:
   - Strong model + Simple task: Direct call
   - Otherwise: Apply AgentFlow structure
4. **Overhead management**: Utilize parallelization, lightweight versions

---

## References

- [ACE-FCA: Advanced Context Engineering for Coding Agents](https://github.com/humanlayer/advanced-context-engineering-for-coding-agents)
- [Youtube: No Vibes Allowed: Solving Hard Problems in Complex Codebases – Dex Horthy, HumanLayer](https://www.youtube.com/watch?v=rmvDxxNubIg)
- [Youtube: Advanced Context Engineering for Agents](https://www.youtube.com/watch?v=IS_y40zY-hc)
- [AgentFlow: In-the-Flow Agentic System Optimization](https://arxiv.org/abs/2510.05592)
- [AgentFlow GitHub Repository](https://github.com/lupantech/AgentFlow)
- [AgentFlow Project Page](https://agentflow.stanford.edu/)
