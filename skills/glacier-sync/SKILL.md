---
name: glacier-sync
description: Syncs work tracking between the current repo and Glacier board. Auto-invoke after PR merge, branch creation, or when the user mentions board sync, card status, or Glacier tracking. Only activates in projects with a `.glacier.json` config file in the repo root.
---
# Glacier Sync

Keep the Glacier board in sync with development activity. This skill bridges GitHub workflow events to Glacier cards via MCP.

## Activation

This skill only activates when a `.glacier.json` file exists in the project root:

```json
{
  "enabled": true,
  "workspace": "blackbear",
  "project": "glacier",
  "mcp_url": "https://www.getglacier.ai/api/mcp"
}
```

If no `.glacier.json` is found, skip silently — do not prompt the user to create one.

## Capabilities

### 1. Card status sync (after PR merge or branch work)
- When a PR is merged, check if any linked Glacier cards should move to Done
- Match cards by: GitHub issue link on the card, branch name containing card prefix (e.g. `GLR-42`), or PR title referencing a card
- Use `Glacier:list_cards` → find matching card → `Glacier:update_card` to move column

### 2. Card creation from TODOs
- When the user asks to "sync TODOs" or "create cards from issues":
  - Scan for `// TODO(glacier):` comments in changed files
  - Create cards via `Glacier:create_card` with the TODO text as title
  - Report created cards with their IDs

### 3. Board status report
- When the user asks "what's on the board" or "board status":
  - Use `Glacier:list_columns` → `Glacier:list_cards` for the configured project
  - Report: cards per column, WIP limit status, any blockers
  - Flag columns at or over WIP limit

### 4. Link GitHub issues to cards
- After creating a GitHub issue from Claude Code, offer to link it to an existing or new Glacier card
- Use `Glacier:link_card_to_github` with the issue URL

## Column mapping

Default mapping (override in `.glacier.json` under `column_mapping`):

| GitHub event          | Glacier transition        |
|-----------------------|---------------------------|
| Branch created        | → In Progress             |
| PR opened             | → In Review               |
| PR merged             | → Done                    |
| PR closed (not merged)| No change (notify only)   |

## MCP tools used

- `Glacier:list_projects` — resolve project ID from slug
- `Glacier:list_columns` — get column IDs and WIP status
- `Glacier:list_cards` — find cards by project or column
- `Glacier:get_card` — check card details and linked GitHub issues
- `Glacier:update_card` — move cards between columns
- `Glacier:create_card` — create new cards from TODOs
- `Glacier:link_card_to_github` — link cards to GitHub issues/PRs

## Rules

- Never move a card without confirming the match is correct. If ambiguous, ask the user.
- Never create duplicate cards — check existing cards by title before creating.
- Always report what was synced: "Moved GLR-42 to Done (PR #87 merged)".
- Respect WIP limits — if the target column is at limit, warn instead of moving.
- Keep output concise: card ID, action taken, column. No verbose summaries.
