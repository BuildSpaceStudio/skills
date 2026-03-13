# Contributing

Thanks for your interest in contributing to BuildSpace Skills! This guide covers how to add new skills.

## Adding a new skill

1. **Fork** this repository and create a branch.

2. **Create a directory** under `skills/` with your skill name:

   ```
   skills/your-skill-name/
     SKILL.md
   ```

3. **Write your `SKILL.md`** using the [template](template/SKILL.md) as a starting point. Every skill must have:
   - YAML frontmatter with `name` and `description` fields
   - Clear, actionable instructions that an AI agent can follow
   - Code examples with correct imports and types

4. **Test your skill** by copying it into a project's `.claude/skills/` directory and verifying the agent uses it correctly.

5. **Submit a pull request** with a brief description of what the skill teaches and when it should be used.

## Skill guidelines

- **Be specific.** Skills should give agents concrete code to produce, not vague guidance.
- **Include complete examples.** Every code block should be copy-pasteable with correct imports.
- **State when to use it.** The `description` field in frontmatter is what the agent sees when deciding whether to load the skill.
- **Keep it focused.** One skill per concern. If a skill is getting long, split it.
- **Add supporting files** (like `api-reference.md`) as siblings in the same directory if needed.

## Structure

```
skills/
  your-skill-name/
    SKILL.md              # Required — the skill definition
    api-reference.md      # Optional — supporting reference material
    examples/             # Optional — additional examples
```
