---
name: glacier-sync
description: Sync current repo activity with the Glacier board
---

Run the glacier-sync skill:

1. Check for `.glacier.json` in the repo root. If not found, tell the user and stop.
2. Read the config to get workspace, project, and MCP URL.
3. Ask what to sync:
   - **Status**: Show board status (cards per column, WIP limits)
   - **PR sync**: Match recent merged PRs to cards, move to Done
   - **TODOs**: Scan for `// TODO(glacier):` comments, create cards
   - **Link**: Link a specific GitHub issue to a Glacier card
4. Execute the selected action using Glacier MCP tools.
5. Report results concisely.
