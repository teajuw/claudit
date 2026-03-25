---
name: claudit
description: Audit all Claude Code configurations — generates an interactive HTML dashboard showing settings, skills, plugins, MCP servers, memory, and security health
user_invocable: true
---

# Claudit — Claude Code Configuration Audit

Generate an interactive HTML dashboard that audits all Claude Code configurations in `~/.claude/`.

## Usage

- `/claudit` — audit from the current working directory
- `/claudit <path>` — audit from a specific project directory (e.g., `/claudit ~/helwa-ai`)

## Instructions

When the user runs `/claudit`, follow these steps exactly:

### Step 0: Determine the target project

- If the user passed a path argument (e.g., `/claudit ~/helwa-ai`), use that as the **target project directory**. Resolve `~` to the home directory.
- If no argument, use the **current working directory** as the target project directory. This means running `/claudit` from `~/projects/helwa` audits helwa, and running it from `~/projects/claudit` audits claudit — no arguments needed.
- To find the matching Claude project config, convert the absolute path to the encoded format used in `~/.claude/projects/`: replace all `/` with `-` and drop the leading `-` if needed. For example, `/Users/john/helwa-ai` → `-Users-john-helwa-ai`.
- Look for this encoded directory in `~/.claude/projects/`. If it exists, use its memory files and config. If not, the project simply has no Claude memory yet.
- Also check for a `CLAUDE.md` file in the target project directory itself (at the project root or in `.claude/CLAUDE.md`).

### Step 0.5: Detect the active model

- Run `echo $CLAUDE_MODEL` or check the environment to determine the active model. Alternatively, infer from the system prompt (you know what model you are).
- Map the model to its context window size:
  - `claude-opus-4-6` (with 1M context) → 1,000,000 tokens
  - `claude-opus-4-6` (standard) → 200,000 tokens
  - `claude-sonnet-4-6` → 200,000 tokens
  - `claude-haiku-4-5` → 200,000 tokens
  - Default fallback → 200,000 tokens
- Store both the model name and context limit for use in the dashboard.

### Step 1: Read all configuration files

Read the following files in parallel. If a file or directory doesn't exist, skip it gracefully — many users won't have all of these. The only guaranteed file is `~/.claude/settings.json`. Show "none found" in the dashboard for any empty section rather than omitting the tab.

**Core settings (global ○):**
- `~/.claude/settings.json`
- `~/.claude/settings.local.json`

**Core settings (local ●):**
- `<target>/.claude/settings.json` — project-level setting overrides
- `<target>/.claude/settings.local.json` — project-level permission overrides

**Instructions (global ○):**
- `~/.claude/CLAUDE.md`

**Instructions (local ●):**
- `<target>/CLAUDE.md` and/or `<target>/.claude/CLAUDE.md`

**Skills (global ○):**
- Glob for `~/.claude/skills/*/SKILL.md`

**Skills (local ●):**
- Glob for `<target>/.claude/skills/*/SKILL.md`
- In the dashboard, show skills with scope labels (○ global / ● local) and use dividers if both exist

**Plugins:**
- `~/.claude/plugins/installed_plugins.json`
- `~/.claude/plugins/blocklist.json`

**MCP (global ○):**
- `~/.claude/mcp-needs-auth-cache.json`
- MCP servers defined in `~/.claude/settings.json` under `mcpServers`

**MCP (local ●):**
- MCP servers defined in `<target>/.claude/settings.json` under `mcpServers`

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
  - `MEMORY.md` + linked memory files for the target project (found via the encoded path in `~/.claude/projects/`)
  - Skill description lines from SKILL.md frontmatter (all skills)
  - System reminders: estimate at ~400 tokens (cannot be directly measured)
- Sum all token estimates to get total consumed tokens.
- Calculate the percentage: `(total consumed / model context limit) × 100`.
- The context budget label should show: "Current project: <target directory basename>" with the model name and percentage prominently displayed.
- Format the percentage as: "~X.X% of <model context limit formatted> tokens (<model name>)"
- Example: "~0.8% of 1M tokens (Opus 4.6)" or "~4.2% of 200K tokens (Sonnet 4.6)"

### Step 4: Generate the HTML dashboard

Write a single self-contained HTML file to `/tmp/claudit-report.html`. The file must have NO external dependencies — all CSS and JS inline.

#### Design spec:

