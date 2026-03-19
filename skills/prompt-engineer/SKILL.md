---
name: prompt-engineer
description: >
  Use this skill whenever an agent needs to create, modify, rewrite, evaluate,
  or review any prompt. Triggers include "write a prompt for...",
  "create a system prompt", "update this prompt", "improve this prompt",
  "rewrite the system prompt", "optimize this prompt", "change this prompt to...",
  "make this prompt better", "fix this prompt", "review this prompt",
  "is this a good prompt", "evaluate this prompt", or any task where the agent
  is about to write or edit a prompt string in code, config, or instructions.
  This skill enforces proven prompting best practices as mandatory rules
  for all prompt operations.
---

# Prompt Engineer Skill

A skill that governs how an agent must behave when **creating**, **modifying**, or **evaluating** any prompt — system prompts, user messages, or instruction strings.

Identify which operation applies, then follow the corresponding section below before proceeding.

---

## Operation: CREATE — Writing a New Prompt from Scratch

### Mandatory Pre-Write Checklist

Before writing a single word, verify:

- [ ] I have the **problem statement** from `data-scientist` (if available)
- [ ] I know the **context layout** from `context-engineer` — what data the agent will see in its context window, so the prompt can reference it correctly
- [ ] I understand the **task type** (coding, analysis, writing, classification, conversation, etc.)
- [ ] I know the **target model** and context (AI coding agent, API, chat interface)
- [ ] I know the **expected output format** (prose, JSON, code, list, etc.)
- [ ] I know if **examples** are needed or available
- [ ] I know if a **role** would help focus the model's behavior

**The prompt must be written WITH KNOWLEDGE of what's in the context window.** If `context-engineer` loads SIP logs and probe logs in a specific layout, the prompt should tell the agent how to use that data — not just what to do in the abstract.

If any item is unclear, ask the user before writing.

### Creation Steps

1. **Define the role** (if relevant) — specific, not generic
2. **State the task clearly** — one primary objective, explicit
3. **Add context/motivation** — explain *why* constraints exist
4. **Specify output format** — always, when it matters
5. **Add examples** — if behavior is non-obvious (3–5 in `<example>` tags)
6. **Structure with XML** — if the prompt mixes instructions, context, and input
7. **Run the self-contained test** — read it cold; would a model know exactly what to do?

---

## Operation: MODIFY — Editing an Existing Prompt

### Mandatory Pre-Edit Checklist

Before making **any** change, verify:

- [ ] I have read the **entire existing prompt** in full — no skimming
- [ ] I understand the **original intent** and will preserve it
- [ ] I know **why** this change is being made (what problem it solves)
- [ ] I have identified which rules below apply to this specific change

If any item is unclear, ask the user before editing.

### Modification Rules

- Never remove an instruction without understanding why it was there
- If an instruction seems wrong or redundant, **flag it to the user** — do not silently delete it
- Retain the original tone and voice unless explicitly asked to change it
- After editing, re-read the full prompt cold to confirm coherence

---

## Operation: EVALUATE — Reviewing Prompt Quality

### How to Run an Evaluation

When asked to review or evaluate a prompt, score it against each dimension below and produce a structured report.

**Output format for evaluations:**

```
## Prompt Evaluation Report

### Scores
| Dimension         | Score (1–5) | Notes |
|-------------------|-------------|-------|
| Clarity           |             |       |
| Context           |             |       |
| Output Format     |             |       |
| Role Definition   |             |       |
| Self-Containment  |             |       |
| Example Coverage  |             |       |
| Instruction Style |             |       |

### Overall Score: X / 35

### Critical Issues (must fix)
- ...

### Suggestions (nice to have)
- ...

### Revised Prompt (if issues found)
[provide improved version]
```

### Evaluation Dimensions

| Dimension | Score 1 | Score 5 |
|---|---|---|
| **Clarity** | Vague, ambiguous instructions | Explicit, specific, no guessing needed |
| **Context** | No motivation for constraints | Every rule has a "why" when it helps |
| **Output Format** | Format unspecified | Format fully defined with positive framing |
| **Role Definition** | Generic or absent | Specific, task-relevant role |
| **Self-Containment** | Relies on implicit context | Works cold, no external knowledge assumed |
| **Example Coverage** | No examples on complex tasks | 3–5 relevant, diverse examples with tags |
| **Instruction Style** | Negative-only ("don't do X") | Positive reframes ("do Y instead") |

---

## Core Rules (Apply to All Operations)

### 1. Be Clear and Direct
- Use **explicit, specific language** — never rely on vague phrases like "do your best" or "try to"
- State the desired output format and constraints directly
- Use numbered lists or bullets when **order or completeness of steps matters**
- Think of the prompt reader as a new employee with no implicit context

**Before:**
```
Help with customer emails
```
**After:**
```
You are a customer support assistant. When the user provides a customer email:
1. Identify the issue category (billing, technical, account, other)
2. Draft a professional reply that addresses the issue directly
3. Keep replies under 150 words
4. Always end with next steps for the customer
```

