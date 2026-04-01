# .Agents

A collection of AI skills, prompts, and agent configurations maintained by [@solanticai](https://github.com/solanticai).

## Overview

This repository serves as a centralized library of reusable AI assets — skills, system prompts, tool definitions, and agent workflows. These are designed to be portable across AI coding assistants and agent frameworks.

## Structure

```
.Agents/
├── skills/          # Standalone skill definitions (prompts, instructions, workflows)
│   └── *.md         # Individual skill files with frontmatter metadata
├── prompts/         # System prompts and prompt templates
├── tools/           # Tool/function definitions and schemas
└── workflows/       # Multi-step agent workflows and orchestration configs
```

## Skills

Skills are self-contained instruction sets that can be loaded into AI assistants to give them specialized capabilities.

| Skill | Description |
|-------|-------------|
| `project-audit` | Multi-phase project completion audit covering task verification, code quality, bugs, types, security, performance, and build checks |

## Usage

Skills in this repo use a standard frontmatter format:

```yaml
---
name: skill-name
description: When and how to use this skill.
---
```

The description field doubles as a trigger — it tells the AI assistant which user intents should activate the skill.

## Adding a New Skill

1. Create a new `.md` file in the appropriate directory (e.g., `skills/`)
2. Add frontmatter with `name` and `description`
3. Write clear, structured instructions the AI can follow
4. Include phases, checklists, or decision trees where applicable

## License

MIT
