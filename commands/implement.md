---
description: Implement a feature or fix using TDD workflow with agent delegation.
---
Follow the "Workflow: Implementing Issues (TDD)" section in CLAUDE.md exactly.
The user's request is: $ARGUMENTS

Start with step 0 (classify the issue as `logic`, `ui`, or `config`), then step 1 (branch creation).
After step 1, attempt step 1b: if glacier-sync is enabled and `GLACIER_ENABLED=true` + `GLACIER_WORKSPACE_ID` + `GLACIER_PROJECT_ID` are set in the environment, move the linked card to In Progress. Always pass `workspace_id` to Glacier MCP calls. If glacier-sync is not available or env vars are missing, skip silently and continue to step 2.
After step 9, attempt step 9b: if glacier-sync is active and a PR was opened, move the linked card to In Review. Skip silently if not available.
Remember: Red → Green → Refactor. Steps 4–7 apply to `logic` issues only.
