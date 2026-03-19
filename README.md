# AI Agent Skills

Skills that help AI agents build better AI agents — prompt engineering, architecture, evaluation, guardrails, and more. Provider-agnostic, works with any AI coding assistant or agent framework.

## Quick Install

Install all skills:
```bash
npx skills add ravi2799/ai-agent-skills
```

Install a specific skill:
```bash
npx skills add ravi2799/ai-agent-skills --skill prompt-engineer
```

## Available Skills

| Skill | Description |
|---|---|
| [prompt-engineer](skills/prompt-engineer/) | Create, modify, and evaluate prompts using proven best practices |
| [agent-architect](skills/agent-architect/) | Design, review, and debug multi-agent systems |
| [tool-designer](skills/tool-designer/) | Create, review, and convert tool/function definitions for agents |
| [eval-designer](skills/eval-designer/) | Build evaluation frameworks to measure agent quality |
| [guardrails](skills/guardrails/) | Add safety rails, input validation, and output checks to agents |
| [skill-creator](skills/skill-creator/) | Write new SKILL.md files that follow best practices |
| [model-gateway](skills/model-gateway/) | Configure and call LLMs through OpenRouter, LiteLLM, or direct APIs |
| [data-scientist](skills/data-scientist/) | Phase 0 — understand the problem and data before building (problem definition, EDA, profiling, signal vs noise) |
| [context-engineer](skills/context-engineer/) | Phase 0.5 — build context loading strategies and scripts for optimal token usage |
| [agent-builder](skills/agent-builder/) | Meta-skill — orchestrates the full agent build lifecycle end-to-end |
| [langgraph-orchestrator](skills/langgraph-orchestrator/) | Build stateful workflows with LangGraph — state machines, cycles, human-in-the-loop |

## What Are Skills?

Skills are structured instruction sets that give AI agents specialized capabilities. Each skill defines **when** it should activate, **what rules** to follow, and **how** to produce consistent, high-quality output for a specific task domain.

Compatible with the [skills CLI](https://skills.sh) from [vercel-labs/skills](https://github.com/vercel-labs/skills).

## Repository Structure

```
ai-agent-skills/
  skills/
    prompt-engineer/
      SKILL.md
    <your-skill>/
      SKILL.md
```

## Manual Installation

### Claude Code
```bash
cp -r skills/<skill-name> ~/.claude/skills/
```

### Other Agents
Each `SKILL.md` is a self-contained instruction file. You can:
- Include it in your system prompt
- Reference it as a project-level instruction file
- Paste it into your agent's custom instructions

## Adding a New Skill

1. Create a new folder under `skills/` with your skill name
2. Add a `SKILL.md` file with frontmatter:
   ```yaml
   ---
   name: your-skill-name
   description: One-line description of when this skill should trigger
   ---
   ```
3. Write the skill instructions in the body
4. Update the table in this README
5. Submit a pull request

## Contributing

Contributions are welcome! Please ensure your skill is:
- **Provider-agnostic** — works with any AI agent, not tied to a specific tool
- **Self-contained** — the SKILL.md file has everything needed
- **Well-structured** — clear triggers, rules, and examples

## License

MIT
