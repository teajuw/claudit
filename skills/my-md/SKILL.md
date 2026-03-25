---
name: my-md
description: List all CLAUDE.md and memory files with token sizes and scope (global/local). Quick terminal output, no artifact.
user_invocable: true
---

# /my-md — CLAUDE.md & Memory file sizes

Print a quick terminal summary of all markdown instruction and memory files that feed into Claude's context window.

## Instructions

When the user runs `/my-md`:

### 1. Determine target project
- Use the **current working directory** as the target project
- Encode the path for `~/.claude/projects/` lookup: replace `/` with `-`, drop leading `-`

### 2. Read and measure these files

**Global (○):**
- `~/.claude/CLAUDE.md`

**Local (●):**
- `<cwd>/CLAUDE.md`
- `<cwd>/.claude/CLAUDE.md`

**Memory (for target project):**
- Look up the encoded project path in `~/.claude/projects/<encoded>/`
- Read `memory/MEMORY.md` if it exists
- Read any memory files linked from MEMORY.md

For each file found, count words and estimate tokens (words × 1.3).

### 3. Print output

Print directly to terminal in this format:

```
/my-md · /Users/you/projects/helwa

── global ○ ──────────────────────────────
  CLAUDE.md              420 words  ~546 tok

── local ● ──────────────────────────────
  CLAUDE.md              185 words  ~241 tok
  .claude/CLAUDE.md      not found

── memory ● ─────────────────────────────
  MEMORY.md              32 words   ~42 tok
  user_role.md           18 words   ~23 tok
  feedback_testing.md    45 words   ~59 tok

── total ────────────────────────────────
  ~911 tokens injected into context
```

- Use `not found` for files that don't exist
- Show each memory file individually
- End with a total token estimate
- Keep it short — no HTML, no artifacts, just text
