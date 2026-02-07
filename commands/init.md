---
description: Initialize a new Claude Code session with project context. Run at the start of every session.
---
Read CLAUDE.md and the project's package.json.
Summarize:
1. Project name and purpose (one line)
2. Key dependencies
3. Available npm scripts
4. Last 3 git commits (`git log -3 --oneline`)
5. Any pending TODO items in the codebase (`grep -r "TODO" --include="*.ts" --include="*.tsx" -l`)

Keep the summary under 15 lines. Do NOT explore the full codebase — just orient.
