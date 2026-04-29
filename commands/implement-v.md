---
name: implement-v
description: Verbose/foreground variant of /implement. Runs the TDD workflow inline in the main thread instead of delegating to the implement agent, so step banners and Glacier beats stream into the terminal live. Intended for demos and any session where the audience needs to see orchestration happen rather than receive a post-hoc summary. Same entry modes as /implement (card / issue / free-form).
---
You are running the implement workflow **inline in the main thread**. Do NOT delegate this whole task to the `implement` agent — that's what the regular `/implement` command does, and its narration ends up in a sidechain task file invisible to the user.

Verbose mode is implicitly ON for this command, regardless of the `[verbose]` flag or `VERBOSE` env var. Print every banner.

## Why this command exists

For demos, the user needs to see each step land in the terminal as it happens — boxed dividers, delegation callouts, Glacier transitions. The standard `/implement` command runs as an agent (sidechain), which means narration is written to a task output file rather than the main terminal. This command runs the same workflow but inline, so the audience sees the orchestration unfold in real time.

You may still delegate the **heavy individual steps** (explore, test-gen, code-review) to their respective subagents — those are bounded, single-purpose tasks where backgrounding is acceptable. What stays inline is the **orchestration**: classification, branching, planning, Glacier transitions, step banners, outcome lines.

## Argument parsing

Parse `$ARGUMENTS` exactly like the implement agent does:

- `card <card_id>` → Glacier card mode (requires `GLACIER_ENABLED=true`, `GLACIER_WORKSPACE_ID`, `GLACIER_PROJECT_ID` in env)
- `issue <number>` → GitHub issue mode
- Anything else → free-form prompt mode

If `[verbose]` appears in the arguments, strip it before parsing — verbose is already implicit.
If `[skip-tests]` appears, honour it the same way `/implement` does.

If the env check fails for card mode, print a single concise line telling the user which vars are missing and stop. Do not paste shell snippets — they're in the README.

## Workflow

Follow the same 11-step TDD workflow as `agents/implement.md` (steps 0–10). For each step:

1. Print the banner first:
   ```
   ─── <n>/10 · <Step name> ─────────────────────────────
   ```
2. Print the delegation callout if applicable (see table below)
3. Execute the step
4. Print a brief one-line outcome before the next banner

### Step table

| Step | Name | Delegation callout | Inline or delegated |
|------|------|---------------------|---------------------|
| 0 | Classify | (none) | inline |
| 1 | Branch | `↳ Glacier: "<title>" → In Progress (pending)` then `✓` after the move | inline |
| 2 | Explore | `→ delegating to explorer agent` | **delegated** (Task) |
| 3 | Plan | (none) | inline — pause for approval |
| 4 | Test first | `→ invoking test-gen skill` (logic only) | **delegated** (Task, skill) |
| 5 | Verify red | (none) | inline |
| 6 | Implement | (none) | inline |
| 7 | Verify green | (none) | inline |
| 8 | Refactor | (none) | inline |
| 9 | Review | `→ invoking code-review skill` | **delegated** (Task, skill) |
| 10 | Report | `↳ Glacier: "<title>" → In Review ✓` after `gh pr create` | inline |

### Glacier transitions in verbose mode

Two transitions are narrated here:

1. **Step 1 — Backlog → In Progress** (card mode only)
   - Before the Glacier MCP call: `↳ Glacier: "<card title>" → In Progress (pending)`
   - After the move succeeds: `↳ Glacier: "<card title>" → In Progress ✓`
   - If no card linked: skip both lines silently. Don't fake it.

2. **Step 10 — In Progress → In Review**
   - After `gh pr create` succeeds, call `glacier-sync` explicitly to move the card
   - Print: `↳ Glacier: "<card title>" → In Review ✓` after the move
   - On failure: `↳ Glacier: move failed (continuing)`. Never block the PR. Never paste stack traces in front of clients.

The Done transition (In Review → Done) happens after merge — outside this command's scope.

### Skipped steps

- For `ui` / `config` work: still print banners 4–7 with `skipped (ui)` or `skipped (config)` instead of executing. Keeps the audience oriented.
- For `[skip-tests]` requests: print banner with `skipped ([skip-tests] flag)`.

### Quiet zone

Hook-based skills (`secret-scan`, `db-migration`, `stripe-integration`) do NOT print banners — they fire on file events, not workflow steps, and announcing them would clutter the demo. `glacier-sync` is the exception, but only for the two transitions above.

## Pause for plan approval

At step 3 (Plan), present the plan and **wait for the user to confirm** before continuing. This is the natural demo beat — the audience reads the plan, the presenter confirms, the build resumes. Do not auto-proceed.

## Failure modes

- Glacier disabled or env vars missing → skip all `↳ Glacier:` lines silently. Don't print "glacier disabled" — it's noise.
- Card not linked to issue → skip silently for the same reason.
- Glacier API error → one line, continue. No stack traces.

## When NOT to use this command

- Long sessions where you're not watching — use `/implement` instead. The agent variant doesn't crowd the main context window.
- Hot-fix work where speed matters more than narration.
- Anything `[skip-tests]` on logic — refuse same as `/implement`.

Red → Green → Refactor.
