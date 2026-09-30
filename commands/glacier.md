---
name: glacier
description: Manual Glacier board operations — board status, PR sync, TODO scanning, issue linking. Workflow transitions (In Progress, In Review) are called explicitly by the implement agent, /implement-v and /triage; use this command for status, Done after merge, TODOs and linking.
---

Run the glacier-sync skill for manual operations:

1. Check environment: `GLACIER_ENABLED` must be `true`, `GLACIER_WORKSPACE_ID` and `GLACIER_PROJECT_ID` must be set. If not, tell the user to add them to `.env.local` and stop.
2. Resolve column IDs via `Glacier:list_columns` using `project_id` and `workspace_id` from env.
3. Ask what to sync:
   - **Status**: Show board status (cards per column, WIP limits)
   - **PR sync**: Match recent merged PRs to cards, move to Done
   - **TODOs**: Scan for `// TODO(glacier):` comments in branch diff, create cards
   - **Link**: Link a specific GitHub issue to a Glacier card
4. Execute the selected action using Glacier MCP tools. Always pass `workspace_id` from env.
5. Report results concisely.
