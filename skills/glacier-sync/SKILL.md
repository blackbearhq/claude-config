---
name: glacier-sync
description: Glacier board sync, called explicitly with a transition (in-progress, in-review, done) by the implement agent, /implement-v, /triage, or /glacier. Only activates when GLACIER_ENABLED=true, GLACIER_WORKSPACE_ID, and GLACIER_PROJECT_ID are set. Skips silently if not configured.
model: haiku
effort: low
tools: Bash, Read
---
# Glacier Sync (explicit transitions)

Keeps the Glacier board in sync with repo activity. Callers invoke it at known workflow points with a named transition. No hooks.

**Fully optional.** If env vars are missing, skip silently. Never block the parent workflow.

## Why explicit, not hooks

The previous version relied on a `FileChanged` hook on `.git/HEAD`. It was unreliable and cannot work with parallel runs:

- `FileChanged` matchers take literal filenames, and `if` only applies to tool events, so the shell-test condition was ignored.
- Skill hooks register only after the skill is invoked, so the first branch creation in a session was missed.
- In a git worktree `.git` is a file and HEAD lives under the main repo's `.git/worktrees/<name>/`, so the hook never fires for worktree-isolated agents.

Explicit calls at fixed workflow steps are deterministic and work the same in the main checkout, in worktrees, and in parallel batches.

## Activation conditions

All must be true, otherwise silent skip:
1. `GLACIER_ENABLED=true`
2. `GLACIER_WORKSPACE_ID` is set
3. `GLACIER_PROJECT_ID` is set

## Configuration

In `.env.local` (gitignored — list it in `.worktreeinclude` so worktrees get a copy):

```
GLACIER_ENABLED=true
GLACIER_WORKSPACE_ID=<uuid from Project Settings>
GLACIER_PROJECT_ID=<uuid from Project Settings>
```

MCP server URL is hardcoded: `https://www.getglacier.ai/api/mcp`

Column IDs resolve at runtime via `Glacier:list_columns` — no stored IDs, board can be restructured freely.

## Invocation contract

Callers pass:

| Input | Required | Values |
|-------|----------|--------|
| `transition` | yes | `in-progress`, `in-review`, `done` |
| `card_id` | preferred | Glacier card UUID (skip matching when provided) |
| `issue` | fallback | GitHub issue number, used to find the card |
| `narrate` | no | `true` when called from `/implement-v` (demo formatting) |

### Who calls what

| Transition | Caller | When |
|------------|--------|------|
| `in-progress` | `implement` agent / `/implement-v` | Right after the feature branch is created (step 1) |
| `in-progress` | `/triage` | For every card in a batch, before agents are dispatched |
| `in-review` | `implement` agent / `/implement-v` / `/triage` | After `gh pr create` succeeds |
| `done` | `/glacier` (PR sync) | After merge — merges happen outside Claude Code |

**Single writer rule for batches:** when `/triage` dispatches parallel agents, it passes `glacier: orchestrator` to each agent, and the agents make no Glacier writes. The orchestrator performs all transitions. This avoids concurrent writes and WIP-limit races.

## Transition rules

1. Resolve columns (cached per session): match names case-insensitively — "Backlog", "Ready", "In Progress", "In Review", "Done".
2. Resolve the card: `card_id` if given, else match via `issue` (see Card matching).
3. Never regress a card. Allowed moves:
   - `in-progress`: from Backlog or Ready
   - `in-review`: from Backlog, Ready, or In Progress
   - `done`: from In Review (or In Progress if the PR is already merged)
   - Card already in the target column or later → no-op, print nothing.
4. Check the WIP limit of the target column. If at limit, warn in one line and still move only if the caller said `force_wip: true`; otherwise skip and report.
5. Move with `Glacier:update_card`.

## Card matching strategy

In order of reliability:
1. **Explicit `card_id`** from the caller
2. **GitHub issue link** — `Glacier:list_cards` + `Glacier:get_card_github_status`, match the issue URL/number
3. **Title fuzzy match** — manual `/glacier` runs only, ask for confirmation

No confident match → silent skip. Never create cards from a transition call.

## Output formatting

### Default
Single compact line, only on actual moves:

```
Glacier: "Stripe webhook retry logic" → In Progress
```

### `narrate: true` (called from `/implement-v`)

```
↳ 🧊 Glacier: "Stripe webhook retry logic" → In Progress ✓
↳ 🧊 Glacier: move failed (continuing) — <one-line reason>
```

`/implement-v` prints the `(pending)` line itself before calling this skill. Print nothing for no-ops, unmatched cards, or disabled Glacier — silence is correct.

## Manual operations (via `/glacier`)

- **Status**: board overview (cards per column, WIP limits, blockers)
- **PR merged → Done**: scan recent merges, move matching cards
- **TODO scanning**: scan `// TODO(glacier):` comments in branch diff, create cards
- **Issue linking**: link a GitHub issue to an existing card

## MCP call pattern

EVERY Glacier MCP call must include `workspace_id` from `GLACIER_WORKSPACE_ID` (OAuth tokens are user-scoped, not workspace-scoped).

Example: `Glacier:list_columns(project_id: $GLACIER_PROJECT_ID, workspace_id: $GLACIER_WORKSPACE_ID)`

## MCP tools used

- `Glacier:list_columns` — column IDs and WIP status
- `Glacier:list_cards` — find cards by project
- `Glacier:get_card` — card details
- `Glacier:get_card_github_status` — verify GitHub issue/PR links
- `Glacier:update_card` — move between columns
- `Glacier:create_card` — from TODOs (manual only)
- `Glacier:link_card_to_github` — link issues (manual only)

## Rules

- Always pass `workspace_id` to every MCP call.
- Never move a card without a confident match.
- Never regress a card.
- Respect WIP limits.
- One line per move. No stack traces.
- NEVER block the parent workflow. If anything fails, warn in one line and continue.
