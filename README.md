# BlackBear Studio — Claude Code Configuration

> Shared Claude Code configuration for Black Bear Studio team members

## What is this repo?

This repository contains the shared Claude Code configuration that enforces Black Bear Studio's building principles, workflow standards, and tech stack conventions across all projects.

When you install this configuration, Claude Code will automatically:
- Follow our TDD workflow (Red → Green → Refactor)
- Enforce branching rules (never commit to `main`)
- Use conventional commits
- Apply our code standards (TypeScript strict, React hooks, Drizzle ORM, etc.)
- Tune effort per agent/skill for cost discipline (Claude Code 2.1.94+ effort levels)
- Run quality checks before task completion
- Auto-invoke safety skills (db-migration, secret-scan, stripe-integration) on relevant file changes

## What's Included

- **CLAUDE.md** — Main workflow rules, code standards, effort-level conventions
- **skills/** — Auto-invoked skills
  - `code-review` — Pre-PR quality review (auto at workflow step 9)
  - `test-gen` — TDD failing-test generation (auto at workflow step 4, logic issues only)
  - `db-migration` — Enforces DB safety rules on schema file changes
  - `stripe-integration` — Webhook verification, idempotency, test vs live mode checks
  - `secret-scan` — Pre-commit leak detection (Stripe keys, .env, tokens)
  - `deploy-checklist` — Pre-deployment verification for Vercel/Next.js
  - `glacier-sync` — Hook-based Glacier board sync (optional, env-gated)
- **agents/** — Specialized agents with scoped effort and turn limits
  - `architect` (Opus, xhigh) — System design and cross-cutting changes
  - `explorer` (Haiku, low) — Fast codebase exploration
  - `implement` (Sonnet, high) — Main TDD implementation agent (silent; use `/implement-v` for narrated demo runs)
  - `pr-prep` (Haiku, low) — PR description generation
- **commands/** — Workflow slash commands
  - `/implement` — Full TDD workflow, silent (card/issue/free-form modes)
  - `/implement-v` — Same workflow, narrated inline in the main terminal (for demos and live presentations)
  - `/init` — Session initialization
  - `/cost-check` — Token usage, rate limits, optimizations
  - `/skip-tests` — Bypass tests for ui/config issues
  - `/glacier` — Manual Glacier operations (alias: `/glacier-sync` kept for back-compat)

## Glacier Sync (opt-in, hooks-based)

The `glacier-sync` skill bridges GitHub workflow to [Glacier](https://getglacier.ai) boards via MCP — now via Claude Code hooks for reliability.

### Enable for a project

Add three variables to your `.env.local` (already gitignored in Next.js projects):

```
GLACIER_ENABLED=true
GLACIER_WORKSPACE_ID=<uuid from Project Settings>
GLACIER_PROJECT_ID=<uuid from Project Settings>
```

To find your IDs: open Glacier → Project Settings → copy the workspace ID and project ID.

The MCP server URL (`https://www.getglacier.ai/api/mcp`) is hardcoded. Column IDs are resolved dynamically at runtime via `list_columns`, so the board can be restructured without config changes.

### Auto-triggers (via Claude Code hooks)

| Hook | What fires it | Glacier transition |
|------|---------------|-------------------|
| `FileChanged` on `.git/HEAD` | Branch creation or switch | Ready / Backlog → **In Progress** |
| `CwdChanged` | Working directory change | Refresh column cache |
| `PostCompact` | Conversation compaction | Refresh column cache |

Card matching uses GitHub issue links first, then issue number in branch name, then title fuzzy match.

### Not covered by local hooks

PR open and PR merge happen outside Claude Code, so they're handled two ways:
- `implement` agent explicitly calls the skill after `gh pr create` succeeds (→ In Review)
- User runs `/glacier` after merging (→ Done)

### Manual capabilities via `/glacier`

- **Board status** — Cards per column, WIP limit status, blockers
- **PR sync** — Match recent merged PRs to cards, move to Done
- **TODO scanning** — Creates cards from `// TODO(glacier):` comments in branch diff
- **Issue linking** — Links GitHub issues to Glacier cards

`/glacier-sync` is kept as a deprecated alias that forwards to the same workflow.

### Disable for a project

Remove the env vars or set `GLACIER_ENABLED=false`. The skill skips silently when env vars are missing — it never blocks the workflow.

## Demo Mode

The `implement` agent runs the full TDD cycle (explore → plan → test-gen → implement → review → PR) silently by default — fine for solo work, hard to explain in front of an audience.

For demos, use `/implement-v` — it runs the same workflow inline in the main terminal with CLI-runner-style narration: each step gets a phased banner with a status suffix, Glacier card transitions are announced inline, and the run closes with a summary card.

### Run a demo

```
/implement-v card <id>
```

That's it — no flag, no env var. The command always narrates. There used to be a `[verbose]` flag and a `VERBOSE=true` env var on `/implement`; both have been removed. Two tools, two distinct uses, no overlap.

### What you'll see

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

─── 3/10 · Plan ──────────────────────────────────────────  DONE
   5 files to add/modify, plan approved

━━━ BUILD ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
─── 4-7/10 · Test cycle ──────────────────────────────────  SKIP
   reason: ui classification — no failing tests required

─── 6/10 · Implement ─────────────────────────────────────  DONE
─── 8/10 · Refactor ──────────────────────────────────────  DONE

━━━ SHIP ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
─── 9/10 · Review ────────────────────────────────────────  DONE
─── 10/10 · Report ───────────────────────────────────────  DONE
   ↳ 🧊 Glacier: "Build accounts list and create flow" → In Review ✓

━━━ DONE ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
   Card     CRM-5 — Build accounts list and create flow
   Branch   feat/crm-5-accounts-list (4 files, +127 -0)
   PR       #18 — opened
   Glacier  In Review
   Time     6m 24s
```

The `(pending)` → `✓` two-line beat on Glacier transitions is intentional: audience hears the announcement, looks at the Glacier board, watches the card move, sees the confirmation. The 🧊 emoji is the only emoji in the output — it appears twice per run, on the two board transitions, and nowhere else.

### Behaviour notes

- **Glacier narration is opt-in twice** — needs both `/implement-v` AND a working Glacier setup (env vars + linked card). Missing either → silent skip, no error noise for the audience.
- **Hook-based skills stay quiet** — `secret-scan`, `db-migration`, `stripe-integration` don't print banners. They fire on file events, not workflow steps.
- **Skipped steps collapse** — `ui` and `config` work shows one combined `SKIP` banner for steps 4–7 instead of four separate skip lines.
- **Silent failure for missing env** — Glacier API failures print one line and continue. Never block the PR. Never paste stack traces in front of clients.

### Recommended demo setup

1. Glacier board open in a browser tab visible to the audience (full-screen)
2. Terminal in another window
3. Source `.env.local` into the shell so the agent's subprocesses inherit `GLACIER_*`: `set -a; source .env.local; set +a` (Claude Code subprocesses don't auto-load `.env.local` — it's a Next.js convention, not a shell one)
4. Pick a small linked card and run `/implement-v card <id>` against it

## Installation

### Prerequisites
- Claude Code CLI installed ([installation guide](https://docs.claude.com))
- Git configured with your credentials

### Setup

1. **Backup your existing config (if any)**
   ```bash
   mv ~/.claude ~/.claude.backup
   ```

2. **Clone this repo to your home directory**
   ```bash
   git clone git@github.com:blackbearhq/claude-config.git ~/.claude
   # Or HTTPS:
   # git clone https://github.com/blackbearhq/claude-config.git ~/.claude
   ```

3. **Verify installation**
   ```bash
   ls ~/.claude/CLAUDE.md
   ```

4. **Start Claude Code in any project**
   ```bash
   cd /path/to/your-project
   claude
   ```

## Updating the Configuration

```bash
cd ~/.claude
git pull origin main
```

## Contributing Updates

1. Branch: `cd ~/.claude && git checkout -b feat/your-change`
2. Edit `CLAUDE.md` or add/modify files under `agents/`, `skills/`, `commands/`
3. Commit and push
4. Open a PR for team review

## Configuration Files

- `CLAUDE.md` — Main configuration with workflow rules and standards
- `README.md` — This file
- `.gitignore` — Excludes personal/machine-specific files
- `plans/` — Personal plan files (not tracked in git)

---

## Black Bear Studio — Building Principles

> AI-native products grounded in proven engineering practices

### Core Philosophy

Fast experimentation requires serious craft. We combine Agile mindset, eXtreme Programming principles, and Lean Startup methodology with AI-native development to move fast without breaking things that matter.

### Development Principles

**Quality First**
- Clean code and refactoring enable fast iteration
- Test-driven development for critical paths
- Technical debt is a strategic choice, not default state
- AI-assisted code generation with human review

**Ship and Learn**
- Working software over comprehensive plans
- Build-Measure-Learn cycles with real users
- MVPs validated through alpha programs
- Data-driven decisions: pivot or persevere

**Sustainable Pace**
- Continuous integration with frequent commits
- Small changes over big bang releases
- No burnout culture
- Long-term thinking over short-term heroics

**Privacy and Trust**
- Privacy by design: data minimization from day one
- User data ownership: export, deletion, transparency
- Security scales with user count
- Transparent communication when things break

### Tech Stack

**Core:** TypeScript, Next.js, React, Tailwind, Vercel
**Preferred (new projects):** Neon, Clerk, Drizzle ORM, Stripe
**Also in use:** Supabase (Auth + Postgres), pgvector

### Workflow

**Branching:** Never commit to `main` directly. Use `feat/`, `fix/`, `chore/`, `docs/` prefixes
**Testing:** TDD for business logic. Red → Green → Refactor
**Commits:** Conventional commits (`feat:`, `fix:`, `chore:`, `docs:`)
**Code Review:** Required before merge

### Quality Gates

**Before Alpha:**
- Core functionality reliable
- Basic test coverage on critical paths
- Privacy policy implemented
- Feedback mechanisms active

**Before Production:**
- Comprehensive test coverage
- Security audit completed
- Performance benchmarks met
- Monitoring and alerting configured

---

**Builder:** [Black Bear Studio](https://blackbear.so)
