---
name: agent-builder
description: >
  The meta-skill for building AI agents end-to-end. Use this skill when
  someone asks "build me an agent", "create an agent that does X",
  "I need an agent for...", "help me build an AI agent", "set up an
  agent system", or any task that requires designing, implementing,
  and shipping a complete agent from scratch. This skill orchestrates
  all other skills in the correct order to deliver a working agent.
---

# Agent Builder Skill

The **conductor skill** — orchestrates the full agent build lifecycle from idea to working system. This skill knows about all other skills and calls them in the right order.

**When someone says "build me an agent that does X", follow this skill.**

---

## The Build Lifecycle

Every agent build follows these 9 phases in order. Do not skip phases — each one depends on the previous.

```
Phase 0:   DATA-ANALYST      → Understand the data before anything else
Phase 0.5: CONTEXT-ENGINEER  → Build loading strategy to fill context optimally
Phase 1:   PLAN              → What does this agent need to do?
Phase 2:   MODEL             → Configure LLM access
Phase 3:   TOOLS             → Design the capabilities it needs
Phase 4:   PROMPT            → Write the system prompt
Phase 5:   GUARD             → Add safety rails
Phase 6:   EVAL              → Build tests to verify it works
Phase 7:   SHIP              → Package and deploy
```

**Phase 0 and 0.5 are mandatory when the agent works with user-provided data.** Skip them only if the agent has no data inputs (e.g., a pure conversational agent).

### Decision Point: Simple vs Complex

Before starting, determine which path to take:

| Signal | Path | Implementation |
|---|---|---|
| Single task, 1-5 tools, linear flow | **Simple agent** | Single LLM + tools (ReAct pattern) |
| Multiple tasks, branching logic, state | **Complex agent** | LangGraph with state machine |
| Multiple specialists, handoffs needed | **Multi-agent** | LangGraph with subgraphs or agent-per-node |

**Default to simple.** Only escalate to complex/multi-agent when the task genuinely requires it.

---

## Phase 1: PLAN — Define What the Agent Does

*Related skill: `agent-architect`*

### Mandatory Questions (Answer All Before Proceeding)

1. **What is the agent's single job?** — Describe in one sentence without "and"
2. **Who is the user?** — Who will interact with this agent and how?
3. **What are the inputs?** — What data does the agent receive?
4. **What are the outputs?** — What should the agent produce?
5. **What tools does it need?** — What actions must it perform?
6. **What can go wrong?** — What are the failure modes?
7. **What should it NOT do?** — What are the boundaries?

### Output: Agent Specification

```
## Agent Spec: [name]

**Job:** [one sentence]
**User:** [who interacts with it]
**Input:** [what it receives]
**Output:** [what it produces]
**Tools needed:** [list]
**Boundaries:** [what it must NOT do]
**Success criteria:** [how to know it works]
**Path:** simple / complex / multi-agent
```

---

## Phase 2: MODEL — Configure LLM Access

*Related skill: `model-gateway`*

### Steps

1. **Choose a model** based on the task:

   | Task Type | Model Tier | Temperature |
   |---|---|---|
   | Classification, routing | Fast + cheap | 0.0 |
   | Analysis, reasoning, code | Strong | 0.0 |
   | Creative writing, reports | Strong | 0.5-0.7 |
   | Simple extraction | Mid-tier | 0.0 |

2. **Set up the LLM factory** — create or reuse `create_llm()`:

   ```python
   from common.llm import create_llm
   llm = create_llm()  # reads model + key + URL from .env
   ```

3. **Configure `.env`** with gateway URL, API key, and model name

4. **Enable tracing** (LangSmith) for debugging:
   ```env
   LANGSMITH_TRACING_V2=true
   LANGSMITH_PROJECT=your-agent-name
   ```

### Checkpoint: Can you make a basic LLM call and get a response? Test before proceeding.

---

## Phase 3: TOOLS — Design Agent Capabilities

*Related skill: `tool-designer`*

### Steps