---

### 2. Provide Context and Motivation
- When adding a constraint, include **why** it matters so the model can generalize
- Don't just say what to avoid — explain the downstream reason

**Before:** `Never use bullet points`
**After:** `Never use bullet points — this output is rendered in a plain-text email client that displays them as raw characters.`

---

### 3. Use Examples for Complex Behavior
- Add `<example>` tags (or `<examples>` for multiple) when behavior is non-obvious
- Examples must be **relevant**, **diverse** (cover edge cases), and **structured**
- Target 3–5 examples for best reliability
- Skip examples for trivially simple tasks

---

### 4. Use XML Tags to Structure Complex Prompts
- When a prompt mixes instructions, context, examples, and inputs — separate them with XML tags
- Use consistent, descriptive names: `<instructions>`, `<context>`, `<examples>`, `<input>`, `<output_format>`

```xml
<instructions>
  Your role and task here.
</instructions>

<context>
  Background info the model needs.
</context>

<examples>
  <example>...</example>
</examples>

<input>
  {{USER_INPUT}}
</input>
```

---

### 5. Control Output Format Explicitly
- Always specify expected output format when it matters
- Use positive framing:
  - ❌ `Do not use markdown`
  - ✅ `Write your response as plain prose paragraphs with no markdown formatting`

#### Structured Output (When Code Consumes the Response)

When agent output must be parsed by code (not read by humans), use these patterns:

**Pattern A — JSON Mode**
Tell the model to output JSON and specify the exact schema:
```
Respond with a JSON object matching this exact schema:
{
  "verdict": "pass" | "fail",
  "confidence": 0.0 to 1.0,
  "reason": "string explaining the verdict"
}
Output ONLY the JSON object, no other text.
```

**Pattern B — Structured Sections with Markers**
Use clear delimiters the parser can find:
```
Output your analysis in this exact format:

VERDICT: [pass/fail]
CONFIDENCE: [0.0-1.0]
REASON: [one sentence]
DETAILS: [detailed explanation]
```

**Pattern C — Pydantic Output Parser (LangChain)**
```python
from langchain_core.output_parsers import PydanticOutputParser
from pydantic import BaseModel, Field

class Verdict(BaseModel):
    verdict: str = Field(description="pass or fail")
    confidence: float = Field(description="0.0 to 1.0")
    reason: str = Field(description="one sentence explanation")

parser = PydanticOutputParser(pydantic_object=Verdict)
prompt = prompt_template + "\n" + parser.get_format_instructions()
chain = prompt | llm | parser  # auto-parses to Verdict object
```

**Pattern D — Retry on Parse Failure**
```python
from langchain.output_parsers import RetryOutputParser

retry_parser = RetryOutputParser.from_llm(parser=parser, llm=llm)
# Automatically re-prompts the model if output doesn't parse
```

**Rule:** If the output feeds code, always validate it. If validation fails, retry with the parse error as feedback — do not silently accept malformed output.

---

### 6. Give the Model a Specific Role (When Relevant)
- Open the system prompt with a role sentence when domain expertise helps
- Keep roles specific to the task — generic filler adds little

**Weak:** `You are a helpful assistant.`
**Strong:** `You are a senior backend engineer specializing in Python API design and performance optimization.`

---

### 7. Tell the Model What To Do, Not What To Avoid

| Avoid | Prefer |
|---|---|
| Don't be verbose | Keep responses under 3 sentences |
| Don't hallucinate | Only state facts you're confident about; say "I don't know" if uncertain |
| Don't use jargon | Use plain language accessible to a non-technical reader |

---

### 8. Self-Contained Prompts
- The prompt must work **without relying on context the model won't have**
- Variable placeholders like `{{USER_INPUT}}` are fine — undefined implicit context is not
- Read the final prompt cold: would a model know exactly what to do?

---

## Post-Operation Verification (All Operations)

After creating, modifying, or evaluating a prompt, confirm:

- [ ] Original intent is preserved (modify) or user goal is met (create)
- [ ] No instructions were silently removed (modify only)
- [ ] Output format is specified (if it matters)
- [ ] Prompt is self-contained
- [ ] No vague language remains ("try to", "maybe", "if possible")
- [ ] XML structure is valid if tags were used
- [ ] Examples present if behavior is non-obvious

---

## What NOT To Do (All Operations)

- ❌ Write or edit a prompt without fully understanding the task first
- ❌ Remove instructions without flagging to the user
- ❌ Use generic role fillers like "You are a helpful assistant" unless specifically needed
- ❌ Leave only negative instructions without positive reframes
- ❌ Over-engineer a simple prompt with unnecessary XML structure
- ❌ Change tone or voice unless explicitly asked

---

## Reference

This skill applies proven prompting best practices drawn from industry research and official provider documentation. For deep reference on specific techniques (few-shot examples, long-context structuring, thinking/reasoning prompts, tool use instructions), consult your model provider's prompt engineering guide.
