---
name: glacier-sync
description: Sync current repo activity with the Glacier board
---

Run the glacier-sync skill:

1. Check environment: `GLACIER_ENABLED` must be `true` and `GLACIER_PROJECT_ID` must be set. If not, tell the user to add them to `.env.local` and stop.
2. Resolve column IDs via `Glacier:list_columns` using the project ID from env.
3. Ask what to sync:
   - **Status**: Show board status (cards per column, WIP limits)
   - **PR sync**: Match recent merged PRs to cards, move to Done
   - **TODOs**: Scan for `// TODO(glacier):` comments in branch diff, create cards
   - **Link**: Link a specific GitHub issue to a Glacier card
4. Execute the selected action using Glacier MCP tools.
5. Report results concisely.
