---
description: Implement a feature or fix using TDD workflow with agent delegation.
---
Follow the "Workflow: Implementing Issues (TDD)" section in CLAUDE.md exactly.
The user's request is: $ARGUMENTS

## Step 0a — Parse entry mode

The first token of $ARGUMENTS determines the entry path:

### Mode 1: `card <card_id>`
The user passed a Glacier card UUID.

1. Check glacier-sync prerequisites: `GLACIER_ENABLED=true`, `GLACIER_WORKSPACE_ID`, `GLACIER_PROJECT_ID` must all be set. If not, stop and tell the user: "Glacier env vars required for card mode. Add GLACIER_ENABLED, GLACIER_WORKSPACE_ID, and GLACIER_PROJECT_ID to .env.local."
2. Call `Glacier:get_card(card_id, workspace_id: $GLACIER_WORKSPACE_ID)` → get title, description, linked docs.
3. Call `Glacier:get_card_github_status(card_id, workspace_id: $GLACIER_WORKSPACE_ID)` → check for linked GitHub issues.
4. **If a GitHub issue is linked:**
   - Extract owner, repo, and issue number from the linked URL.
   - Call `github:get_issue(owner, repo, issue_number)` → pull the full issue body (implementation details, acceptance criteria, affected files, schema changes).
   - Use the **issue body** as the implementation spec. Use the issue title + number for branch naming (e.g. `feat/issue-42-label-crud`).
   - The card provides board context (priority, column, linked docs). The issue provides implementation details.
5. **If no GitHub issue is linked:**
   - Use the card title and description as the implementation brief.
   - Ask the user: "No GitHub issue linked to this card. Want me to create one before starting, or proceed with the card description only?"
   - If proceeding without an issue, use the card title for branch naming.
6. Store the resolved card_id for use in steps 1b and 9b (board sync).

### Mode 2: `issue <issue_number>`
The user passed a GitHub issue number (e.g. `issue 42` or `issue #42`).

1. Strip any `#` prefix from the issue number.
2. Determine the repo: use the current working directory's git remote. Parse owner and repo from the remote URL. If ambiguous, default to `blackbearhq` + current repo name.
3. Call `github:get_issue(owner, repo, issue_number)` → pull the full issue body.
4. Use the issue body as the implementation spec. Use issue title + number for branch naming.
5. **If glacier-sync is enabled**, attempt to find the linked Glacier card:
   - Call `Glacier:list_cards(project_id: $GLACIER_PROJECT_ID, workspace_id: $GLACIER_WORKSPACE_ID)`
   - For each card, call `Glacier:get_card_github_status` to find a card linked to this issue URL.
   - If found, store the card_id for board sync in steps 1b and 9b.
   - If not found, continue without board sync. Do not create a card automatically.

### Mode 3: custom prompt (default)
Anything that doesn't start with `card` or `issue` is treated as a free-form description.

1. Use $ARGUMENTS as the implementation brief directly.
2. No Glacier card lookup. No GitHub issue lookup.
3. Board sync (steps 1b, 9b) will only fire if glacier-sync can match the work to a card later (e.g. via branch name matching).

---

After step 0a, proceed to step 0 (classify as `logic`, `ui`, or `config`), then step 1 (branch creation).

After step 1, attempt step 1b: if glacier-sync is enabled and a card_id was resolved in step 0a (or matched via issue link), move the card to In Progress. Always pass `workspace_id` to Glacier MCP calls. If glacier-sync is not available, env vars are missing, or no card was resolved, skip silently and continue to step 2.

After step 9, attempt step 9b: if glacier-sync is active, a PR was opened, and a card_id was resolved, move the card to In Review. Skip silently if not available.

Remember: Red → Green → Refactor. Steps 4–7 apply to `logic` issues only.
