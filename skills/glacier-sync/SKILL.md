---
name: glacier-sync
description: Hooks-based Glacier board sync. Auto-fires on branch creation, PR open, and PR merge via Claude Code hooks. Only activates when GLACIER_ENABLED=true, GLACIER_WORKSPACE_ID, and GLACIER_PROJECT_ID are set. Skips silently if not configured.
model: haiku
effort: low
tools: Bash, Read
hooks:
  FileChanged:
    if: |
      # Fires when .git/HEAD changes — signals branch switch / creation
      test "$FILE_PATH" = ".git/HEAD"
  PostCompact: true
  CwdChanged: true
---
# Glacier Sync (hooks-based)

Keeps the Glacier board in sync with repo activity. Wires into Claude Code hooks instead of manual workflow steps — more reliable, zero cognitive load.

**Fully optional.** If env vars are missing, skip silently. Never block the parent workflow.

## Activation conditions

All four must be true:
1. Skill is enabled in the session
2. `GLACIER_ENABLED=true`
3. `GLACIER_WORKSPACE_ID` is set
4. `GLACIER_PROJECT_ID` is set

If any fails → silent skip.

## Configuration

In `.env.local` (gitignored):

```
GLACIER_ENABLED=true
GLACIER_WORKSPACE_ID=<uuid from Project Settings>
GLACIER_PROJECT_ID=<uuid from Project Settings>
```

MCP server URL is hardcoded: `https://www.getglacier.ai/api/mcp`

Column IDs resolve at runtime via `Glacier:list_columns` — no stored IDs, board can be restructured freely.

## Hook triggers

### `FileChanged` on `.git/HEAD`
Fires when branch is created or switched.

1. Read new HEAD: `git rev-parse --abbrev-ref HEAD`
2. Extract issue number from branch name (patterns: `feat/issue-42-*`, `fix/42-*`, `*/issue-42`, `#42`)
3. If issue number found:
   - Find Glacier card linked to that GitHub issue via `Glacier:list_cards` + `Glacier:get_card_github_status`
   - If card is in **Backlog** or **Ready** → move to **In Progress**
   - If already In Progress or later → do nothing (don't regress)
   - Check WIP limit before moving; warn if at limit
4. Report: `Glacier: moved "<card title>" → In Progress`
5. If no issue number or no matching card → silent skip

### `CwdChanged`
Fires when the working directory changes (e.g. switching worktrees).

1. Re-resolve column IDs for the new project context (column cache is per-session)
2. No board action — just cache refresh

### `PostCompact`
Fires after conversation compaction.

1. Re-resolve column IDs (cache may have been compacted out)
2. No board action

## Manual triggers (via `/glacier-sync` command)

Still available for on-demand operations that hooks don't cover:
- **Status**: board overview (cards per column, WIP limits, blockers)
- **PR merged → Done**: scan recent merges, move matching cards (gh doesn't fire a local hook for this)
- **TODO scanning**: scan `// TODO(glacier):` comments in branch diff, create cards
- **Issue linking**: link a GitHub issue to an existing card

## PR lifecycle (no local hook available)

`gh pr create` and PR merges happen outside Claude Code, so hooks can't catch them directly. Two options:

1. **The `implement` agent calls this skill explicitly** after `gh pr create` succeeds → move card to **In Review**
2. **User runs `/glacier-sync`** after merging → move card to **Done**

If the Glacier MCP ever exposes a webhook relay, this could also become fully automatic.

## MCP call pattern

EVERY Glacier MCP call must include `workspace_id` from `GLACIER_WORKSPACE_ID` env var (OAuth tokens are user-scoped, not workspace-scoped).

Example: `Glacier:list_columns(project_id: $GLACIER_PROJECT_ID, workspace_id: $GLACIER_WORKSPACE_ID)`

## Column resolution (cached per-session)

1. Call `Glacier:list_columns` once per session (or after `CwdChanged` / `PostCompact`)
2. Match names case-insensitively: "Backlog", "Ready", "In Progress", "In Review", "Done"
3. Cache the mapping — don't re-fetch on every trigger

## Card matching strategy

In order of reliability:
1. **GitHub issue link** — `Glacier:get_card_github_status` check for issue URL match
2. **Issue number in branch name** — match `feat/issue-42-*` to cards with issue #42 linked
3. **Title fuzzy match** — last resort, ask for confirmation

If no match → silent skip. Do not create cards automatically from branch creation.

## MCP tools used

All require `workspace_id` from env:

- `Glacier:list_columns` — column IDs and WIP status
- `Glacier:list_cards` — find cards by project
- `Glacier:get_card_github_status` — verify GitHub issue/PR links
- `Glacier:update_card` — move between columns
- `Glacier:create_card` — from TODOs (manual only)
- `Glacier:link_card_to_github` — link issues (manual only)

## Rules

- Always pass `workspace_id` to every MCP call. Never omit.
- Never move a card without a confident match. Ambiguous → skip silently on auto-triggers, ask on manual.
- Never regress a card (Done → In Review, etc.)
- Respect WIP limits — warn before moving into an at-limit column
- Report concisely: card title + action + column. One line.
- NEVER block the parent workflow. If anything fails, warn and continue.
