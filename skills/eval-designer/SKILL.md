---
name: eval-designer
description: >
  Use this skill when designing evaluation frameworks, writing test cases,
  or analyzing results for AI agent systems. Triggers include "evaluate this
  agent", "how do I test my agent", "build an eval", "measure agent quality",
  "create test cases", "is my agent working well", "benchmark this prompt",
  or any task involving agent evaluation, metrics, grading rubrics, or
  performance measurement.
---

# Eval Designer Skill

A skill that governs how to **design**, **implement**, or **analyze** evaluation frameworks for AI agent systems.

Identify which operation applies, then follow the corresponding section.

---

## Operation: DESIGN — Designing an Evaluation Framework

### Pre-Design Checklist

Before building any evaluation, verify:

- [ ] I have **defined success criteria** — what does "good" look like?
- [ ] Success criteria are **specific and measurable** (not "performs well")
- [ ] I know the **task type** (classification, generation, extraction, conversation, multi-step)
- [ ] I have **real examples** of expected inputs and outputs
- [ ] I know whether **automated or human** evaluation is appropriate

### Step 1 — Define Success Criteria

Use the SMART framework:

| Property | Bad | Good |
|---|---|---|
| **Specific** | "Good performance" | "Correctly classifies sentiment as positive/negative/neutral" |
| **Measurable** | "High quality" | "F1 score above 0.85 on test set" |
| **Achievable** | "100% accuracy" | "95% accuracy on well-formed inputs" |
| **Relevant** | "Fast response" (when accuracy matters) | "Accurate extraction" (for a data pipeline) |

### Step 2 — Choose Evaluation Categories

Select the categories relevant to your agent:

| Category | What It Measures | Example Metric |
|---|---|---|
| **Task fidelity** | Core accuracy on the primary task | Accuracy, F1, exact match |
| **Edge case handling** | Behavior on unusual inputs | Pass rate on adversarial set |
| **Consistency** | Same input produces similar output | Cosine similarity across runs |
| **Format compliance** | Output matches expected schema | JSON validation pass rate |
| **Tone and style** | Voice matches requirements | Likert scale (1-5) |
| **Context utilization** | Uses provided context effectively | Grounded answer rate |
| **Safety** | No harmful or leaking outputs | PII detection rate, refusal rate |
| **Latency** | Response speed | p50, p95 response time |
| **Cost** | Token/API usage | Cost per task completion |

### Step 3 — Choose Evaluation Method

Prefer methods higher in this hierarchy:

**1. Code-based (fastest, most reliable)**
- Exact string match — classification, yes/no answers
- Regex match — structured outputs with known patterns
- JSON schema validation — API responses, structured data
- Unit tests — deterministic transformations

**2. LLM-as-judge (flexible, verify reliability first)**
- Binary classification — "Did the response answer the question? yes/no"
- Likert scale — "Rate helpfulness 1-5"
- Ordinal comparison — "Which response is better, A or B?"
- Rubric-based — score against detailed criteria

**3. Human review (most flexible, use sparingly)**
- Expert evaluation — domain-specific quality
- User satisfaction — real-world acceptance
- Preference ranking — compare multiple outputs

---

## Operation: IMPLEMENT — Building Test Cases

### Minimum Requirements

- **At least 3 evaluation scenarios** per capability being tested
- Include **happy path**, **edge cases**, and **failure modes**
- Each test case has: input, expected output (or acceptance criteria), and evaluation method

### Test Case Template

```
## Test Case: [descriptive name]

**Input:**
[exact input to the agent]

**Expected Output:**
[expected response or acceptance criteria]

**Evaluation Method:**
[exact_match | regex | json_schema | llm_judge | human]

**Category:**
[task_fidelity | edge_case | consistency | format | safety]

**Notes:**
[why this test case matters, what it catches]
```

### Evaluation Patterns

**Pattern 1 — Exact Match**
Best for classification, entity extraction, yes/no questions.
```python
def eval_exact_match(output, expected):
    return output.strip().lower() == expected.strip().lower()
```

**Pattern 2 — JSON Schema Validation**
Best for structured outputs.
```python
def eval_schema(output, schema):
    try:
        data = json.loads(output)
        jsonschema.validate(data, schema)
        return True
    except (json.JSONDecodeError, jsonschema.ValidationError):
        return False
```

**Pattern 3 — LLM-as-Judge with Rubric**
Best for subjective quality. Always:
- Write a detailed rubric (not "rate quality")
- Request a numeric score (not prose)
- Use reasoning tags, then extract only the score
- Test grader reliability on 10+ examples before trusting it

```
Rate the following response on a scale of 1-5 for helpfulness.

<rubric>
1 - Does not address the question at all
2 - Partially addresses the question but misses key points
3 - Addresses the question but lacks depth or contains minor errors
4 - Addresses the question well with good detail
5 - Excellent response that fully addresses the question with insight
</rubric>

<response>
{{AGENT_OUTPUT}}
</response>

First reason about the response in <thinking> tags, then output
only a single integer 1-5 as your final answer.
```

**Pattern 4 — A/B Comparison**
Best for prompt iteration. Show both outputs to a judge without revealing which is A/B (randomize order).

### Building an Eval Suite

```
eval_suite/
  config.yaml          # eval settings, model, thresholds
  test_cases/
    task_fidelity.yaml  # core accuracy tests
    edge_cases.yaml     # unusual inputs
    safety.yaml         # adversarial inputs
  graders/
    rubric_helpful.md   # LLM judge rubric
  results/
    baseline.json       # scores before changes
    current.json        # scores after changes
```

---

## Operation: ANALYZE — Interpreting Results

### Analysis Steps

1. **Compare to baseline** — are scores better, worse, or same?
2. **Identify failure patterns** — do failures cluster by category, input type, or length?
3. **Rank improvements by impact** — fix the highest-frequency failures first
4. **Check for regressions** — did fixing one thing break another?
5. **Track over time** — maintain a score history across iterations

### Results Report Format

```
## Evaluation Report

### Summary
| Metric | Baseline | Current | Delta |
|--------|----------|---------|-------|
| Task accuracy | 78% | 85% | +7% |
| Edge case pass rate | 45% | 62% | +17% |
| Format compliance | 92% | 95% | +3% |
| Average latency | 2.1s | 2.3s | +0.2s |

### Failure Analysis
| Failure Type | Count | Example | Root Cause |
|---|---|---|---|
| Missing field | 12 | [example] | Schema not specified in prompt |
| Wrong format | 5 | [example] | Ambiguous format instruction |

### Recommended Actions (priority order)
1. [highest impact fix]
2. [next priority]
3. [...]
```

### Evaluation-Driven Development Loop

1. Run agent on real tasks **without** changes — document failures
2. Create eval scenarios **from those failures**
3. Establish **baseline scores**
4. Make **minimal changes** to address the top failures
5. Re-evaluate — compare to baseline
6. Repeat until target scores are met

### The Two-Agent Testing Pattern

- **Agent A** (designer) — writes or refines the prompt/skill
- **Agent B** (fresh instance) — tests it on real tasks with no prior context
- Observe Agent B's behavior, bring insights back to Agent A
- This prevents the designer from assuming context that won't exist in production

---

## What NOT To Do

- Evaluate without defining success criteria first
- Use only happy-path test cases (always include edge cases)
- Trust an LLM judge without testing its reliability
- Use human review when code-based evaluation would work
- Optimize for a metric that doesn't reflect real-world quality
- Skip the baseline — you cannot measure improvement without a starting point
- Create test cases that only pass with exact wording (brittle tests)
