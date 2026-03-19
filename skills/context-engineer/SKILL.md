---
name: context-engineer
description: >
  Use this skill after data analysis to design and build context loading
  strategies for AI agents. Triggers include "engineer the context",
  "build a context loader", "optimize context window", "what should the
  agent see", "design context strategy", "load data efficiently", "manage
  context budget", "compress context", or any task involving deciding what
  goes into an agent's context window, in what order, and how to maximize
  signal-to-noise ratio within the token budget.
---

# Context Engineer Skill

The **Phase 0.5 skill** — takes the output from `data-analyst` and builds the actual loading strategy, scripts, and transformations that fill the agent's context window with maximum signal.

**Context engineering is not prompt engineering.** Prompt engineering is about *what you tell the agent to do*. Context engineering is about *what the agent sees when it starts working*.

---

## Core Principle

> Find the smallest possible set of high-signal tokens that maximizes the likelihood of desired outcomes.

Every token in the context window costs attention budget. Context is a **finite resource with diminishing returns** — more data does not mean better results. The right 5,000 tokens outperform the wrong 50,000.

---

## The Context Budget

### The 50-60% Rule

Never fill more than 50-60% of the context window with input data. The remaining capacity is needed for:

| Reserve | What It's For | Typical % |
|---|---|---|
| **Agent reasoning** | Chain-of-thought, analysis, planning | 15-20% |
| **Tool calls & results** | Tool inputs, outputs, intermediate results | 10-15% |
| **Output generation** | The agent's final response | 10-15% |
| **System prompt + tools** | Instructions and tool definitions | 5-10% |

### Budget Calculation

```
Available context = Model context window × 0.55
System prompt + tools ≈ measure actual tokens
Data budget = Available context - System prompt - Tools

Example (200K context model):
  Available: 200,000 × 0.55 = 110,000 tokens
  System prompt + tools: ~5,000 tokens
  Data budget: ~105,000 tokens
  ≈ 75,000 words ≈ 300 pages of text
```

---

## Operation: DESIGN — Creating a Context Strategy

### Input Required

This skill requires the output from `data-analyst` — specifically:
- File inventory with sizes and token estimates
- Data profiles showing structure and content
- Signal vs noise classification per file
- Quality issues identified
- Context loading recommendations

### Step 1 — Prioritize by Signal Value

Rank all data by how directly it helps solve the task:

| Priority | What Goes Here | Loading Order |
|---|---|---|
| **P0 — Critical** | Data without which the task cannot be solved | Load first, always |
| **P1 — Important** | Data that significantly improves accuracy | Load second |
| **P2 — Supporting** | Data that provides helpful context | Load if budget allows |
| **P3 — Reference** | Data that might be needed for edge cases | Make available via tools, don't preload |
| **P4 — Noise** | Data that adds tokens without value | Never load |

### Step 2 — Choose a Loading Strategy

| Strategy | When to Use | How It Works |
|---|---|---|
| **Full load** | Total data fits in 50% of context | Load everything relevant, no transformation needed |
| **Filtered load** | Data fits after removing noise | Strip noise lines/fields, load the rest |
| **Summarized load** | Data too large even after filtering | Summarize or aggregate, load summaries |
| **Progressive load** | Data much larger than context | Load critical data upfront, give agent tools to load more on demand |
| **Hybrid** | Mixed data types and sizes | Combine strategies per file |

### Step 3 — Design Transformations

For each file that needs transformation before loading:

```
## Transformation Plan

### File: app.log (12,847 lines → target: ~2,000 lines)
1. Filter to ERROR and WARN only (removes 11,401 DEBUG/INFO lines)
2. Deduplicate consecutive identical messages (keeps first + count)
3. Strip common prefix boilerplate from each line
4. Result: ~1,446 lines ≈ 4,300 tokens

### File: results.json (45 entries → target: 45 entries, compressed)
1. Remove null 'details' fields (8 entries)
2. Flatten nested metadata to single level
3. Remove internal IDs not relevant to task
4. Result: ~45 entries ≈ 2,100 tokens

### File: trace.pcap (5.1 MB binary → text summary)
1. Cannot load binary into context
2. Pre-extract SIP messages and GTPv2 summaries using tshark
3. Load extracted text summary only
4. Result: ~500 lines ≈ 1,500 tokens
```

### Step 4 — Define the Context Layout

Structure how data appears in the context window. Order matters — information placed earlier gets stronger attention.

