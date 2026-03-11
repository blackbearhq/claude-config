# BlackBear Studio — Claude Code Configuration

> Shared Claude Code configuration for Black Bear Studio team members

## What is this repo?

This repository contains the shared Claude Code configuration that enforces Black Bear Studio's building principles, workflow standards, and tech stack conventions across all projects.

When you install this configuration, Claude Code will automatically:
- Follow our TDD workflow (Red → Green → Refactor)
- Enforce branching rules (never commit to `main`)
- Use conventional commits
- Apply our code standards (TypeScript strict, React hooks, Drizzle ORM, etc.)
- Optimize for cost (Sonnet for implementation, Haiku for exploration)
- Run quality checks before task completion

## What's Included

This configuration includes:

- **CLAUDE.md** - Main workflow rules and code standards
- **skills/** - Custom Black Bear Studio skills (auto-invoked or via slash commands)
  - `bbs-brand` - Brand guidelines for client-facing materials
  - `deploy-checklist` - Pre-deployment verification for Vercel/Next.js
  - `code-review` - Code quality and security reviews (auto-invoked at review step)
  - `test-gen` - TDD test generation, write failing tests first (auto-invoked at test step)
  - `glacier-sync` - Sync repo activity with Glacier board via MCP (auto-invoked on branch creation, PR open, PR merge; opt-in via env vars)
- **agents/** - Custom agent definitions for specialized tasks
  - `explorer` - Fast codebase exploration
  - `pr-prep` - PR description generation
  - `architect` - System design and architecture planning
- **commands/** - Workflow slash commands
  - `implement` - Full TDD implementation workflow
  - `init` - Session initialization
  - `cost-check` - Token usage monitoring
  - `skip-tests` - Bypass test steps for ui/config issues
  - `glacier-sync` - Sync current repo with Glacier board

All agents and commands follow Black Bear Studio's engineering practices.

## Glacier Sync (opt-in)

The `glacier-sync` skill bridges GitHub workflow to [Glacier](https://getglacier.ai) boards via MCP. It only activates in projects where `GLACIER_ENABLED=true` and `GLACIER_PROJECT_ID` are set in the environment.

### Enable for a project

Add two variables to your `.env.local` (already gitignored in Next.js projects):

```
GLACIER_ENABLED=true
GLACIER_PROJECT_ID=<uuid from Project Settings>
```

To find your project ID: open Glacier → Project Settings → copy the project ID.

The MCP server URL (`https://www.getglacier.ai/api/mcp`) is hardcoded in the skill — no configuration needed.

Column IDs are resolved dynamically at runtime via `list_columns`, so the board can be restructured without updating any config.

### What it does

**Auto-triggers during `/implement` workflow:**

| Trigger | Workflow step | Glacier transition |
|---------|---------------|-------------------|
| Branch created | Step 1b | Ready / Backlog → **In Progress** |
| PR opened | Step 9b | In Progress → **In Review** |
| PR merged | Post-merge | In Review → **Done** |

Card matching uses GitHub issue links first, then issue number in branch name, then title fuzzy match (with confirmation).

**Manual capabilities via `/glacier-sync`:**

- **Board status** — Cards per column, WIP limit status, blockers
- **PR sync** — Match recent merged PRs to cards, move to Done
- **TODO scanning** — Creates cards from `// TODO(glacier):` comments in branch diff (`git diff --name-only main...HEAD`)
- **Issue linking** — Links GitHub issues to Glacier cards after creation

### Disable for a project

Remove the env vars or set `GLACIER_ENABLED=false`. The skill skips silently when env vars are missing — it never blocks the workflow.

## Installation

### Prerequisites
- Claude Code CLI installed ([installation guide](https://docs.claude.com))
- Git configured with your credentials

### Setup

1. **Backup your existing config (if any)**
   ```bash
   # If you have an existing .claude directory
   mv ~/.claude ~/.claude.backup
   ```

2. **Clone this repo to your home directory**
   ```bash
   git clone git@github.com:blackbearhq/claude-config.git ~/.claude
   # Or use HTTPS:
   # git clone https://github.com/blackbearhq/claude-config.git ~/.claude
   ```

3. **Verify installation**
   ```bash
   ls ~/.claude/CLAUDE.md
   # Should show the main configuration file
   ```

4. **Start Claude Code in any project**
   ```bash
   cd /path/to/your-project
   claude
   ```

   Claude will now use the Black Bear Studio configuration automatically.

## Updating the Configuration

To get the latest team updates:

```bash
cd ~/.claude
git pull origin main
```

## Contributing Updates

If you want to propose changes to the shared configuration:

1. Create a branch in this repo
   ```bash
   cd ~/.claude
   git checkout -b feat/improve-workflow
   ```

2. Make your changes to `CLAUDE.md` or other config files

3. Commit and push
   ```bash
   git add .
   git commit -m "feat: add Go language support"
   git push origin feat/improve-workflow
   ```

4. Open a PR and discuss with the team

## Configuration Files

- `CLAUDE.md` - Main configuration with workflow rules and standards
- `README.md` - This file
- `.gitignore` - Excludes personal/machine-specific files
- `plans/` - Personal plan files (not tracked in git)

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
**Common:** PostgreSQL + pgvector, Drizzle ORM, Stripe, Supabase Auth

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
