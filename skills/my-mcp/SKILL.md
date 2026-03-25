---
name: my-mcp
description: List active MCP servers with scope (global/local) and auth status. Quick terminal output, no artifact.
user_invocable: true
---

# /my-mcp — Active MCP servers

Print a quick terminal list of all configured MCP servers.

## Instructions

When the user runs `/my-mcp`:

### 1. Read MCP server configs

**Global (○):**
- Read `~/.claude/settings.json` → look for `mcpServers` key

**Local (●):**
- Read `<cwd>/.claude/settings.json` → look for `mcpServers` key

**Auth cache:**
- Read `~/.claude/mcp-needs-auth-cache.json` if it exists — shows which servers need re-auth

### 2. Extract server details

For each MCP server entry, extract:
- Server name (the key)
- Command/transport type (`command`, `url`, etc.)
- Whether it appears in the auth cache (needs re-auth)

### 3. Print output

```
/my-mcp · /Users/you/projects/helwa

── global ○ ──────────────────────────────
  ● Claude Preview     command: claude-preview
  ● Claude in Chrome   command: claude-chrome
  ● mcp-registry       command: mcp-registry

── local ● ──────────────────────────────
  ▲ supabase           command: supabase-mcp  (needs auth)

── total: 4 servers (3 global, 1 local) ──
```

- `●` green = working, `▲` orange = needs auth (found in auth cache)
- Show the command or URL for each server
- If no servers in a section, show `  (none)`
- End with count summary
- Do NOT show env vars, tokens, or API keys from server configs — only show the command name