1. **List every action** the agent needs to perform (from Phase 1 spec)
2. **One tool per action** — never bundle multiple actions into one tool
3. **Write each tool** with:
   - Clear name (verb_noun, snake_case)
   - Description that says WHAT it does and WHEN to use it
   - Typed parameters with descriptions
   - Error handling (return error strings, don't crash)

4. **Choose the format** based on your stack:
   - LangChain agent → `@tool` decorator with type hints
   - LangGraph agent → `@tool` + register in `ToolNode`
   - Raw API → JSON Schema tool definition

5. **Test each tool independently** before connecting to the agent

### Tool Checklist

For each tool, verify:
- [ ] Name is specific (not `helper` or `process`)
- [ ] Description includes when to use it
- [ ] All parameters have descriptions
- [ ] Required vs optional is correct
- [ ] Error cases return useful messages
- [ ] Tested independently with sample inputs

### Checkpoint: Can you call each tool manually and get correct results?

---

## Phase 4: PROMPT — Write the System Prompt

*Related skill: `prompt-engineer`*

### Steps

1. **Define the role** — specific to the domain, not generic
2. **State the task** — one primary objective, explicit
3. **List the rules** — what the agent must always/never do
4. **Specify the output format** — what the response should look like
5. **Add examples** — if behavior is non-obvious (3-5 examples)

### System Prompt Template

```
You are a [specific role] that [primary task].

You have access to the following tools:
[tools are provided automatically — do not list them manually]

## Rules
1. [most important rule]
2. [second rule]
...

## Output Format
[specify exactly what the response should look like]

## Examples (if needed)
<example>
Input: ...
Output: ...
</example>
```

### Common Mistakes

- Writing "You are a helpful assistant" (too generic — be specific)
- Listing tools in the prompt (they're injected automatically)
- Using vague instructions ("try to", "do your best")
- Not specifying output format (the agent guesses)

### Checkpoint: Does the prompt pass the "new employee" test? Would someone with no context know exactly what to do?

---

## Phase 5: GUARD — Add Safety Rails

*Related skill: `guardrails`*

### Minimum Guardrails (Every Agent Needs These)

1. **Input validation** — reject malformed input before processing
2. **Rate limiting** — cap tool calls to prevent loops (e.g., max 20 per turn)
3. **Action gating** — require confirmation for irreversible actions
4. **Output check** — verify output matches expected format before returning

### Risk Assessment

| Agent Action | Risk Level | Guardrail |
|---|---|---|
| Read files, search | Safe | Auto-approve |
| Write files, create | Moderate | Log + proceed |
| Delete, send, deploy | High | Require user confirmation |
| Access credentials, PII | Critical | Block + alert |

### Checkpoint: What's the worst thing this agent could do? Is there a guardrail preventing it?

---

## Phase 6: EVAL — Build Tests

*Related skill: `eval-designer`*

### Minimum Eval Set

Create at least **5 test cases**:

| Test Type | Count | Purpose |
|---|---|---|
| Happy path | 2 | Agent handles normal input correctly |
| Edge case | 2 | Agent handles unusual/boundary input |
| Failure mode | 1 | Agent handles errors gracefully |

### For Each Test Case

```
Input: [exact input to the agent]
Expected: [what the agent should do/return]
Method: [exact_match / schema_validation / llm_judge / human]
```

### Run the Eval Loop

1. Run all test cases against the agent
2. Score pass/fail for each
3. Fix failures — adjust prompt, tools, or guardrails
4. Re-run until all tests pass
5. Save results as baseline for future changes

### Checkpoint: Does the agent pass all 5+ test cases?

---

## Phase 7: SHIP — Package and Deploy

### For a Standalone Agent

```
your-agent/
  .env.example        # all config vars with placeholders
  README.md           # what it does, how to run
  src/
    agent.py           # main agent code
    tools.py           # tool definitions
    prompts.py         # system prompt
    llm.py             # LLM factory (or import from common)
  tests/
    test_agent.py      # eval test cases
```

### For a Reusable Skill

*Related skill: `skill-creator`*

Package the agent's knowledge as a SKILL.md so other agents can use it:
```bash
npx skills add your-org/your-repo --skill your-agent-skill
```

### Pre-Ship Checklist

- [ ] All eval tests pass
- [ ] `.env.example` documents every required variable
- [ ] No API keys committed to git
- [ ] Error handling covers all tool failures
- [ ] README explains how to set up and run
- [ ] Tracing enabled for production debugging

---

## Quick Reference: Skill Dependencies

When building an agent, these skills are called in order:

| Phase | Skill Called | What It Does |
|---|---|---|
| 0. Data | `data-analyst` | Understand the data before building |
| 0.5. Context | `context-engineer` | Build loading strategy for optimal context |
| 1. Plan | `agent-architect` | Design the agent system |
| 2. Model | `model-gateway` | Configure LLM calls |
| 3. Tools | `tool-designer` | Create tool definitions |
| 4. Prompt | `prompt-engineer` | Write the system prompt |
| 5. Guard | `guardrails` | Add safety rails |
| 6. Eval | `eval-designer` | Build test cases |
| 7. Ship | `skill-creator` | Package for distribution |

For complex multi-agent systems, also use:
- `langgraph-orchestrator` — when you need state machines, cycles, or human-in-the-loop

---

## What NOT To Do

- Skip the planning phase and jump straight to code
- Build a multi-agent system when a single agent would suffice
- Write tools without testing them independently first
- Deploy without any eval tests
- Use a generic system prompt ("You are a helpful assistant")
- Hardcode API keys or model names in source code
- Skip guardrails because "it's just a prototype"
- Build everything at once — follow the phases in order, checkpoint after each