**Theme** — "Hybrid TUI" — warm dark terminal aesthetic with rounded box-drawing characters:
```
--bg-primary: #292524
--bg-secondary: #1f1e1d
--bg-card: #353130
--border: #4a4544
--text-primary: #ece7df
--text-secondary: #a39e93
--accent: #E8734A
--accent-light: #F09A7A
--accent-subtle: rgba(232, 115, 74, 0.15)
--warning: #E8A84A
--danger: #E84A4A
--success: #7AC87A
--purple: #B87AE8
```

The feel is a polished terminal UI — like a luxury ncurses/Textual app rendered in HTML. Monospace throughout, rounded box-drawing characters (╭╮╰╯│─) for structure, warm dark background. Orange is used sparingly: active tab, severity indicators, interactive elements.

**Typography:**
- **Everything is monospace**: `'Söhne Mono', 'SF Mono', 'Fira Code', 'Fira Mono', 'Menlo', monospace`
- No sans-serif fonts. The entire dashboard is monospace to feel like a terminal extension.
- Base font size: `14px`, line-height `1.5`
- Header "claudit" — lowercase, `font-size: 1.4rem`, `font-weight: 600`, color `--text-primary` (NOT orange — keep the logo neutral)
- Path and model info in `--text-secondary`, same font size as body

**Global vs Local scope system:**

This is a consistent visual language used across ALL tabs:
- **Global** items (from `~/.claude/`): displayed with a hollow dot `○` prefix and `--text-secondary` scope label "global"
- **Local** items (from `<project>/.claude/` or project root): displayed with a filled dot `●` prefix and `--accent` scope label "local"
- In cards that contain both global and local items, show them in separate sections with a thin divider
- The context bar legend also uses ○/● to distinguish global vs local token contributions

**Layout:**
- Max width: `900px`, centered with `margin: 0 auto`, `padding: 32px 24px`
- Header inside a TUI box (rounded corners using CSS `border-radius: 8px` with `border: 1px solid var(--border)`):
  ```
  ╭───────────────────────────────────────╮
  │  claudit                              │
  │  /Users/teajuw/projects/claudit       │
  │  opus 4.6 · 1M tokens                │
  ╰───────────────────────────────────────╯
  ```
  - Use actual CSS borders (not box-drawing chars in text) to create this effect
  - `background: var(--bg-card)`, `border: 1px solid var(--border)`, `border-radius: 8px`, `padding: 16px 20px`
  - `margin-bottom: 16px`
- Tab bar: horizontal row below the header box, no border
  - Active tab: wrapped in brackets `[overview]` with `color: var(--accent)`
  - Inactive tabs: just the name, `color: var(--text-secondary)`, on hover `color: var(--text-primary)`
  - Tabs spaced with `gap: 16px`, same monospace font
  - `margin-bottom: 16px`
- Content area: directly below tabs
- **Overview is the default tab**

**Card/Box style** (used for every content block):
- Styled to look like TUI panels: `background: var(--bg-card)`, `border: 1px solid var(--border)`, `border-radius: 8px`
- `padding: 16px 20px`, `margin-bottom: 12px`
- Section label embedded in the top border area: render the label text in a small span with `background: var(--bg-primary)` positioned over the top border (using negative margin or relative positioning), creating the effect of `╭─ context ─────────╮`
- `font-size: 0.75rem`, `text-transform: uppercase`, `letter-spacing: 0.06em`, `color: var(--text-secondary)`

**Overview tab (default):**

Context box:
- Box label: "context"
- Inside: percentage in large text (`font-size: 1.6rem`, `font-weight: 600`, `color: var(--text-primary)`) — e.g. "~0.1%"
- Same line or below: "of 1M tokens" in `--text-secondary`
- Progress bar using block characters: `█` for filled, `░` for empty, rendered as styled spans
  - Or use a CSS bar: `height: 8px`, `border-radius: 4px`, `overflow: hidden`, `background: var(--bg-secondary)`
  - Segments proportional to tokens, clickable, `onclick` calls `go('tabname')`
  - Colors: `--accent` for CLAUDE.md (split into ○ global / ● local segments), `--purple` for Skills, `--warning` for System, `#6BA3D6` for Memory
- Legend below: inline items using ○/● dots with scope indication
  - `○ claude.md 559 global  ● claude.md 403 local  ○ skills 130  ○ system 400  ● memory 0`
  - Clickable — navigates to corresponding tab