```
## Context Layout

[SYSTEM PROMPT]                          ← Agent instructions
[TOOL DEFINITIONS]                       ← Available capabilities

--- DATA SECTION (loaded by context-engineer) ---

## Task
[User's task description]

## Key Facts
[Critical entities: phone numbers, IPs, timeline, etc.]

## Primary Evidence
[P0 data — the most important files/sections]

## Supporting Evidence
[P1 data — important but secondary]

## Additional Context
[P2 data — if budget allows]

--- END DATA SECTION ---

[AGENT REASONING SPACE]                  ← Reserved for the agent
```

**Layout rules:**
- Place the task description and key facts FIRST — the agent needs to know what it's solving before seeing data
- Place most critical data immediately after the task
- Place supporting data later
- Long documents go at the TOP (before instructions) for long-context models, or use document tags

---

## Operation: BUILD — Writing the Loading Scripts

### Script Pattern: Context Loader

```python
"""
Context loader for [agent name].
Loads and transforms data files into optimized context.

Generated by context-engineer based on data-analyst output.
"""

import os
from pathlib import Path


def load_context(data_dir: str, max_tokens: int = 105_000) -> str:
    """Load and transform data files into agent context.

    Args:
        data_dir: Path to the data folder
        max_tokens: Maximum token budget for data (default: 105K)

    Returns:
        Formatted context string ready for the agent
    """
    data_dir = Path(data_dir)
    sections = []
    token_count = 0

    # P0: Critical data (always load)
    p0_files = [
        ("sip_log.txt", load_sip_log),
        ("probe_log.txt", load_probe_log),
    ]
    for filename, loader in p0_files:
        filepath = data_dir / filename
        if filepath.exists():
            content = loader(filepath)
            tokens = estimate_tokens(content)
            if token_count + tokens <= max_tokens:
                sections.append(content)
                token_count += tokens

    # P1: Important data (load if budget allows)
    p1_files = [
        ("device_info.json", load_device_info),
        ("screenshot_metadata.txt", load_screenshots),
    ]
    for filename, loader in p1_files:
        filepath = data_dir / filename
        if filepath.exists():
            content = loader(filepath)
            tokens = estimate_tokens(content)
            if token_count + tokens <= max_tokens * 0.85:  # leave margin
                sections.append(content)
                token_count += tokens

    return format_context(sections, token_count, max_tokens)


def load_sip_log(filepath: Path) -> str:
    """Load SIP log, filtered to REGISTER and INVITE messages only."""
    lines = filepath.read_text().splitlines()
    relevant = [l for l in lines if any(k in l for k in
                ["REGISTER", "INVITE", "200 OK", "403", "404", "480"])]
    return f"## SIP Log ({len(relevant)} relevant messages)\n\n" + "\n".join(relevant)


def load_probe_log(filepath: Path) -> str:
    """Load probe log, deduplicated and stripped of repeated polling."""
    lines = filepath.read_text().splitlines()
    seen = set()
    filtered = []
    for line in lines:
        # Deduplicate consecutive identical commands
        key = line.split("|", 2)[-1].strip() if "|" in line else line
        if key not in seen:
            seen.add(key)
            filtered.append(line)
    return f"## Probe Log ({len(filtered)} entries)\n\n" + "\n".join(filtered)


def estimate_tokens(text: str) -> int:
    """Rough token estimate: ~4 characters per token."""
    return len(text) // 4


def format_context(sections: list, used: int, budget: int) -> str:
    """Format sections with budget summary."""
    header = f"## Context Budget: {used:,} / {budget:,} tokens ({used*100//budget}% used)\n\n"
    return header + "\n\n---\n\n".join(sections)
```

### Script Pattern: Transformation Functions

Common transformations you'll need:

```python
# Filter log by severity
def filter_by_severity(lines, min_level="WARN"):
    levels = {"DEBUG": 0, "INFO": 1, "WARN": 2, "ERROR": 3, "FATAL": 4}
    min_val = levels.get(min_level, 0)
    return [l for l in lines if any(
        lvl in l and levels.get(lvl, 0) >= min_val for lvl in levels
    )]

# Deduplicate consecutive repeated lines
def dedup_consecutive(lines):
    result = []
    prev = None
    count = 0
    for line in lines:
        if line == prev:
            count += 1
        else:
            if count > 0:
                result.append(f"  [repeated {count} more times]")
            result.append(line)
            prev = line
            count = 0
    return result

# Extract time window
def filter_time_range(lines, start, end, time_parser):
    return [l for l in lines if start <= time_parser(l) <= end]

# Strip fields from JSON
def strip_fields(data, remove_keys):
    if isinstance(data, dict):
        return {k: strip_fields(v, remove_keys)
                for k, v in data.items() if k not in remove_keys}
    if isinstance(data, list):
        return [strip_fields(item, remove_keys) for item in data]
    return data

# Truncate to token budget
def truncate_to_budget(text, max_tokens):
    estimated = len(text) // 4
    if estimated <= max_tokens:
        return text
    char_limit = max_tokens * 4
    truncated = text[:char_limit]
    return truncated + f"\n\n[TRUNCATED: {estimated - max_tokens} tokens omitted]"
```

