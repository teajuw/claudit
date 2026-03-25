---
name: my-skills
description: List all installed skills with name, description, and scope (global/local). Quick terminal output, no artifact.
user_invocable: true
---

# /my-skills — Installed skills inventory

Print a quick terminal list of all skills available in this session.

## Instructions

When the user runs `/my-skills`:

### 1. Find all skills

**Global (○):**
- Glob for `~/.claude/skills/*/SKILL.md`

**Local (●):**
- Glob for `<cwd>/.claude/skills/*/SKILL.md`

### 2. Extract metadata

For each SKILL.md, read the YAML frontmatter to get:
- `name` — the skill name (also the directory name)
- `description` — one-line description

### 3. Print output

```
/my-skills · /Users/you/projects/helwa

── global ○ ──────────────────────────────
  /idea         park an idea as GitHub Issue
  /groom        triage the idea parking lot
  /claudit      configuration audit dashboard
  /my-md        CLAUDE.md & memory file sizes
  /my-skills    this command
  /my-mcp       active MCP servers
  /my-hooks     active hooks

── local ● ──────────────────────────────
  /deploy       run CI pipeline for helwa

── total: 8 skills (7 global, 1 local) ──
```

- Skill name in `/name` format
- Description after the name, aligned with spacing
- If no local skills exist, show `  (none)` under the local section
- End with a count summary