Health + Suggestions — **2-column grid** (`grid-template-columns: 1fr 1fr`, `gap: 12px`):
- Left box: label "health" — list of severity items:
  - Use symbols: `■` CRITICAL (colored `--danger`), `▲` HIGH (colored `--warning`), `▲` MEDIUM (colored `#e3b341`), `·` INFO (colored `--accent`)
  - Each on its own line: `■ CRITICAL  bypassPermissions enabled`
- Right box: label "suggestions" — list with `·` prefix in `--accent`:
  - `· scope permissions for better security`
  - `· restrict sudo to specific commands`
- On narrow screens (`@media max-width: 640px`): stack to single column

**Config tab:**
- Box label: "settings.json"
- Rows as key-value pairs: `key .............. value` (dots as visual filler, or just spaced with flexbox justify-content: space-between)
- Dangerous values colored `--danger`
- Second box: "permission overrides · settings.local.json" with the same row format
- Show scope: settings.json items as `○ global`, project settings.json as `● local`

**Skills tab:**
- Box label: "skills (N)" where N = total count
- Split into two sections if both exist:
  - `── global ──` divider, then global skills listed as `/name — description`
  - `── local ──` divider, then project-local skills
- Skill names in `--accent` monospace
- If only global skills exist, skip the divider — just list them

**Plugins tab:**
- Two boxes: "installed (N)" and "blocked (N)"
- Each entry: name, scope, date info in `--text-secondary`

**MCP tab:**
- Box label: "mcp servers (N)"
- Each server: `● name` (green dot = authenticated, orange dot = pending), status and timestamp in `--text-secondary`
- Show scope: `○ global` for servers in `~/.claude/settings.json`, `● local` for project-level servers

**Memory tab:**
- Box label: "target project memory"
- Show memory for the target project (or "no memory files" if none)
- Second box: "all projects with memory (N)" — list each with decoded path, file count, topic summary
- Third box: "all registered projects (N)" — simple list of decoded paths
- Do NOT dump raw memory content — summarize only

**CLAUDE.md tab:**
- Two boxes with scope labels:
  - "claude.md · global" — render `~/.claude/CLAUDE.md` content in a `<pre>` block
  - "claude.md · local" — render project-level CLAUDE.md in a `<pre>` block, or "no project-level CLAUDE.md found"
- Pre blocks: `background: var(--bg-secondary)`, `border-radius: 4px`, `padding: 12px`, monospace

**Health Audit** (on Overview tab, left column):
- Check `permissions.defaultMode` === "bypassPermissions" → `■ CRITICAL` in `--danger`
- Check `skipDangerousModePermissionPrompt` === true → `■ CRITICAL` in `--danger`
- Check if permission allowlist contains `sudo` → `▲ HIGH` in `--warning`
- Check if permission allowlist contains unrestricted `curl` → `▲ MEDIUM` in `#e3b341`
- Check blocked plugins count → `· INFO` in `--accent`
- If memory contains credentials/tokens → `▲ HIGH` in `--warning`

**Optimize Suggestions** (on Overview tab, right column):
- Each suggestion prefixed with `·` in `--accent`
- If memory exceeds 500 tokens: "trim MEMORY.md — currently ~N tokens"
- If `bypassPermissions` enabled: "scope permissions for better security"
- If `sudo` in allowlist: "restrict sudo to specific commands"
- If blocked plugins exist: "review blocked plugins"
- If memory contains credentials: "move credentials to env vars"
- If no issues: "no suggestions — config looks clean" in `--text-secondary`

Keep the entire HTML under 400 lines. Every element monospace, terminal-native, intentional.

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
Project: ~/helwa-ai (or whatever the target was)

File: /tmp/claudit-report.html
Run /claudit again to refresh, or /claudit <path> to audit a different project.
```

Always include the file path so the user can reopen it if the browser tab is closed.

## Important notes

- This skill generates a SNAPSHOT — the HTML reflects the state at generation time
- If the user wants to see updated data, they should run `/claudit` again
- NEVER include credentials, tokens, passwords, or secrets in the generated HTML
- The HTML file is written to /tmp and is ephemeral — it will be cleaned up by the OS
- Print progress updates during each step so the user has transparency into what's happening
