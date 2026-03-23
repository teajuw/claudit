# Claudit

A Claude Code skill that audits your entire `~/.claude/` configuration and generates an interactive HTML dashboard.

## The Problem

Claude Code silently configures settings, skills, plugins, MCP servers, memory, and CLAUDE.md prompts across `~/.claude/`. Over time, you lose track of what's active, what's eating your context window, and what might be a security risk.

## What It Does

Run `/claudit` in any Claude Code session. It reads all your config files and generates a self-contained HTML dashboard with:

- **Config** — permission mode, effort level, voice, update channel, dangerous mode settings
- **Skills** — every installed skill with its description
- **Plugins** — installed and blocked plugins
- **MCP Servers** — connected servers and auth status
- **Memory** — per-project memory files (summarized, never raw)
- **CLAUDE.md** — your global and project-level instructions

Click **"Run Full Report"** to reveal:
- **Health Audit** — security warnings with severity badges (CRITICAL/HIGH/MEDIUM/INFO)
- **Context Budget** — color-coded stacked bar showing what's eating your context window
- **Optimize** — actionable suggestions to trim bloat and harden security

## Security

- Credentials, tokens, passwords, and secrets are **never included** in the HTML output
- Memory files are **summarized**, not dumped raw
- The HTML file is written to `/tmp/` (ephemeral)
- No external dependencies, no network requests, no telemetry
- Everything runs locally in your Claude Code session

## Install

```bash
cp -r skills/claudit ~/.claude/skills/claudit
```

Then run `/claudit` in any Claude Code session.

## Requirements

- Claude Code (any recent version)
- A `~/.claude/` directory (created automatically by Claude Code)

## License

MIT
