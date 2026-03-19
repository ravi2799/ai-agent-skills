---
name: tool-designer
description: >
  Use this skill when creating, reviewing, or converting tool and function
  definitions for AI agents. Triggers include "write a tool definition",
  "create a function schema", "review this tool", "convert MCP tool",
  "improve tool description", "design agent tools", "build a LangChain tool",
  "add capability to my agent", or any task involving JSON Schema tool
  definitions, function calling schemas, MCP tool specs, or LangChain tools.
---

# Tool Designer Skill

A skill that governs how to **design**, **create**, **review**, or **convert** tool/function definitions for AI agents.

Think of this as: **an LLM comes to you and says "I want to enable this capability" — your job is to design and build the tool that grants it.**

Identify which operation applies, then follow the corresponding section.

---

## Operation: CREATE — Designing a New Tool

### Thinking Framework: Capability-First Design

When someone says "I want my LLM to be able to do X", follow this process:

1. **Define the capability** — What exactly should the LLM be able to do? (e.g., "search a database", "send an email", "check weather")
2. **Identify the boundary** — What does the LLM decide (parameters) vs. what the tool handles (execution logic)?
3. **Design the interface** — What inputs does the LLM need to provide? What output helps the LLM reason about the result?
4. **Choose the format** — JSON Schema definition, LangChain `@tool`, or `StructuredTool` depending on the stack

### Pre-Creation Checklist

Before writing a tool definition, verify:

- [ ] I understand the **capability** this tool grants to the LLM
- [ ] I know the **required vs optional** inputs
- [ ] I know the **expected output** format — what the LLM sees back
- [ ] I have considered **error cases** and what the tool returns on failure
- [ ] I have checked for **existing tools** that already cover this functionality
- [ ] I have considered **safety** — does this tool need confirmation for destructive actions?

---

### Format A: JSON Schema Tool Definition

Best for: provider-agnostic definitions, MCP tools, OpenAI function calling

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

---

### Format B: LangChain Python Tools

Best for: Python agents using LangChain/LangGraph

#### Method 1: `@tool` Decorator (Simplest)

The function docstring becomes the tool description the LLM reads. Type hints define the input schema automatically.

```python
from langchain.tools import tool

@tool
def search_issues(query: str, status: str = "open", limit: int = 10) -> str:
    """Search the issue tracker for bugs and feature requests matching the given criteria.
    Use this when the user asks about existing bugs, wants to find related issues,
    or needs to check for duplicates before filing a new issue.

    Args:
        query: Free-text search query matching issue titles and descriptions
        status: Filter by issue status — 'open', 'closed', or 'all'. Defaults to 'open'
        limit: Maximum number of results to return (1-100). Defaults to 10
    """
    # Tool implementation here
    results = db.search(query, status=status, limit=limit)
    return format_results(results)
```

**Key rules for `@tool`:**
- Type hints are **mandatory** — they define the input schema
- The docstring is **critical** — it is what the LLM reads to decide when to call this tool
- Use `Args:` section in docstring to describe each parameter
- Return a string for simple responses the LLM can read directly

#### Method 2: `@tool` with Custom Name and Pydantic Schema (Complex Inputs)

Use Pydantic models when you need validation, complex types, or detailed field descriptions:

```python
from langchain.tools import tool
from pydantic import BaseModel, Field
from typing import Literal

class WeatherInput(BaseModel):
    """Input for weather queries."""
    location: str = Field(description="City name or coordinates (e.g., 'San Francisco' or '37.7749,-122.4194')")
    units: Literal["celsius", "fahrenheit"] = Field(
        default="celsius",
        description="Temperature unit preference"
    )
    include_forecast: bool = Field(
        default=False,
        description="Whether to include a 5-day forecast in the response"
    )

@tool("get_weather", args_schema=WeatherInput)
def get_weather(location: str, units: str = "celsius", include_forecast: bool = False) -> str:
    """Get current weather conditions and optional forecast for a location.
    Use this when the user asks about weather, temperature, or outdoor conditions.

    Args:
        location: City name or coordinates
        units: Temperature unit — 'celsius' or 'fahrenheit'. Defaults to 'celsius'
        include_forecast: Include 5-day forecast. Defaults to False
    """
    data = weather_api.get(location, units=units)
    result = f"Current weather in {location}: {data['temp']}°{units[0].upper()}"
    if include_forecast:
        result += f"\nForecast: {data['forecast']}"
    return result
```

