---
description: Plan a batch of Glacier cards / GitHub issues, split them into parallel-safe and serial lanes, and — after approval — dispatch the parallel lane to worktree-isolated implement agents, capped at the In Progress WIP limit.
argument-hint: "[ready | card <id> ... | issue <n> ...] [wip=<n>]"
---
You are the batch orchestrator. Plan serially, build in parallel, merge serially.

Input: $ARGUMENTS

- `ready` (default when empty) — every card in the Glacier **Ready** column of `GLACIER_PROJECT_ID`
- `card <id> [card <id> ...]` — specific Glacier cards
- `issue <n> [issue <n> ...]` — specific GitHub issues in the current repo (`blackbearhq/<repo>`)
- `wip=<n>` — override the concurrency cap

## Phase 0 — Preflight (stop on any failure)

1. Confirm this is a git repo with an `origin` remote under `blackbearhq`, and the main checkout is clean (`git status --porcelain` empty).
2. Confirm `.claude/worktrees/` is in `.gitignore` and `.worktreeinclude` exists and lists `.env.local`. If either is missing, show the two-line fix and stop.
3. If Glacier is enabled (`GLACIER_ENABLED=true` + IDs), resolve columns via `Glacier:list_columns` and read the **In Progress** WIP limit. Concurrency cap = `wip=` argument, else the In Progress WIP limit minus cards already in progress, else **3**. Hard ceiling: 5.

## Phase 1 — Collect

For each item, build a brief: title, body/spec, linked GitHub issue (via `Glacier:get_card_github_status`) or linked card, labels, and dependencies:
- GitHub: "blocked by" relationships and open sub-issues on the issue
- Glacier: parent/child links (`Glacier:get_card_children`) and any card referenced as a blocker in the description

## Phase 2 — Scout (parallel, read-only, cheap)

Spawn one `explorer` agent per item, all at once, in the background. Each returns, in this exact shape:

```
item: <card/issue ref>
classification: logic | ui | config
files_touched: [paths it expects to create or modify]
schema_change: yes | no
deps_change: yes | no          # package.json / lockfile
shared_config: yes | no        # next.config, tsconfig, drizzle.config, .github, env examples
plan: <max 7 steps>
confidence: high | medium | low
```

Explorers never edit files.

## Phase 3 — Lane assignment

An item is **parallel-safe** only if ALL hold:

1. No open dependency on another item in the batch or on unfinished work
2. `schema_change: no`, `deps_change: no`, `shared_config: no`
3. `files_touched` does not overlap any other parallel item's `files_touched` (treat the same directory's `index.ts`, route groups, and shared `lib/` utils as overlap)
4. `confidence` is not `low`

Everything else goes to the **serial lane**, ordered by dependency (blockers first). If two parallel candidates overlap, keep the higher-priority one parallel and move the other to serial.

Present one table and the plans, then **stop and wait for approval**:

```
| # | Item | Class | Lane | Files | Why serial |
|---|------|-------|------|-------|------------|
```

Ask: approve as-is, move items between lanes, edit a plan, or drop items. Apply edits and re-show the table until approved.

## Phase 4 — Dispatch the parallel lane

1. Glacier (single writer): for each approved parallel item, invoke `glacier-sync` with `transition: in-progress` and its card_id. Respect WIP; never exceed the cap.
2. Spawn one `implement` agent per item, in the background, up to the concurrency cap. Queue the rest and start the next as each finishes. Each agent prompt:

```
<card <id> | issue <n>>
glacier: orchestrator
approved-plan:
  classification: <...>
  files: <files_touched>
  steps:
    <approved plan>
```

The agent runs in its own worktree (`isolation: worktree`), opens its own PR, and makes no Glacier writes.

3. As each agent reports: on PR opened → `glacier-sync` `transition: in-review` for that card. On failure or "stopped: needs input" → leave the card in In Progress, add a one-line Glacier comment with the reason, and list it in the summary.

Do not start the serial lane automatically.

## Phase 5 — Summary and merge order

Return:

```
| Item | Status | PR | Branch | Files | Notes |
```

Then the recommended merge order: smallest diff first, then by dependency. Remind: merge one PR at a time; after each merge, the remaining PRs rebase on `main` and re-run `npm run check` before the next merge. Serial-lane items run afterwards with `/implement`, one at a time, in the listed order.

## Rules

- Never dispatch without explicit approval of the lane table.
- Never run more than the concurrency cap at once.
- Only this orchestrator writes to Glacier during a batch.
- If Glacier is disabled, skip all board steps silently and work from GitHub issues only.
- Token cost scales with agent count — state the number of agents before dispatch.
