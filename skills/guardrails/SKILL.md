---
name: guardrails
description: >
  Use this skill when adding safety rails, input validation, or output checks
  to AI agent systems. Triggers include "add guardrails", "make this agent safe",
  "validate agent output", "prevent prompt injection", "add safety checks",
  "rate limit the agent", "restrict agent actions", or any task involving
  agent safety, input sanitization, output filtering, or action gating.
---

# Guardrails Skill

A skill that governs how to **design**, **implement**, or **audit** safety guardrails for AI agent systems.

Identify which operation applies, then follow the corresponding section.

---

## Operation: DESIGN — Designing Guardrails

### Pre-Design Checklist

Before adding guardrails, verify:

- [ ] I have identified the **risk categories** for this agent (data leakage, harmful output, unauthorized actions, injection)
- [ ] I know the **blast radius** of agent actions — what is reversible vs irreversible
- [ ] I have defined **acceptable vs unacceptable** agent behavior
- [ ] I know the **trust boundary** — what input sources are untrusted

### Guardrail Categories

**1. Input Validation**

Sanitize and validate all input before the agent processes it.

| Check | What It Catches | Implementation |
|---|---|---|
| Schema validation | Malformed input | JSON Schema / type checks |
| Length limits | Context overflow attacks | Max character/token count |
| Injection detection | Prompt injection attempts | Pattern matching + classifier |
| Encoding check | Unicode/encoding exploits | Normalize to UTF-8 |

**2. Output Filtering**

Validate agent output before returning to the user.

| Check | What It Catches | Implementation |
|---|---|---|
| PII detection | Leaked personal data | Regex for emails, phones, SSNs + NER |
| Content classification | Harmful or inappropriate content | Classifier or keyword filter |
| Format compliance | Wrong output structure | Schema validation |
| Hallucination indicators | Fabricated facts | Confidence scoring, source verification |
| Consistency check | Contradictory statements | Compare against input context |

**3. Action Gating**

Classify agent actions by risk level and gate accordingly.

| Risk Level | Examples | Gate |
|---|---|---|
| **Safe** | Read files, search, run tests | Auto-approve |
| **Moderate** | Write files, create branches | Log + proceed |
| **High** | Delete files, push code, send messages | Require user confirmation |
| **Critical** | Drop tables, force push, deploy to production | Require explicit approval + reason |

**Key principle — reversibility determines risk level.**
- Reversible actions (create a branch, write a file) → lower risk
- Irreversible actions (delete data, send email, deploy) → higher risk

**4. Rate Limiting**

Prevent runaway behavior.

| Limit | Purpose | Typical Value |
|---|---|---|
| Max tool calls per turn | Prevent infinite loops | 10-25 calls |
| Max consecutive failures | Stop futile retries | 3 failures |
| Max tokens per response | Control cost | Model-dependent |
| Max execution time | Prevent hung agents | 5-10 minutes |
| Max file operations | Prevent mass changes | 20 files per task |

**5. Scope Boundaries**

Restrict what the agent can access.

- **File system** — allowlist of directories, deny patterns (e.g., `**/.env`, `**/credentials*`)
- **Network** — allowlist of domains/APIs the agent can call
- **Database** — read-only access by default, write access per-table
- **External services** — no sending messages/emails without confirmation

---

## Operation: IMPLEMENT — Adding Guardrails to Code

### Pattern 1 — Pre-Execution Check

Validate before a tool call runs.

```python
def pre_check(tool_name, arguments):
    # Block dangerous file paths
    if tool_name == "write_file":
        path = arguments.get("path", "")
        if any(p in path for p in [".env", "credentials", "secrets"]):
            return {"blocked": True, "reason": "Cannot write to sensitive files"}

    # Rate limit
    if call_count[tool_name] > MAX_CALLS:
        return {"blocked": True, "reason": f"Rate limit exceeded for {tool_name}"}

    return {"blocked": False}
```

### Pattern 2 — Post-Execution Check

Validate output before returning to user.

```python
def post_check(output):
    # PII detection
    if contains_pii(output):
        return redact_pii(output)

    # Format validation
    if not matches_schema(output, expected_schema):
        return {"error": "Output format invalid", "raw": output}

    return output
```

### Pattern 3 — Circuit Breaker

Stop agent after repeated failures.

```python
consecutive_failures = 0
MAX_FAILURES = 3

def on_tool_result(result):
    if result.is_error:
        consecutive_failures += 1
        if consecutive_failures >= MAX_FAILURES:
            return stop_agent("Circuit breaker: {MAX_FAILURES} consecutive failures")
    else:
        consecutive_failures = 0
```

### Pattern 4 — Approval Gate

Pause for human approval on high-risk actions.

```python
HIGH_RISK_TOOLS = ["delete_file", "push_to_remote", "send_email", "deploy"]

def before_tool_call(tool_name, arguments):
    if tool_name in HIGH_RISK_TOOLS:
        approved = request_user_approval(
            action=tool_name,
            details=arguments,
            reason="This action is irreversible"
        )
        if not approved:
            return {"blocked": True, "reason": "User denied"}
```

### Pattern 5 — Audit Log

Record all agent decisions for review.

```python
def log_action(event_type, details):
    log_entry = {
        "timestamp": now(),
        "event": event_type,    # tool_call, handoff, decision, error
        "agent": current_agent,
        "details": details,
        "context_size": token_count(current_context)
    }
    append_to_audit_log(log_entry)
```

---

## Operation: AUDIT — Reviewing Existing Guardrails

### Evaluation Dimensions

| Dimension | Score 1 | Score 5 |
|---|---|---|
| **Input validation** | No validation | Schema + injection + length checks |
| **Output filtering** | No checks | PII + content + format checks |
| **Action gating** | All actions auto-approved | Risk-tiered with confirmation gates |
| **Scope boundaries** | Unrestricted access | Allowlisted paths, domains, tables |
| **Rate limiting** | No limits | Tool calls, time, and cost capped |
| **Logging** | No audit trail | All decisions and actions logged |

### Output Format

```
## Guardrails Audit Report

### Scores
| Dimension         | Score (1-5) | Notes |
|-------------------|-------------|-------|
| Input validation  |             |       |
| Output filtering  |             |       |
| Action gating     |             |       |
| Scope boundaries  |             |       |
| Rate limiting     |             |       |
| Logging           |             |       |

### Overall Score: X / 30

### Critical Gaps
- ...

### Recommendations (priority order)
1. ...
2. ...
```

### Red Team Checklist

Test these attack vectors:

- [ ] Prompt injection via user input — does the agent follow injected instructions?
- [ ] Path traversal — can the agent access files outside allowed directories?
- [ ] Sensitive data extraction — does the agent leak PII, API keys, or credentials?
- [ ] Privilege escalation — can the agent be tricked into high-risk actions without confirmation?
- [ ] Resource exhaustion — can input cause infinite loops or excessive API calls?
- [ ] Output manipulation — can the agent be made to produce harmful content?

---

## What NOT To Do

- Ship an agent with no guardrails and plan to "add them later"
- Use over-permissive defaults (default should be restrictive)
- Trust all tool results without validation
- Implement guardrails that can be bypassed by rephrasing
- Log sensitive data (PII, credentials) in audit logs
- Gate every action — too many confirmations cause user fatigue
- Rely solely on the model's built-in safety for application-specific risks
