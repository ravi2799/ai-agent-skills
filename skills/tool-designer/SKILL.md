---
name: tool-designer
description: >
  Use this skill when creating, reviewing, or converting tool and function
  definitions for AI agents. Triggers include "write a tool definition",
  "create a function schema", "review this tool", "convert MCP tool",
  "improve tool description", "design agent tools", or any task involving
  JSON Schema tool definitions, function calling schemas, or MCP tool specs.
---

# Tool Designer Skill

A skill that governs how to **create**, **review**, or **convert** tool/function definitions for AI agents.

Identify which operation applies, then follow the corresponding section.

---

## Operation: CREATE — Writing a New Tool Definition

### Pre-Creation Checklist

Before writing a tool definition, verify:

- [ ] I understand the **action** this tool performs
- [ ] I know the **required vs optional** inputs
- [ ] I know the **expected output** format and structure
- [ ] I have considered **error cases** and what the tool returns on failure
- [ ] I have checked for **existing tools** that already cover this functionality

### Anatomy of a Good Tool Definition

```json
{
  "name": "search_issues",
  "description": "Search the issue tracker for bugs and feature requests matching the given criteria. Use this when the user asks about existing bugs, wants to find related issues, or needs to check for duplicates before filing a new issue.",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Free-text search query matching issue titles and descriptions"
      },
      "status": {
        "type": "string",
        "enum": ["open", "closed", "all"],
        "description": "Filter by issue status. Defaults to 'open' if not specified"
      },
      "limit": {
        "type": "integer",
        "description": "Maximum number of results to return (1-100). Defaults to 10"
      }
    },
    "required": ["query"]
  }
}
```

### Core Rules

**1. Name — verb_noun format, specific**
| Weak | Strong |
|---|---|
| `do_thing` | `search_issues` |
| `helper` | `validate_schema` |
| `process` | `parse_csv_to_json` |
| `run` | `execute_sql_query` |

**2. Description — what it does AND when to use it**

The model reads the description to decide whether to call this tool. Include:
- What the tool does (first sentence)
- When to use it (second sentence)
- What it does NOT do, if there's a similar tool that could cause confusion

**Before:**
```
"description": "Gets data"
```
**After:**
```
"description": "Retrieve customer records from the database by customer ID, email, or name. Use this when the user asks about a specific customer's account, order history, or profile. For aggregate customer analytics, use customer_stats instead."
```

**3. Parameters — every parameter gets a description**

The model uses parameter descriptions to decide what values to pass. Include:
- What the parameter represents
- Valid values or ranges
- Default behavior when omitted
- An example for non-obvious formats

**Before:**
```json
{ "date": { "type": "string" } }
```
**After:**
```json
{ "date": { "type": "string", "description": "Date in ISO 8601 format (YYYY-MM-DD). Example '2025-03-15'" } }
```

**4. Required — only truly required fields**

Mark a parameter as required only when the tool cannot function without it. Optional parameters with sensible defaults reduce friction.

**5. Enum — constrain when values are known**

If a parameter accepts a fixed set of values, use `enum`. This prevents the model from guessing invalid values.

**6. Single responsibility — one tool, one action**

If a tool does multiple unrelated things based on a "mode" parameter, split it into separate tools.

### Tool Design Patterns

| Pattern | When to Use | Example |
|---|---|---|
| **CRUD** | Managing resources | `create_user`, `get_user`, `update_user`, `delete_user` |
| **Query** | Flexible search | `search_logs` with filter params (date, severity, source) |
| **Batch** | Processing multiple items | `send_emails` with an array of recipients |
| **Confirmation** | Dangerous operations | `preview_delete` (dry run) + `confirm_delete` (execute) |
| **Paginated** | Large result sets | `list_records` with `cursor` and `limit` params |

---

## Operation: REVIEW — Evaluating Tool Definitions

### Evaluation Dimensions

| Dimension | Score 1 | Score 5 |
|---|---|---|
| **Name clarity** | Vague (`helper`, `run`) | Specific verb_noun (`search_issues`) |
| **Description** | Missing or one word | Explains what, when, and disambiguation |
| **Schema completeness** | Missing type or properties | Full JSON Schema with types and constraints |
| **Parameter descriptions** | None or "the value" | Clear purpose, format, defaults, examples |
| **Error handling** | No mention of failures | Documents error responses and edge cases |
| **Scope** | Does 5 things via mode param | Single responsibility, one action |

### Output Format

```
## Tool Definition Review

### Scores
| Dimension              | Score (1-5) | Notes |
|------------------------|-------------|-------|
| Name clarity           |             |       |
| Description            |             |       |
| Schema completeness    |             |       |
| Parameter descriptions |             |       |
| Error handling         |             |       |
| Scope                  |             |       |

### Overall Score: X / 30

### Critical Issues
- ...

### Suggestions
- ...

### Revised Definition (if issues found)
[provide improved JSON]
```

### Common Mistakes

- **No description on parameters** — the model guesses what to pass
- **Too many parameters** — more than 8 parameters suggests the tool should be split
- **Missing enum constraints** — letting the model guess valid values
- **Overlapping tools** — two tools that do similar things confuse the model
- **No error documentation** — the model doesn't know what failures look like
- **Nested objects without descriptions** — deep schemas need descriptions at every level

---

## Operation: CONVERT — Translating Between Formats

### MCP to Standard Format

```
inputSchema  →  input_schema
```

All other fields map directly. Wrap MCP `call_tool()` responses in `tool_result` messages.

### Common Format Mappings

| Field | OpenAI | Standard | MCP |
|---|---|---|---|
| Function name | `function.name` | `name` | `name` |
| Description | `function.description` | `description` | `description` |
| Parameters | `function.parameters` | `input_schema` | `inputSchema` |
| Required | `parameters.required` | `input_schema.required` | `inputSchema.required` |

### Conversion Checklist

- [ ] All field names mapped to target format
- [ ] JSON Schema types preserved
- [ ] Enum values preserved
- [ ] Required fields preserved
- [ ] Descriptions carried over (not dropped)
- [ ] Tested with a sample call after conversion

---

## What NOT To Do

- Write a tool with no description or a one-word description
- Create a "god tool" that handles multiple unrelated actions
- Use `required` for parameters that have sensible defaults
- Leave parameter descriptions empty
- Copy tool definitions without verifying field name mappings
- Add tools the agent will never need — fewer tools means better tool selection
