---
name: agent-architect
description: >
  Use this skill when designing, reviewing, or debugging multi-agent systems.
  Triggers include "design an agent system", "how should I structure my agents",
  "review this agent architecture", "why is my agent looping", "add another agent",
  "single vs multi-agent", or any task involving agent orchestration, handoffs,
  tool selection, memory design, or multi-agent coordination.
---

# Agent Architect Skill

A skill that governs how to **design**, **review**, or **debug** multi-agent systems.

Identify which operation applies, then follow the corresponding section.

---

## Operation: DESIGN — Designing an Agent System

### Required Input from Previous Phases

If `data-scientist` (Phase 0) and `context-engineer` (Phase 0.5) have run, use their output:

- **Problem Statement** — from `data-scientist` (what we're solving, who needs it, what success looks like)
- **Data Profile** — from `data-scientist` (what data exists, what's signal vs noise)
- **Context Spec** — from `context-engineer` (how data loads into context, token budget, loading strategy)

**Do not re-discover what Phase 0 already answered.** The problem statement IS your starting point — design the agent to solve THAT problem with THAT data.

### Pre-Design Checklist

Before proposing any architecture, verify:

- [ ] I have the **problem statement** from `data-scientist` (or directly from the user)
- [ ] I have the **context spec** from `context-engineer` (if data is involved)
- [ ] I have considered whether a **single agent** can handle it (start here)
- [ ] I know the **tools** available or needed
- [ ] I know the **input/output contract** — what goes in, what comes out
- [ ] I have identified **failure modes** and how to handle them

If any item is unclear, ask the user before designing.

### Rule 1 — Start with One Agent

A single agent with good tools solves most problems. Add agents only when:

- The task has **distinct phases** requiring different expertise or tool sets
- Context window limits force **splitting long workflows**
- **Parallel execution** of independent subtasks provides meaningful speedup
- **Safety boundaries** require separating privileged from unprivileged operations

If none of these apply, use one agent.

### Rule 2 — One Task, One Agent (Critical for Accuracy)

**An agent that does one thing does it well. An agent that does many things does all of them poorly.**

Splitting responsibilities into single-task agents dramatically improves accuracy because:
- The agent's full context window focuses on one objective
- Tool selection is unambiguous — fewer tools means fewer wrong choices
- Prompts stay short and specific — no competing instructions
- Failures are isolated — one agent failing does not corrupt another's work
- Evaluation is straightforward — test one behavior, not a bundle

Every agent must have:

- **One task** — describable in a single sentence without "and"
- A **defined tool set** — only the tools needed for that one task
- **Clear input/output contracts** — what it receives, what it returns
- **No overlap** with other agents' responsibilities

**Test:** If your agent description contains "and" (e.g., "parses logs **and** generates reports"), split it into two agents.

**Before:**
```
Agent: "general helper" — has all tools, handles everything
  Result: mediocre at all tasks, picks wrong tools, bloated context
```
**After:**
```
Agent 1: "code reviewer" — receives a diff, returns structured feedback
  Tools: read_file, grep, run_tests
  Input: { diff: string, context: string }
  Output: { issues: [{severity, file, line, message}], summary: string }

Agent 2: "report generator" — receives review results, produces markdown report
  Tools: write_file, format_markdown
  Input: { issues: [...], summary: string }
  Output: { report_path: string }
```

### Rule 3 — Choose an Orchestration Pattern

| Pattern | When to Use | How It Works |
|---|---|---|
| **Sequential** | Ordered pipeline (parse → validate → transform) | Each agent's output feeds the next agent's input |
| **Parallel** | Independent subtasks (lint + test + typecheck) | Run simultaneously, aggregate results |
| **Hierarchical** | Complex task needing delegation | Orchestrator breaks task down, delegates to specialists, synthesizes |
| **Router** | Input determines which specialist to use | Classifier directs to the right agent based on input type |

Prefer **deterministic routing** (if/else, regex, keyword match) over LLM-based routing when the categories are well-defined. Use LLM routing only when input is ambiguous.

### Rule 4 — Design State and Memory

- Use **structured formats** (JSON objects) for state that agents read/write
- Use **unstructured text** for progress notes and reasoning traces
- Keep shared state **minimal** — pass only what the next agent needs
- For long-running tasks, use **checkpoints** (git commits, file snapshots) so work is not lost on failure

### Rule 5 — Define Handoff Protocols

For each handoff between agents, specify:

1. **Trigger** — what condition causes the handoff
2. **Payload** — exactly what data transfers (not "everything")
3. **Context** — minimal background the receiving agent needs
4. **Fallback** — what happens if the receiving agent fails

### Rule 6 — Design for Failure

Every agent system must handle:

- **Agent failure** — one agent errors or returns garbage → retry with backoff, then escalate
- **Loops** — agent calls the same tool repeatedly → circuit breaker after N attempts
- **Context overflow** — input too large → summarize or chunk before passing
- **Wrong routing** — input goes to the wrong agent → add a catch-all with re-routing logic

### Rule 7 — Design for Observability

Log these events at minimum:

- Agent activation (which agent, why, what input)
- Tool calls (which tool, what arguments, what result)
- Handoffs (from which agent, to which agent, what payload)
- Failures (what failed, what fallback was taken)
- Final output (what was returned to the user)

---

## Operation: REVIEW — Reviewing an Agent Architecture

### Evaluation Dimensions

| Dimension | Score 1 | Score 5 |
|---|---|---|
| **Simplicity** | Over-engineered, too many agents | Minimal agents, each clearly justified |
| **Responsibility clarity** | Overlapping or vague roles | Each agent has one sentence job description |
| **Tool coverage** | Missing tools or unnecessary tools | Each agent has exactly the tools it needs |
| **Error handling** | No failure handling | Retries, circuit breakers, fallbacks defined |
| **State management** | Entire context shared everywhere | Minimal, structured state passed explicitly |
| **Observability** | No logging | All decisions, handoffs, and failures logged |

### Output Format

```
## Agent Architecture Review

### Scores
| Dimension              | Score (1-5) | Notes |
|------------------------|-------------|-------|
| Simplicity             |             |       |
| Responsibility clarity |             |       |
| Tool coverage          |             |       |
| Error handling         |             |       |
| State management       |             |       |
| Observability          |             |       |

### Overall Score: X / 30

### Critical Issues
- ...

### Suggestions
- ...
```

### Common Anti-Patterns

- **The everything agent** — one agent with all tools and no clear role
- **Premature multi-agent** — using 5 agents where 1 would suffice
- **Context dumping** — passing entire conversation history between agents instead of a summary
- **LLM router for known categories** — using an LLM to classify when a regex or keyword match works
- **No failure path** — no defined behavior when an agent errors
- **Circular handoffs** — Agent A calls Agent B calls Agent A

---

## Operation: DEBUG — Diagnosing Agent Issues

### Systematic Diagnosis

When an agent system misbehaves, check in this order:

1. **Check routing** — Is the right agent receiving the task? Log the routing decision.
2. **Check tool definitions** — Are tool names, descriptions, and schemas correct? The model reads these to decide which tool to call.
3. **Check context size** — Is the agent receiving too much or too little context? Measure token count.
4. **Check handoff contracts** — Is the payload between agents complete? Missing fields cause downstream failures.
5. **Check for loops** — Is an agent calling the same tool repeatedly? Add a call counter.
6. **Check output format** — Is the agent returning what the next step expects? Schema mismatches cause silent failures.

### Common Failure Modes

| Symptom | Likely Cause | Fix |
|---|---|---|
| Agent loops on same tool | Tool returns unhelpful result, agent retries | Improve tool error messages, add circuit breaker |
| Agent calls non-existent tool | Tool name mismatch or hallucination | Verify tool names match exactly, reduce tool count |
| Context overflow errors | Too much state passed between agents | Summarize context, pass only what's needed |
| Wrong agent handles task | Router misclassifies input | Add examples to router, use deterministic rules |
| Agent produces empty output | Missing required context | Check input contract, ensure all required fields present |
| Cascading failures | No error isolation between agents | Add try/catch per agent, define fallback behavior |

---

## Post-Design Verification

After designing, reviewing, or debugging, confirm:

- [ ] Each agent has a one-sentence role description
- [ ] No two agents have overlapping responsibilities
- [ ] Tool sets are minimal — no agent has tools it does not need
- [ ] Input/output contracts are explicit for every handoff
- [ ] Failure handling is defined for every agent
- [ ] State passed between agents is minimal and structured
- [ ] The system could be explained to a new team member in under 2 minutes
