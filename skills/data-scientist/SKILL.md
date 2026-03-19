---
name: data-scientist
description: >
  Use this skill before building any agent that works with data. Triggers
  include "analyze this data", "what's in this folder", "understand the
  problem", "understand the data first", "do EDA", "profile this dataset",
  "what data do I have", "what are we solving", "define the problem", or
  any task where you need to understand the problem and raw data before
  designing an agent or solution. This is always Phase 0.
---

# Data Scientist Skill

The **Phase 0 skill** — understand the problem AND the data before writing a single line of agent code. This is what every good engineer does before solving a problem.

**Rule: Never build an agent without first understanding what you're solving and what data you have.**

---

## The Analysis Pipeline

```
DEFINE PROBLEM → DISCOVER → PROFILE → ASSESS → MAP → RECOMMEND
```

Follow these phases in order. Each phase produces output that feeds the next. Every phase after Problem Definition is viewed through the lens of **"does this help solve the problem?"**

---

## Phase 0: DEFINE PROBLEM — What Are We Solving?

Before touching any data, understand the problem completely. This frames everything that follows.

### Mandatory Questions (Answer All)

1. **What is the problem?** — Describe in one sentence what needs to be solved
2. **Who needs it solved?** — Who is the end user and what do they care about?
3. **What does success look like?** — What output would solve the problem?
4. **What decisions will be made?** — How will the output be used?
5. **What domain is this?** — What expertise is needed to interpret the data?
6. **What constraints exist?** — Time, accuracy requirements, output format, etc.

### Output: Problem Statement

```
## Problem Statement

**Problem:** [one sentence — what needs to be solved]
**User:** [who needs this and why]
**Success:** [what the output looks like when the problem is solved]
**Domain:** [what expertise is needed]
**Constraints:** [accuracy, format, speed, etc.]
**Key questions the data must answer:**
1. [most important question]
2. [second question]
3. [third question]
```

**This problem statement guides every subsequent phase.** When profiling data, you're asking "does this field help answer my key questions?" When assessing quality, you're asking "will this issue prevent me from solving the problem?" When mapping signal vs noise, the problem statement IS the filter.

---

## Phase 1: DISCOVER — What Files Exist?

Scan the data folder and catalog everything.

### For Each File, Record:

| Field | Example |
|---|---|
| **File name** | `trace_1772811276.pcapng` |
| **Format** | PCAP, JSON, CSV, TXT, XML, log, Excel |
| **Size** | 2.3 MB |
| **Last modified** | 2026-03-06 |
| **Encoding** | UTF-8, ASCII, binary |

### Output: File Inventory

```
## Data Inventory

| # | File | Format | Size | Description |
|---|---|---|---|---|
| 1 | logs/app.log | Plain text log | 1.2 MB | Application log with timestamps |
| 2 | data/results.json | JSON | 450 KB | Structured test results |
| 3 | captures/trace.pcap | PCAP | 5.1 MB | Network packet capture |

Total files: 3
Total size: 6.75 MB
Estimated tokens: ~1.8M (text files only)
```

---

## Phase 2: PROFILE — What's Inside Each File?

Open each file and understand its structure.

### For Text/Log Files:
- What is the log format? (timestamp, level, message, etc.)
- What delimiters are used? (tab, comma, pipe, space)
- How many lines/entries?
- Sample of first 5 and last 5 lines
- What patterns repeat? (timestamps, IDs, status codes)

### For JSON Files:
- Top-level structure (object vs array)
- Key names and nesting depth
- Data types per field
- Array sizes
- Sample entry

### For CSV/Excel Files:
- Column names and types
- Row count
- Sample rows (first 3)
- Unique value counts per column
- Date ranges if timestamps exist

### For Binary Files (PCAP, images, etc.):
- File type confirmation (magic bytes)
- What tools are needed to parse it
- Whether it can be converted to text for context loading
- Key metadata extractable without full parsing

### Output: Data Profile

For each file, produce:

```
## Profile: logs/app.log

**Format:** Structured log, one entry per line
**Lines:** 12,847
**Time range:** 2026-03-06 18:30:00 to 2026-03-06 18:45:22
**Fields per line:** timestamp | level | component | message
**Log levels found:** INFO (8,201), ERROR (342), WARN (1,104), DEBUG (3,200)
**Key components:** AuthService, DBPool, APIGateway, CacheManager
**Notable patterns:**
- Burst of ERROR entries between 18:38:00–18:39:30
- DBPool timeout messages repeating every 2 seconds
- AuthService entries reference user IDs (UUID format)

**Sample entries:**
2026-03-06 18:30:01 | INFO  | APIGateway | Request received GET /api/users/123
2026-03-06 18:38:15 | ERROR | DBPool     | Connection timeout after 30000ms
```

