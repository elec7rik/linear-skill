# Linear Skill for Claude Code

Fix your Linear issues without leaving the terminal.

> **Quick install:** ask Claude Code — *"Install this skill: https://github.com/elec7rik/linear-skill"*

## Install

1. Copy the skill file:
   ```bash
   mkdir -p ~/.claude/skills/linear
   curl -o ~/.claude/skills/linear/SKILL.md \
     https://raw.githubusercontent.com/elec7rik/linear-skill/main/SKILL.md
   ```

2. Add the Linear MCP server globally (run this once):
   ```bash
   python3 -c "
   import json, os
   p = os.path.expanduser('~/.claude.json')
   c = json.load(open(p)) if os.path.exists(p) else {}
   c.setdefault('mcpServers', {})['linear-server'] = {'type': 'http', 'url': 'https://mcp.linear.app/mcp'}
   json.dump(c, open(p, 'w'), indent=2)
   print('linear-server added globally')
   "
   ```

3. Restart Claude Code, then run `/mcp` to authenticate with Linear.

4. Run `/linear` — you're all set.

## Usage

- `/linear` — see all your assigned issues, triage them
- `/linear ENG-123` — jump straight to a specific issue

## What it does

- **Triage**: mark issues as Done or Won't Fix without leaving the terminal
- **Fix**: deploys an agent that explores your codebase, proposes a plan, and implements after your approval
- **Secure**: uses Linear's official MCP server with OAuth — no API keys stored anywhere

## Requirements

- [Claude Code](https://code.claude.com) CLI installed
- Linear account
