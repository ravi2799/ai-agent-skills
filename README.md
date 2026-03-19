# AI Agent Skills

A curated collection of provider-agnostic skills for AI agents. These skills work with any AI coding assistant or agent framework — Claude Code, Cursor, Windsurf, Cline, Aider, and more.

## What Are Skills?

Skills are structured instruction sets that give AI agents specialized capabilities. Each skill defines **when** it should activate, **what rules** to follow, and **how** to produce consistent, high-quality output for a specific task domain.

## Available Skills

| Skill | Description |
|---|---|
| [prompt-engineer](skills/prompt-engineer/) | Create, modify, and evaluate prompts using proven best practices |

## Repository Structure

```
ai-agent-skills/
  skills/
    prompt-engineer/
      SKILL.md
    <your-skill>/
      SKILL.md
```

## How to Use

### Claude Code
Copy the skill folder into `~/.claude/skills/` for global access:
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