---

## Operation: OPTIMIZE — Improving Context Quality

### Technique 1 — Compaction

When context approaches the limit during long agent runs, compress the conversation:

- Preserve: decisions made, unresolved issues, key findings
- Discard: raw tool outputs already processed, repeated messages
- Summarize: multi-turn reasoning into conclusions

### Technique 2 — Structured Note-Taking

For long-running tasks, the agent should write notes to external memory:

```python
# Agent maintains a NOTES.md file
notes = {
    "key_findings": ["PGW rejected WLAN bearer", "Error code 72"],
    "decisions_made": ["Focus on GTPv2 frames 551-614"],
    "remaining_tasks": ["Check HSS subscriber profile"],
    "important_values": {"a_party": "+306971279001", "error_code": 72},
}
```

This lets the agent offload details and reclaim context space.

### Technique 3 — Progressive Disclosure via Tools

Instead of preloading everything, give the agent tools to load data on demand:

```python
@tool
def read_log_section(filename: str, start_line: int, end_line: int) -> str:
    """Read a specific section of a log file.
    Use this when you need to examine a particular time range
    or section in detail. Prefer this over loading entire files."""
    lines = Path(filename).read_text().splitlines()
    section = lines[start_line:end_line]
    return "\n".join(section)
```

This is the **just-in-time strategy** — lightweight identifiers upfront, detailed data loaded at runtime.

### Technique 4 — Sub-Agent Context Isolation

For complex analysis requiring multiple data sources:

```
Main agent context: task + summary + coordination (small)
    └─ Sub-agent 1: loads and analyzes log files (full exploration)
       └─ Returns: 1,000-token summary of findings
    └─ Sub-agent 2: loads and analyzes PCAP data (full exploration)
       └─ Returns: 1,000-token summary of findings
Main agent: synthesizes sub-agent summaries into final answer
```

Each sub-agent gets a clean context window for deep analysis. The main agent only sees condensed summaries.

---

## Output: Context Engineering Specification

The final deliverable is a spec that the builder agent uses:

```
## Context Engineering Spec

### Model & Budget
- Model: [name] with [X]K context
- Data budget: [Y]K tokens (55% of context)
- System prompt + tools: ~[Z]K tokens

### Loading Strategy: [full / filtered / summarized / progressive / hybrid]

### Context Layout (in order)
1. Task description + key facts (500 tokens)
2. [P0 file] filtered to [criteria] (~X tokens)
3. [P0 file] with [transformation] (~X tokens)
4. [P1 file] if budget allows (~X tokens)
5. Tools for on-demand loading of P2/P3 data

### Transformations Required
- [file]: [transformation] → saves [X] tokens
- [file]: [transformation] → saves [X] tokens

### Loading Script
- Location: src/context_loader.py
- Entry point: load_context(data_dir, max_tokens)

### Optimization Techniques Enabled
- [ ] Compaction (for long-running tasks)
- [ ] Note-taking (for multi-step tasks)
- [ ] Progressive disclosure (for large datasets)
- [ ] Sub-agent isolation (for multi-source analysis)
```

---

## Core Rules

1. **Context is finite** — treat every token as a cost against the attention budget
2. **50-60% maximum** — never fill more than 60% with input data
3. **Signal over volume** — 5K relevant tokens beats 50K of everything
4. **Order matters** — critical data goes first, noise never loads
5. **Measure, don't guess** — estimate tokens for every file and transformation
6. **Build scripts, not manual processes** — context loading must be repeatable
7. **Design for the agent, not for humans** — what helps the model reason, not what looks organized to you

---

## What NOT To Do

- Dump all files into context and hope the agent figures it out
- Exceed 60% of context window with input data
- Load data without transforming or filtering first
- Ignore token costs ("it's a big context window, it'll be fine")
- Load binary files directly into text context
- Assume more data means better results (context rot is real)
- Build context strategies without first running `data-analyst`
- Hardcode file paths instead of building reusable loading scripts