#### Method 3: StructuredTool (Programmatic Creation)

Use when building tools dynamically or from configuration:

```python
from langchain.tools import StructuredTool

def execute_sql(query: str, database: str = "main") -> str:
    """Run a read-only SQL query against the specified database."""
    # Implementation
    return str(db.execute(query, database))

sql_tool = StructuredTool.from_function(
    func=execute_sql,
    name="execute_sql_query",
    description="Run a read-only SQL query. Use this when the user asks data questions that require querying the database. Only SELECT statements are allowed.",
)
```

#### LangChain Tool Return Patterns

| Return Type | When to Use | Example |
|---|---|---|
| **`str`** | Simple human-readable output the LLM processes | `return "Found 5 matching records"` |
| **`dict`/object** | Structured data the LLM reasons about | `return {"count": 5, "results": [...]}` |
| **`Command`** | Tool needs to update agent state | `return Command(update={"user_name": name})` |

#### Reserved Parameter Names (LangChain)

These names **cannot** be used as tool arguments — they are reserved by the framework:
- **`config`** — Reserved for `RunnableConfig`
- **`runtime`** — Reserved for `ToolRuntime` (use this to access conversation state, store, and stream writer)

#### Accessing Runtime Context

Use `ToolRuntime` when the tool needs conversation state, user context, or persistent storage:

```python
from langchain.tools import tool, ToolRuntime

@tool
def get_user_preferences(runtime: ToolRuntime) -> str:
    """Retrieve the current user's saved preferences.
    Use this when the user asks to see their settings or when you need
    to personalize a response based on their preferences."""
    store = runtime.store
    prefs = store.get(("preferences",), runtime.context.user_id)
    return str(prefs.value) if prefs else "No preferences saved"
```

#### Streaming Progress Updates

For long-running tools, emit progress updates so the LLM can inform the user:

```python
@tool
def analyze_repository(repo_url: str, runtime: ToolRuntime) -> str:
    """Analyze a code repository for quality issues and generate a report.
    Use this when the user asks for a code review or quality analysis."""
    writer = runtime.stream_writer
    writer(f"Cloning repository: {repo_url}")
    repo = clone(repo_url)
    writer(f"Analyzing {len(repo.files)} files...")
    results = run_analysis(repo)
    writer("Generating report...")
    return format_report(results)
```

#### Registering Tools with a LangGraph Agent

```python
from langgraph.prebuilt import ToolNode, create_react_agent

# Collect all tools
tools = [search_issues, get_weather, execute_sql_query]

# Option 1: Create a ReAct agent directly
agent = create_react_agent(model, tools)

# Option 2: Use ToolNode for custom graph workflows
tool_node = ToolNode(tools, handle_tool_errors=True)
```

---

### Core Design Rules (All Formats)

**1. Name — snake_case, verb_noun format, specific**
| Weak | Strong |
|---|---|
| `do_thing` | `search_issues` |
| `helper` | `validate_schema` |
| `process` | `parse_csv_to_json` |
| `run` | `execute_sql_query` |

**2. Description — what it does AND when to use it**

The LLM reads the description to decide whether to call this tool. Include:
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

The LLM uses parameter descriptions to decide what values to pass. Include:
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

In LangChain, this means either detailed `Args:` docstrings or Pydantic `Field(description=...)`.

**4. Required — only truly required fields**

Mark a parameter as required only when the tool cannot function without it. Optional parameters with sensible defaults reduce friction.

**5. Enum — constrain when values are known**

If a parameter accepts a fixed set of values, use `enum` (JSON Schema) or `Literal` (Python type hint). This prevents the LLM from guessing invalid values.

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
| **Stateful** | Needs conversation/user context | Use `ToolRuntime` to access state and store |
| **Streaming** | Long-running operations | Use `runtime.stream_writer` for progress updates |