---

## Phase 3: ASSESS — What's the Quality?

Evaluate data quality issues that could affect analysis.

### Quality Checklist

| Check | What to Look For |
|---|---|
| **Completeness** | Missing fields, empty values, truncated entries |
| **Consistency** | Mixed formats (date formats, naming conventions), encoding issues |
| **Duplicates** | Repeated entries, overlapping time ranges across files |
| **Noise** | Debug logs, boilerplate, headers/footers that add no value |
| **Corruption** | Malformed JSON, broken lines, binary garbage in text files |
| **Time alignment** | Do timestamps across files align? Different timezones? |

### Output: Quality Report

```
## Data Quality Assessment

| Issue | Severity | File(s) | Details |
|---|---|---|---|
| Mixed timestamp formats | Medium | app.log, results.json | Log uses ISO 8601, JSON uses Unix epoch |
| 12% entries have empty message field | Low | app.log | DEBUG entries often have no message |
| Duplicate entries in time range 18:38-18:39 | High | app.log | Same log line appears 2-3 times |
| JSON has nested nulls | Medium | results.json | 8 of 45 entries have null in 'details' |
```

---

## Phase 4: MAP — What Matters for the Task?

Given the user's task, determine what data is signal vs noise.

### For Each File, Classify:

| Classification | Meaning | Action |
|---|---|---|
| **Critical** | Directly needed to solve the task | Load into context first |
| **Supporting** | Provides useful context but not essential | Load if context budget allows |
| **Noise** | Irrelevant to the task | Do not load |

### For Each Field/Column, Classify:

| Classification | Meaning | Action |
|---|---|---|
| **Key field** | Directly used in analysis | Always include |
| **Context field** | Helps interpret key fields | Include if space allows |
| **Noise field** | Adds tokens without value | Strip before loading |

### Signal-to-Noise Analysis

```
## Task: "Find why VoWiFi registration failed"

### Critical Data (must load):
- SIP log entries (REGISTER messages, response codes)
- Probe log entries (automation commands, timestamps)
- PCAP summary (GTPv2 messages, bearer setup)

### Supporting Data (load if budget allows):
- Device properties (model, IMSI, firmware)
- Screenshot metadata
- WiFi connection log

### Noise (strip):
- DEBUG log entries (8,201 lines — 64% of log file)
- Repeated polling entries (same command every 2 seconds)
- Boilerplate headers in each log file
- Binary PCAP payload (not readable in context)
```

---

## Phase 5: RECOMMEND — What Should Be Done?

Produce a final recommendation that feeds into `context-engineer`.

### Output: Analysis Summary

```
## Data Analysis Summary

### Task: [user's task description]

### Data Overview
- X files, Y total size, Z estimated tokens (text only)
- Time range covered: [start] to [end]
- Key entities found: [phone numbers, IPs, user IDs, etc.]

### Key Findings
1. [Most important observation about the data]
2. [Second most important]
3. [Third]

### Data Quality Issues
- [Issues that need handling before loading]

### Context Loading Recommendation
- **Must load:** [files/sections, estimated tokens]
- **Load if space:** [files/sections, estimated tokens]
- **Strip/skip:** [files/sections, why]
- **Estimated context usage:** X% of window (target: 50-60%)

### Transformations Needed
- [e.g., "Filter app.log to ERROR+WARN only — saves 11K lines"]
- [e.g., "Convert Unix timestamps to ISO 8601 for consistency"]
- [e.g., "Extract SIP messages from raw log — discard transport framing"]

### Ready for Context Engineering: YES/NO
```

---

## Core Rules

1. **Never skip Phase 0** — understanding data before building is not optional
2. **Read before you assume** — open files, look at actual content, don't guess from filenames
3. **Quantify everything** — file sizes, line counts, token estimates, percentages
4. **Think in tokens** — every piece of data costs context budget; measure the cost
5. **Signal over volume** — 500 lines of relevant data beats 50,000 lines of everything
6. **Document for the next agent** — your output feeds `context-engineer`, make it actionable

---

## What NOT To Do

- Start building an agent without understanding the data first
- Assume file contents from filenames alone
- Load all data into context without assessing relevance
- Ignore data quality issues (they cause agent failures downstream)
- Skip the token budget estimation (context overflow kills accuracy)
- Produce analysis without actionable recommendations
