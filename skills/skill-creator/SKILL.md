---
name: skill-creator
description: >
  Use this skill when writing new SKILL.md files, reviewing existing skills,
  or preparing skills for distribution. Triggers include "create a new skill",
  "write a SKILL.md", "review this skill", "publish this skill", "is this
  skill well-written", "make a skill for...", or any task involving authoring,
  evaluating, or distributing agent skill files.
---

# Skill Creator Skill

A skill that governs how to **create**, **review**, or **publish** agent skill files (SKILL.md).

Identify which operation applies, then follow the corresponding section.

---

## Operation: CREATE — Writing a New Skill

### Pre-Creation Checklist

Before writing a skill, verify:

- [ ] I understand the **task domain** this skill covers
- [ ] I can name it in **gerund form** (e.g., `processing-pdfs`, not `pdf-helper`)
- [ ] I know the **trigger phrases** that should activate it
- [ ] I have identified **2-4 operations** the skill supports
- [ ] I know if **examples** are needed for non-obvious behavior

### Skill File Structure

```
skills/
  your-skill-name/
    SKILL.md              # Main instructions (under 500 lines)
    resources/            # Optional — templates, scripts, configs
      template.json
      helper.sh
```

### SKILL.md Anatomy

Every SKILL.md follows this structure:

```markdown
---
name: your-skill-name
description: >
  What this skill does and when to trigger it.
  Include specific trigger phrases.
---

# Skill Title

One-line summary of what this skill governs.

---

## Operation: VERB — Doing the Thing

### Pre-Operation Checklist
- [ ] Verify before acting...

### Rules and Guidance
...

### Examples (if behavior is non-obvious)
...

---

## Operation: ANOTHER_VERB — Another Thing

...

---

## Post-Operation Verification
- [ ] Confirm after completing...

## What NOT To Do
- List of anti-patterns
```

### Frontmatter Rules

**name** (required)
- Max 64 characters
- Lowercase letters, numbers, hyphens only
- Use gerund form for action skills

| Weak | Strong |
|---|---|
| `helper` | `processing-pdfs` |
| `utils` | `designing-apis` |
| `tool` | `reviewing-code` |

**description** (required)
- Max 1024 characters
- Third person ("Processes files" not "I help you process files")
- Must include **what** it does AND **when** to trigger
- Must be specific enough for the agent to choose from 100+ skills
- Use YAML folded scalar (`>`) to avoid colon parsing issues

| Weak | Strong |
|---|---|
| "A helpful skill" | "Processes Excel files into structured JSON. Use when the user uploads .xlsx files or asks to extract spreadsheet data" |

### Writing Instructions — Core Principles

**1. Be concise — the agent is smart**

Only write what the agent does not already know. Challenge each paragraph:
does this earn its token cost?

**Before (wasteful):**
```
When you receive a JSON file, you should parse it using a JSON parser.
Make sure the JSON is valid before processing it. JSON stands for
JavaScript Object Notation and is a common data format.
```

**After (useful):**
```
Parse the input JSON. If parsing fails, return the exact error position
and a corrected snippet.
```

**2. Set appropriate degrees of freedom**

- **High freedom** (text guidance) — for flexible, creative tasks
- **Low freedom** (exact scripts/commands) — for fragile operations where precision matters

**3. Use checklists for multi-step workflows**

Checklists prevent the agent from skipping steps and make progress trackable.

**4. Include examples for non-obvious behavior**

Use before/after pairs or input/output pairs wrapped in example blocks.

**5. Keep file references one level deep**

SKILL.md should reference files in its own directory only — not `../../some/other/path`.

**6. Add a table of contents for files over 100 lines**

Helps the agent navigate long instruction sets.

### Progressive Disclosure — Three Levels

| Level | When Loaded | Token Budget | Content |
|---|---|---|---|
| **L1 — Metadata** | Always (startup) | ~100 tokens | name + description from YAML |
| **L2 — Instructions** | When triggered | Under 5,000 tokens | SKILL.md body |
| **L3 — Resources** | As needed | Unlimited | Bundled files, scripts, templates |

Design for this: keep SKILL.md lean (under 500 lines), put heavy content in resources/.

---

## Operation: REVIEW — Evaluating a Skill

### Evaluation Dimensions

| Dimension | Score 1 | Score 5 |
|---|---|---|
| **Description quality** | Vague, missing triggers | Specific what + when + trigger phrases |
| **Conciseness** | Bloated, restates known facts | Every line earns its tokens |
| **Structure** | Wall of text | Clear operations, checklists, examples |
| **Scope** | Too broad ("helps with everything") or too narrow | Focused single domain |
| **Portability** | Provider-specific language | Works with any agent |
| **Examples** | Missing on complex behavior | Relevant, diverse, before/after pairs |

### Output Format

```
## Skill Review Report

### Scores
| Dimension           | Score (1-5) | Notes |
|---------------------|-------------|-------|
| Description quality |             |       |
| Conciseness         |             |       |
| Structure           |             |       |
| Scope               |             |       |
| Portability         |             |       |
| Examples            |             |       |

### Overall Score: X / 30

### Critical Issues
- ...

### Suggestions
- ...
```

### Anti-Patterns to Flag

- Vague skill name (`helper`, `utils`, `assistant`)
- Description without trigger conditions
- Over 500 lines (too expensive to load)
- Time-sensitive information (dates, versions that expire)
- Inconsistent terminology (mixing "function" and "tool" for the same concept)
- Deeply nested file references
- Assuming specific tools or packages are installed
- Provider-specific language (naming specific AI providers)

---

## Operation: PUBLISH — Preparing for Distribution

### Pre-Publish Checklist

- [ ] SKILL.md has valid YAML frontmatter with `name` and `description`
- [ ] Body is under 500 lines
- [ ] No provider-specific language
- [ ] No time-sensitive information
- [ ] Consistent terminology throughout
- [ ] Examples present for non-obvious behavior
- [ ] Tested on at least 3 real scenarios
- [ ] README updated with skill in the available skills table

### Distribution via skills CLI

```bash
# Preview what skills are found in the repo
npx skills add owner/repo --list

# Install all skills from a repo
npx skills add owner/repo

# Install a specific skill
npx skills add owner/repo --skill skill-name
```

### Evaluation-Driven Development

1. Run an agent on tasks **without** the skill — document where it fails
2. Create **3+ eval scenarios** from those failures
3. Establish **baseline scores**
4. Write **minimal instructions** to address the gaps
5. Re-evaluate — compare to baseline
6. Iterate until target quality is met

### The Two-Agent Refinement Pattern

- **Agent A** (designer) — writes or refines the skill
- **Agent B** (fresh instance, no prior context) — tests the skill on real tasks
- Observe Agent B's behavior, bring insights back to Agent A
- This catches assumptions that only work when you already know the answer

---

## What NOT To Do

- Write a skill without understanding the task domain first
- Create a skill for something the agent already does well without instructions
- Pack multiple unrelated capabilities into one skill
- Use vague trigger phrases in the description
- Exceed 500 lines in SKILL.md (move heavy content to resources/)
- Include time-sensitive information that will expire
- Skip testing — always verify with at least 3 real scenarios before publishing