---

## Operation: REVIEW — Evaluating Tool Definitions

### Evaluation Dimensions

| Dimension | Score 1 | Score 5 |
|---|---|---|
| **Name clarity** | Vague (`helper`, `run`) | Specific snake_case verb_noun (`search_issues`) |
| **Description** | Missing or one word | Explains what, when, and disambiguation |
| **Schema completeness** | Missing types or properties | Full schema with types, constraints, and Literal/enum |
| **Parameter descriptions** | None or "the value" | Clear purpose, format, defaults, examples |
| **Error handling** | No mention of failures | Documents error responses and edge cases |
| **Scope** | Does 5 things via mode param | Single responsibility, one action |
| **Return value** | Raw dump or no docs | Clear return type with structure the LLM can reason about |

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
| Return value           |             |       |

### Overall Score: X / 35

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

### Common Format Mappings

| Field | OpenAI | Standard JSON | MCP | LangChain `@tool` |
|---|---|---|---|---|
| Tool name | `function.name` | `name` | `name` | function name or `@tool("name")` |
| Description | `function.description` | `description` | `description` | function docstring (first paragraph) |
| Parameters | `function.parameters` | `input_schema` | `inputSchema` | type hints + `Args:` docstring |
| Required | `parameters.required` | `input_schema.required` | `inputSchema.required` | params without defaults |
| Enum | `enum: [...]` | `enum: [...]` | `enum: [...]` | `Literal["a", "b"]` |
| Param description | `properties.x.description` | `properties.x.description` | `properties.x.description` | `Args:` docstring or `Field(description=...)` |

### JSON Schema → LangChain `@tool`

```json
{
  "name": "search_issues",
  "description": "Search the issue tracker...",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": { "type": "string", "description": "Search terms" },
      "status": { "type": "string", "enum": ["open", "closed", "all"], "description": "Filter by status" }
    },
    "required": ["query"]
  }
}
```

Converts to:

```python
from langchain.tools import tool
from typing import Literal

@tool
def search_issues(query: str, status: Literal["open", "closed", "all"] = "open") -> str:
    """Search the issue tracker...

    Args:
        query: Search terms
        status: Filter by status — 'open', 'closed', or 'all'. Defaults to 'open'
    """
    # implementation
```

**Conversion rules:**
- `required` params → no default value in Python
- Optional params → provide default value
- `enum` → `Literal[...]` type hint
- `description` on each property → `Args:` docstring entry or Pydantic `Field(description=...)`
- Top-level `description` → function docstring first paragraph

### LangChain `@tool` → JSON Schema

Reverse the mapping: extract type hints into `properties`, params without defaults become `required`, `Literal` becomes `enum`, docstring `Args:` become property descriptions.

### MCP to Standard Format

```
inputSchema  →  input_schema
```

All other fields map directly. Wrap MCP `call_tool()` responses in `tool_result` messages.

### Conversion Checklist

- [ ] All field names mapped to target format
- [ ] JSON Schema types preserved (or mapped to Python type hints)
- [ ] Enum values preserved (as `enum` or `Literal`)
- [ ] Required fields preserved (as `required` array or params without defaults)
- [ ] Descriptions carried over (not dropped) — docstrings, Field descriptions, or JSON descriptions
- [ ] Tested with a sample call after conversion

---

## What NOT To Do

- Write a tool with no description or a one-word description
- Create a "god tool" that handles multiple unrelated actions
- Use `required` for parameters that have sensible defaults
- Leave parameter descriptions empty
- Copy tool definitions without verifying field name mappings
- Add tools the agent will never need — fewer tools means better tool selection
- Use reserved names (`config`, `runtime`) as tool parameter names in LangChain
- Omit type hints in LangChain `@tool` functions — the schema won't generate
- Write a docstring that describes implementation details instead of when to use the tool
- Return raw data dumps — format returns so the LLM can reason about them
- Forget to handle errors gracefully — return error strings, don't raise exceptions that crash the agent
