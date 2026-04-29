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

---

## Presentation protocol

This command's output is the demo. Treat formatting as part of the contract, not a stylistic choice. The audience reads the terminal as a CLI runner — `cargo`, `pytest`, `vercel deploy` — so the output should feel like one of those: monospace, status-suffixed, phased, calm.

### Phase headers

Group the 11 steps into four phases. Print the phase header once when entering it, before the first step banner of that phase:

```
━━━ SETUP ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Phases:

| Phase | Steps | What happens |
|-------|-------|--------------|
| `SETUP` | 0–1 | Classify, Branch |
| `DESIGN` | 2–3 | Explore, Plan (closes with the approval gate) |
| `BUILD` | 4–8 | Test cycle (logic only), Implement, Refactor |
| `SHIP` | 9–10 | Review, Report |

The phase header line is exactly 60 characters wide: `━━━ <NAME> ` followed by `━` characters padding to column 60. No emoji. No color.

### Step banners

Each step prints exactly one banner line, formatted to fixed 60-char width with a right-aligned status suffix:

```
─── 0/10 · Classify ──────────────────────────────────────  ...
```

Format breakdown:
- `─── ` (4 chars)
- `<step>/10 · <Step name> ` — variable width
- `─` characters padding to column 56
- 2 spaces
- Status keyword right-aligned at column 60

Status keywords:

| Keyword | When to use |
|---------|-------------|
| `...` | Step is in flight — print this when the banner first appears |
| `DONE` | Step finished successfully — overwrite or reprint the banner with this suffix at completion |
| `SKIP` | Step skipped — append reason in parens on the next line if helpful, or inline if it fits |
| `FAIL` | Step failed — workflow should halt and the user should be told what to do |

The agent prints the banner with `...` first, runs the step, then **reprints the banner with the final status** before the outcome line. This is the visual rhythm of `pytest -v` — line appears, line resolves, next line appears.

If reprinting causes terminal noise (some clients render this as duplication rather than an update), it is acceptable to print the `...` banner, then a blank line of work, then the resolved banner — but never let a step end without a status suffix.

### Outcome lines

After the resolved banner, print one or two lines describing what happened. Keep them short and concrete:

```
─── 1/10 · Branch ────────────────────────────────────────  DONE
   branch: feat/crm-5-accounts-list
   ↳ 🧊 Glacier: "Build accounts list and create flow" → In Progress ✓
```

Indent outcome lines with three spaces. This visually subordinates them to the banner and keeps the left edge clean.

### Skip collapse rule

When two or more adjacent steps will skip for the same reason, print **one combined banner** instead of separate skip lines. This applies most often to UI/config classification, which skips steps 4–7 (test cycle).

```
─── 4-7/10 · Test cycle ──────────────────────────────────  SKIP
   reason: ui classification — no failing tests required
```

Rules for collapsing:
- Step numbers must be contiguous (4-7 yes, 4 + 6 no)
- Skip reason must be identical for all collapsed steps
- The combined banner uses a range (`4-7/10`) in the step position
- Only one outcome line for the whole block

If a step in the middle of a would-be-skip block actually runs (rare but possible), do not collapse. Print each banner individually.

### Glacier transitions

Glacier is the only place in this command where an emoji is allowed: 🧊. Use it on transition lines only, twice per run maximum:

```
↳ 🧊 Glacier: "<card title>" → In Progress (pending)
↳ 🧊 Glacier: "<card title>" → In Progress ✓
↳ 🧊 Glacier: "<card title>" → In Review ✓
```

The 🧊 anchors the audience's eye to the line so the board move and the terminal line are visually paired. Do not use emoji elsewhere — not on status suffixes, not on phase headers, not on outcome lines. The single brand glyph carries more weight than a parade.

### Final summary block

After step 10 (Report) resolves, print a summary card. This is the screenshot moment:

```
━━━ DONE ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Card     CRM-5 — Build accounts list and create flow
   Branch   feat/crm-5-accounts-list (4 files, +127 -0)
   PR       #18 — opened
   Glacier  In Review
   Time     6m 24s
