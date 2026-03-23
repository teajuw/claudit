---
name: claudit
description: Audit all Claude Code configurations — generates an interactive HTML dashboard showing settings, skills, plugins, MCP servers, memory, and security health
user_invocable: true
---

# Claudit — Claude Code Configuration Audit

Generate an interactive HTML dashboard that audits all Claude Code configurations in `~/.claude/`.

## Instructions

When the user runs `/claudit`, follow these steps exactly:

### Step 1: Read all configuration files

Read the following files in parallel. If a file doesn't exist, skip it gracefully.

**Core settings:**
- `~/.claude/settings.json`
- `~/.claude/settings.local.json`

**Instructions:**
- `~/.claude/CLAUDE.md`
- The current project's `CLAUDE.md` (if it exists in the working directory or `.claude/CLAUDE.md`)

**Skills:**
- Glob for `~/.claude/skills/*/SKILL.md` — read each one

**Plugins:**
- `~/.claude/plugins/installed_plugins.json`
- `~/.claude/plugins/blocklist.json`

**MCP:**
- `~/.claude/mcp-needs-auth-cache.json`

**Projects & Memory:**
- List directories in `~/.claude/projects/`
- Glob for `~/.claude/projects/*/memory/MEMORY.md` — read each one
- For each MEMORY.md, also read any linked memory files it references

### Step 2: Sanitize all data

CRITICAL: Before including ANY data in the HTML output, apply these redaction rules:

- **Never include** values for JSON keys matching: `password`, `secret`, `token`, `key`, `credential`, `api_key`, `auth` (case-insensitive). Replace with `[REDACTED]`.
- **Never include** strings that look like tokens: starting with `sk-`, `ghp_`, `ghu_`, `xoxb-`, `xoxp-`, `AKIA`, `AIza`.
- **Never include** SSH connection strings (`ssh user@host`) — redact user and host.
- **Never include** database connection URLs with credentials — redact the password portion.
- **Never include** `.env` file contents or environment variable values that contain secrets.
- For memory files: **summarize** the content (what topics it covers, how many entries) rather than dumping raw text. Only include non-sensitive metadata.
- Safe keys to display as-is: `defaultMode`, `effortLevel`, `autoUpdatesChannel`, `voiceEnabled`, `statusLine`, `enabledPlugins`, `skipDangerousModePermissionPrompt`.

### Step 3: Estimate context token budget

For files that get injected into Claude's context window, estimate their token count:
- Count words in the file, multiply by 1.3 (rough token heuristic).
- Files that contribute to context:
  - `~/.claude/CLAUDE.md` (global — always injected)
  - Project `CLAUDE.md` (if exists — always injected for that project)
  - `MEMORY.md` + linked memory files for the current project
  - Skill description lines from SKILL.md frontmatter (all skills)
  - System reminders: estimate at ~400 tokens (cannot be directly measured)

### Step 4: Generate the HTML dashboard

Write a single self-contained HTML file to `/tmp/claudit-report.html`. The file must have NO external dependencies — all CSS and JS inline.

#### Design spec:

**Theme** — Dark mode:
```
--bg-primary: #0d1117
--bg-secondary: #161b22
--bg-card: #1c2128
--border: #30363d
--text-primary: #c9d1d9
--text-secondary: #8b949e
--accent: #58a6ff
--warning: #d29922
--danger: #f85149
--success: #3fb950
--purple: #bc8cff
```

**Layout:**
- Header: "CLAUDIT" title + subtitle "Claude Code Configuration Audit"
- Horizontal tab bar: Config | Skills | Plugins | MCP | Memory | CLAUDE.md
- Content area below tabs — switches on tab click
- "Run Full Report" button in top-right corner — toggles report sections

**Tab contents (default view — lightweight inventory):**

**Config tab:**
- Display each setting from settings.json as a labeled row
- Key settings to highlight: `permissions.defaultMode`, `effortLevel`, `voiceEnabled`, `autoUpdatesChannel`, `skipDangerousModePermissionPrompt`
- If `settings.local.json` has permission overrides, show them in a "Permission Overrides" subsection

