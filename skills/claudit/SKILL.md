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

Read the following files in parallel. If a file or directory doesn't exist, skip it gracefully — many users won't have all of these. The only guaranteed file is `~/.claude/settings.json`. Show "none found" in the dashboard for any empty section rather than omitting the tab.

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
- Horizontal tab bar: **Overview** | Config | Skills | Plugins | MCP | Memory | CLAUDE.md
- Content area below tabs — switches on tab click
- **Overview is the default tab** — shows context budget bar, health audit, and optimization suggestions

**Overview tab (default):**
- **Context Budget** — stacked horizontal color-coded bar chart at the top. This is the hero element.
  - Each segment proportional to token count. Colors: green=CLAUDE.md, purple=Skills, orange=System reminders, gray=Settings, blue=Memory, dark=Available
  - **Bar segments are clickable** — clicking a segment navigates to the corresponding detail tab (e.g., click CLAUDE.md segment → CLAUDE.md tab). Show "view →" hint on hover.
  - **Legend items are also clickable** — same navigation behavior
  - Implementation: CSS flexbox, each child div gets `flex-grow` proportional to tokens. `onclick` calls `go('tabname')`.
- **Health Audit** — severity-rated warnings below the bar
- **Optimize Suggestions** — actionable recommendations below health

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
- Group by project (decode path-encoded directory names by replacing leading `-` with `/` and internal `-` that follow a `/` with `/`, e.g., `-Users-john-projects-myapp` → `/Users/john/projects/myapp`)
- For each project: show memory file count and a brief summary of topics covered
- Do NOT dump raw memory content

**CLAUDE.md tab:**
- Render the global `~/.claude/CLAUDE.md` content with basic HTML formatting
- If project-level CLAUDE.md files exist, list them below with their project path

**Health Audit** (on Overview tab, below context bar):
- Check `permissions.defaultMode` === "bypassPermissions" → CRITICAL badge
- Check `skipDangerousModePermissionPrompt` === true → CRITICAL badge
- Check if permission allowlist contains `sudo` → HIGH badge
- Check if permission allowlist contains unrestricted `curl` → MEDIUM badge
- Check blocked plugins count → INFO badge
- Badge colors: CRITICAL = `--danger`, HIGH = `--warning`, MEDIUM = yellow (#e3b341), INFO = `--accent`

**Optimize Suggestions** (on Overview tab, below health):
- If memory exceeds 500 tokens: "Consider trimming MEMORY.md — currently ~N tokens"
- If `bypassPermissions` enabled: "Consider switching to scoped permission allows for better security"
- If `sudo` in allowlist: "Consider restricting sudo to specific commands instead of wildcard"
- If blocked plugins exist: "Review blocked plugins — are the blocks still needed?"
- If no backups found or stale: "Consider enabling regular backups"

Keep the entire HTML under 300 lines. Clean, minimal, no bloat.

### Step 5: Progress updates and output

**CRITICAL: Print progress updates as you work so the user knows it's not stuck.**

Throughout Steps 1-4, print short status lines as you go:

```
Reading settings...
Reading skills (5 found)...
Reading plugins...
Reading MCP config...
Scanning 13 projects for memory files...
Estimating context budget...
Generating dashboard...
```

After writing the HTML file, open it:
```bash
open /tmp/claudit-report.html
```
(On Linux, use `xdg-open` instead of `open`. Check the platform first.)

Then print the final message with a file link:

```
Dashboard opened in browser.

File: /tmp/claudit-report.html
Run /claudit again to refresh.
```

Always include the file path so the user can reopen it if the browser tab is closed.

## Important notes

- This skill generates a SNAPSHOT — the HTML reflects the state at generation time
- If the user wants to see updated data, they should run `/claudit` again
- NEVER include credentials, tokens, passwords, or secrets in the generated HTML
- The HTML file is written to /tmp and is ephemeral — it will be cleaned up by the OS
- Print progress updates during each step so the user has transparency into what's happening