```

Format:
- Header line: `━━━ DONE ` padded to 60 chars with `━`
- Five rows, each indented 3 spaces
- Label column is 8 chars wide (left-aligned), then 1 space, then value
- Values are short — no full URLs, no descriptions

Rows:
- **Card** — card ID and title (or `—` if not card mode)
- **Branch** — branch name and `(N files, +X -Y)` from `git diff --shortstat main`
- **PR** — PR number and status (`opened`, `merged`, `—` if `[skip-tests]` and no PR)
- **Glacier** — final column the card landed in (`In Review`, or `—` if not card mode)
- **Time** — elapsed from step 0 banner to step 10 resolve, formatted `Xm Ys`

If a value is unavailable, show `—`. Never fake it.

### What stays quiet

Hook-based skills (`secret-scan`, `db-migration`, `stripe-integration`) do NOT print banners. They fire on file events, not workflow steps, and announcing them would clutter the demo. The audience sees the steps, not the safety nets. `glacier-sync` is the exception, but only for the two transitions narrated above.

### Width and color

- All banners and phase headers are exactly 60 characters wide.
- No ANSI color codes. They break in pipes and logs and look inconsistent across terminals.
- Whitespace, monospace alignment, and the single 🧊 are doing all the visual work.

---

## Workflow

Follow the same 11-step TDD workflow as `agents/implement.md` (steps 0–10). The presentation protocol above governs how each step is announced; the workflow itself is unchanged.

### Step table

| Step | Phase | Name | Delegation | Inline or delegated |
|------|-------|------|------------|---------------------|
| 0 | SETUP | Classify | (none) | inline |
| 1 | SETUP | Branch | Glacier transition (card mode) | inline |
| 2 | DESIGN | Explore | `→ delegating to explorer agent` | **delegated** (Task) |
| 3 | DESIGN | Plan | (none) | inline — pause for approval |
| 4 | BUILD | Test first | `→ invoking test-gen skill` (logic only) | **delegated** (Task, skill) |
| 5 | BUILD | Verify red | (none) | inline |
| 6 | BUILD | Implement | (none) | inline |
| 7 | BUILD | Verify green | (none) | inline |
| 8 | BUILD | Refactor | (none) | inline |
| 9 | SHIP | Review | `→ invoking code-review skill` | **delegated** (Task, skill) |
| 10 | SHIP | Report | Glacier transition after `gh pr create` | inline |

### Glacier transitions

Two transitions are narrated:

1. **Step 1 — Backlog → In Progress** (card mode only)
   - Before the Glacier MCP call: `↳ 🧊 Glacier: "<card title>" → In Progress (pending)`
   - After the move succeeds: `↳ 🧊 Glacier: "<card title>" → In Progress ✓`
   - If no card linked: skip both lines silently. Don't fake it.

2. **Step 10 — In Progress → In Review**
   - After `gh pr create` succeeds, call `glacier-sync` explicitly to move the card
   - Print: `↳ 🧊 Glacier: "<card title>" → In Review ✓` after the move
   - On failure: `↳ 🧊 Glacier: move failed (continuing)`. Never block the PR. Never paste stack traces in front of clients.

The Done transition (In Review → Done) happens after merge — outside this command's scope.

### Pause for plan approval

At step 3 (Plan), present the plan and **wait for the user to confirm** before continuing. This is the natural demo beat — the audience reads the plan, the presenter confirms, the build resumes. Do not auto-proceed.

The plan is the closing beat of the DESIGN phase. After approval, print the BUILD phase header before the next step banner.

### Failure modes

- Glacier disabled or env vars missing → skip all `↳ 🧊 Glacier:` lines silently. Don't print "glacier disabled" — it's noise.
- Card not linked to issue → skip silently for the same reason.
- Glacier API error → one line, continue. No stack traces.
- A step throws → use `FAIL` status on the banner, print the error in one line, halt.

### When NOT to use this command

- Long sessions where you're not watching — use `/implement` instead. The agent variant doesn't crowd the main context window.
- Hot-fix work where speed matters more than narration.
- Anything `[skip-tests]` on logic — refuse same as `/implement`.

---

## Reference output

A complete run on a UI card might look like this:

```
━━━ SETUP ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
─── 0/10 · Classify ──────────────────────────────────────  DONE
   ui — pages + dialog component, Server Action for DB write

─── 1/10 · Branch ────────────────────────────────────────  DONE
   branch: feat/crm-5-accounts-list
   ↳ 🧊 Glacier: "Build accounts list and create flow" → In Progress ✓

━━━ DESIGN ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
─── 2/10 · Explore ───────────────────────────────────────  DONE
   → delegating to explorer agent
   findings: Drizzle accounts table, Clerk currentUser pattern, no existing dialog

─── 3/10 · Plan ──────────────────────────────────────────  DONE
   5 files to add/modify, plan approved

━━━ BUILD ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
─── 4-7/10 · Test cycle ──────────────────────────────────  SKIP
   reason: ui classification — no failing tests required

─── 6/10 · Implement ─────────────────────────────────────  DONE
   wrote actions.ts, new-account-dialog.tsx, page.tsx, [id]/page.tsx

─── 8/10 · Refactor ──────────────────────────────────────  DONE
   no changes — first pass acceptable

━━━ SHIP ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
─── 9/10 · Review ────────────────────────────────────────  DONE
   → invoking code-review skill
   no critical issues

─── 10/10 · Report ───────────────────────────────────────  DONE
   gh pr create succeeded
   ↳ 🧊 Glacier: "Build accounts list and create flow" → In Review ✓

━━━ DONE ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Card     CRM-5 — Build accounts list and create flow
   Branch   feat/crm-5-accounts-list (4 files, +127 -0)
   PR       #18 — opened
   Glacier  In Review
   Time     6m 24s
```

Red → Green → Refactor.
