---
description: Implement a feature or fix using TDD workflow with agent delegation.
---
Follow the "Workflow: Implementing Issues (TDD)" section in CLAUDE.md exactly.
The user's request is: $ARGUMENTS

Start with step 0 (classify the issue as `logic`, `ui`, or `config`), then step 1 (branch creation).
After step 1, attempt step 1b: if glacier-sync is enabled and `.glacier.json` exists, move the linked card to In Progress. If glacier-sync is not available, skip silently and continue to step 2.
Remember: Red → Green → Refactor. Steps 4–7 apply to `logic` issues only.
