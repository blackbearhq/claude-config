---
name: glacier-sync
description: Syncs work tracking between the current repo and Glacier board. Auto-invoke after branch creation (step 1 of implement workflow), PR open, PR merge, or when the user mentions board sync, card status, or Glacier tracking. Only activates in projects with a `.glacier.json` config file in the repo root.
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

## Auto-invoke triggers

This skill should fire automatically at these workflow moments:

| Trigger | When | Action |
|---------|------|--------|
| **Branch creation** | Step 1 of `/implement` workflow | Move linked card → **In Progress** |
| **PR opened** | After `gh pr create` | Move linked card → **In Review** |
| **PR merged** | After PR merge confirmed | Move linked card → **Done** |
| **Manual** | User runs `/glacier-sync` | Interactive menu (see Capabilities §3) |

### Branch creation → In Progress (the key transition)

This is the most important auto-trigger for accurate cycle time tracking. When the implement workflow creates a feature branch:

1. Parse the branch name or the issue reference from the user's request
2. Find the matching Glacier card (by GitHub issue link, card prefix in branch name, or title match)
3. Check the card's current column:
   - If in **Backlog** or **Ready** → move to **In Progress**
   - If already in **In Progress** or later → do nothing (don't regress)
4. Check WIP limit on In Progress column before moving. If at limit, **warn the user** and ask whether to proceed
5. Report: `Moved GLACIE-42 → In Progress (branch feat/issue-42 created)`

**Why this matters:** Cycle time in Kanban is measured from when work enters the first active column to when it reaches Done. Moving the card at branch creation — not at spec approval, not at PR merge — gives accurate lead time data. The Ready column represents "approved and waiting to be pulled"; In Progress represents "someone is actively working on it."

## Capabilities

### 1. Card status sync (after PR merge or branch work)
- When a PR is merged, check if any linked Glacier cards should move to Done
- Match cards by: GitHub issue link on the card, branch name containing card prefix (e.g. `GLACIE-42`), or PR title referencing a card
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

| GitHub event          | Glacier transition              |
|-----------------------|---------------------------------|
| Branch created        | Ready / Backlog → In Progress   |
| PR opened             | In Progress → In Review         |
| PR merged             | In Review → Done                |
| PR closed (not merged)| No change (notify only)         |

## Card matching strategy

When looking for the Glacier card to move, try these in order:

1. **GitHub issue link** — card has a linked GitHub issue URL that matches the issue being implemented
2. **Card prefix in branch name** — branch contains `GLACIE-42` or similar card ID
3. **Issue number in branch name** — branch contains `issue-42` or `#42`, match against cards with GitHub issue #42 linked
4. **Title match** — fuzzy match between issue title and card title (last resort, ask for confirmation)

If no match is found, ask the user: "I couldn't find a Glacier card for this work. Want me to create one?"

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
- Always report what was synced: "Moved GLACIE-42 to Done (PR #87 merged)".
- Respect WIP limits — if the target column is at limit, warn instead of moving.
- Keep output concise: card ID, action taken, column. No verbose summaries.
- Never regress a card (e.g. don't move from In Review back to In Progress).
