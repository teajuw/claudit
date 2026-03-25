---
name: my-hooks
description: List active hooks with event triggers, commands, and scope (global/local). Quick terminal output, no artifact.
user_invocable: true
---

# /my-hooks — Active hooks

Print a quick terminal list of all configured hooks from settings files.

## Instructions

When the user runs `/my-hooks`:

### 1. Read hook configs

**Global (○):**
- Read `~/.claude/settings.json` → look for `hooks` key
- Read `~/.claude/settings.local.json` → look for `hooks` key

**Local (●):**
- Read `<cwd>/.claude/settings.json` → look for `hooks` key
- Read `<cwd>/.claude/settings.local.json` → look for `hooks` key

### 2. Extract hook details

Hooks are organized by event type. Common events:
- `PreToolUse` — runs before a tool is called
- `PostToolUse` — runs after a tool returns
- `Notification` — runs on notifications
- `Stop` — runs when Claude stops responding
- `SubagentStop` — runs when a subagent finishes

Each hook has:
- `matcher` — which tool/event it matches (e.g., `Bash`, `Edit`, `*`)
- `hooks` array — each with a `type` (usually `command`) and `command` string

### 3. Print output

```
/my-hooks · /Users/you/projects/helwa

── global ○ ──────────────────────────────
  PreToolUse
    Bash        → lint-check.sh
    Edit        → format-on-save.sh
  PostToolUse
    *           → log-tool-use.sh
  Stop
    *           → notify-done.sh

── local ● ──────────────────────────────
  PreToolUse
    Bash        → ./scripts/validate-env.sh

── total: 5 hooks (4 global, 1 local) ──
```

- Group by event type, then show matcher → command
- If no hooks in a section, show `  (none)`
- If no hooks at all, show `  no hooks configured`
- Do NOT show full file paths if they contain usernames — use relative paths or basenames
- End with count summary