**Skills tab:**
- For each skill found: show the name (from frontmatter) and description
- Format as: `/skillname` — description
- Show total count in the tab label: "Skills (N)"

**Plugins tab:**
- Two subsections: "Installed" and "Blocked"
- Installed: plugin name, version/scope if available, install date
- Blocked: plugin name, reason if available

**MCP tab:**
- List each MCP server from the auth cache
- Show name and auth status (pending/authenticated)
- Show timestamp of last auth attempt

**Memory tab:**
- Group by project (decode path-encoded names like `-Users-teajuw-projects-foo` to `/Users/teajuw/projects/foo`)
- For each project: show memory file count and a brief summary of topics covered
- Do NOT dump raw memory content

**CLAUDE.md tab:**
- Render the global `~/.claude/CLAUDE.md` content with basic HTML formatting
- If project-level CLAUDE.md files exist, list them below with their project path

**Full Report sections (hidden by default, revealed by "Run Full Report" button):**

When the button is clicked, toggle a CSS class that reveals these additional sections:

**Health Audit** — appears as a banner/section at the top of the page:
- Check `permissions.defaultMode` === "bypassPermissions" → CRITICAL badge
- Check `skipDangerousModePermissionPrompt` === true → CRITICAL badge
- Check if permission allowlist contains `sudo` → HIGH badge
- Check if permission allowlist contains unrestricted `curl` → MEDIUM badge
- Check blocked plugins count → INFO badge
- Badge colors: CRITICAL = `--danger`, HIGH = `--warning`, MEDIUM = yellow (#e3b341), INFO = `--accent`

**Context Budget** — appears as a section below Health:
- Single stacked horizontal bar chart showing all context contributors in one bar
- Each segment is proportional to its token count
- Color coding:
  - `#58a6ff` (blue) — Memory files
  - `#d29922` (orange) — System reminders (estimated)
  - `#3fb950` (green) — CLAUDE.md files
  - `#bc8cff` (purple) — Skill descriptions
  - `#484f58` (gray) — Settings
  - `#21262d` (dark) — Available/remaining context
- Below the bar: legend with color swatches and token counts
- Label: "Context Overhead (estimated, ±20%)" with total token count
- Implementation: CSS flexbox container, each child div gets `flex-grow` proportional to token count. Hover shows tooltip with exact count.

**Optimize Suggestions** — appears below Context Budget:
- If memory exceeds 500 tokens: "Consider trimming MEMORY.md — currently ~N tokens"
- If `bypassPermissions` enabled: "Consider switching to scoped permission allows for better security"
- If `sudo` in allowlist: "Consider restricting sudo to specific commands instead of wildcard"
- If blocked plugins exist: "Review blocked plugins — are the blocks still needed?"
- If no backups found or stale: "Consider enabling regular backups"

#### HTML structure:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Claudit — Config Audit</title>
  <style>
    /* All CSS here — dark theme, tabs, cards, bar chart, badges */
  </style>
</head>
<body>
  <header><!-- Title, subtitle, Run Full Report button --></header>
  <nav><!-- Tab bar --></nav>
  <main><!-- Tab content panels --></main>
  <section class="report-section hidden"><!-- Health, Context, Optimize --></section>
  <script>
    // Tab switching logic
    // Report toggle logic
    // Tooltip logic for context bar
  </script>
</body>
</html>
```

Keep the entire HTML under 300 lines. Clean, minimal, no bloat.

### Step 5: Open the dashboard

After writing the file, run:
```bash
open /tmp/claudit-report.html
```

Then tell the user: "Dashboard opened in browser. Click 'Run Full Report' for health audit, context budget, and optimization suggestions."

## Important notes

- This skill generates a SNAPSHOT — the HTML reflects the state at generation time
- If the user wants to see updated data, they should run `/claudit` again
- NEVER include credentials, tokens, passwords, or secrets in the generated HTML
- The HTML file is written to /tmp and is ephemeral — it will be cleaned up by the OS
- If on Linux, use `xdg-open` instead of `open`. Check the platform first.
