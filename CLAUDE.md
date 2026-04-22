# BlackBear Studio — Global Claude Code Config

## Identity
- Builder: BlackBear Studio (blackbear.so)
- Core Stack: TypeScript, Next.js, React, Tailwind, Vercel
- UI Components: shadcn/ui (default for all new web projects)
- **Preferred stack (new projects):** Neon (Postgres), Clerk (auth), Drizzle ORM, Stripe
- **Also in use (existing projects):** Supabase (Postgres + Auth), pgvector
- When starting a new project, default to the preferred stack unless there's a specific reason not to
- When working on an existing project, follow the conventions already established in that repo

## Repository Context
- GitHub org: `blackbearhq` — ALWAYS use this as the default owner for all repos
- When running `gh` CLI commands, always use `--repo blackbearhq/<repo-name>`
- Never fall back to the authenticated user's personal account for repo resolution
- Studio repos: glacier, claude-config, skillsframe, plato, blackbear-site

## Workflow Rules
- ALWAYS run `npm run check` before considering a task complete
- ALWAYS run tests after modifying business logic
- Use Prettier formatting (auto-configured via .prettierrc)
- Commit messages: conventional commits (feat:, fix:, chore:, docs:)
- Never commit .env files. Use .env.example for documentation

## Git Attribution Rules
- NEVER add "Co-Authored-By: Claude" or any AI co-author trailer to commits
- NEVER set git user.name or user.email to Claude or any AI identity
- NEVER add Claude as a contributor, reviewer, or assignee on GitHub PRs/issues
- Git commits must always use the human developer's identity only
- PR descriptions may reference AI assistance but never attribute authorship

## Database Safety Rules

Schema changes can break other developers, running services, and production users. These rules apply to **any** relational database (PostgreSQL, MySQL, SQLite, etc.) and **any** hosting provider (Supabase, Neon, PlanetScale, Railway, local, etc.).

For the full workflow and command reference, see the `db-migration` skill — it auto-activates on schema file changes.

### Core Principle
NEVER alter a shared or production database schema without explicit approval. When in doubt, **generate a migration file** — don't push directly.

### Before Any Schema Change
1. **Identify the target.** Check which `DATABASE_URL` (or equivalent) is active in `.env.local`. Ask: "Is this a local DB, a shared dev DB, or production?"
2. **If shared or production → STOP.** Do not run schema-altering commands. Generate a migration file instead and present it for review.
3. **If local → proceed.** Direct push is acceptable on a developer's own local database.

### Dangerous Commands (require explicit approval)
These commands alter the database schema directly. NEVER run them against a shared or production database without asking first:

- `npx drizzle-kit push` / `npm run db:push` (Drizzle)
- `npx prisma db push` / `npx prisma migrate deploy` (Prisma)
- Raw SQL: `ALTER TABLE`, `DROP TABLE`, `CREATE TYPE`, `ALTER TYPE`, `DROP COLUMN`, `RENAME`
- Any ORM CLI command that syncs schema to a live database

### Safe Commands
- `npx drizzle-kit generate` — generates migration SQL files locally
- `npx prisma migrate dev --create-only` — generates migration without applying
- `npx drizzle-kit studio` / `npx prisma studio` — read-only DB viewers
- `npm run check` — type checking only

### Multi-Developer Projects
- Prefer local database instances for development (Docker, Supabase CLI `supabase start`, local Postgres)
- If using a shared cloud DB, coordinate schema changes — never push unannounced
- Enum and type changes are especially dangerous: they break instantly for anyone running old code

## UI Component Workflow (shadcn/ui)
- Use shadcn/ui as the default component library for all web projects
- Initialize with `npx shadcn@latest init` when setting up new projects
- Add components on demand: `npx shadcn@latest add <component>`
- Prefer shadcn/ui components over custom implementations for standard UI patterns (buttons, dialogs, forms, tables, etc.)
- Customize via Tailwind classes and CSS variables, not by forking component internals
- When a project already uses a different component library, follow the existing convention

## Workflow: Implementing Issues (TDD)

The full TDD workflow lives in the `implement` agent. Invoke via:
- `/implement card <card_id>` — starts from a Glacier card
- `/implement issue <issue_number>` — starts from a GitHub issue
- `/implement <free-form prompt>` — free-form description

The agent handles classification (logic/ui/config), branching, exploration, planning, TDD cycle, review, and PR prep. Board sync is automatic via the `glacier-sync` skill hooks — no manual step coordination.

### Classification rules
Before implementation, the agent classifies the work:
- `logic` — business logic, API, DB, auth, data transformation → full TDD required
- `ui` — components, layout, styling, copy → skip test-first, use visual review
- `config` — tooling, env, scripts, docs → skip test-first entirely

### Skip protocol
`[skip-tests]` in the prompt bypasses test steps. Only valid for `ui` and `config`. Refused for `logic`. Note skip reason in the PR description.

Red → Green → Refactor for all logic work.

## Branching Rules
- NEVER commit directly to main or master
- When starting work on a new feature, bug fix, or issue:
  1. Create a branch: `git checkout -b <type>/<short-description>`
  2. Branch naming: feat/, fix/, chore/, docs/ prefix
  3. Examples: feat/stripe-webhooks, fix/auth-redirect, chore/update-deps
- When asked to "commit changes" on main, STOP and ask if I want a branch first
- Only push to main via PR merge

## Git Guidelines
- NEVER run `git diff` without flags — it hangs on large diffs
- Always use `git diff --stat` for overview of changes
- Use `git diff --stat --cached` for staged changes
- If you need actual diff content, use `git diff -- <specific-file>` for targeted files only
- ALWAYS use `git mv` instead of `mv` when moving tracked files — preserves commit history

## Cost Discipline (Claude Code 2.1.94+)

Claude Code now has per-session effort tuning — use it instead of manually switching models.

- Default effort for Pro/Max on Opus 4.6 / Sonnet 4.6 is `high` (as of 2.1.117)
- Opus 4.7 supports `xhigh` effort — reserved for architecture and deep debugging
- Switch effort live with `/effort` (interactive slider) or `/effort <level>`
- Auto mode is default-on for Max + Opus 4.7 — let it route between effort levels automatically when appropriate

### Effort level conventions for this config

| Work type | Agent/skill | Effort |
|-----------|-------------|--------|
| Architecture, cross-cutting design | `architect` (Opus) | `xhigh` |
| Implementation | `implement` (Sonnet) | `high` |
| Code review | `code-review` (Sonnet) | `medium` |
| Test generation | `test-gen` (Haiku, escalate Sonnet) | `medium` |
| Exploration, search | `explorer` (Haiku) | `low` |
| PR prep, formatting | `pr-prep` (Haiku) | `low` |
| Deploy checklist, secret scan | skills (Haiku) | `low` |

### Session hygiene
- Use `/clear` between unrelated tasks to reset context
- Use `/cost` and `/usage` regularly to monitor spend and rate limits (see `/cost-check` command)
- Use `/compact` when context > 60% and the session is still going
- Sub-agent prompts stay short and focused (3-5 word descriptions)

## Code Standards
- TypeScript strict mode, no `any` types
- React: functional components + hooks only
- API responses: `{ success: boolean, data: T, error?: string }`
- Database: Drizzle ORM, never raw SQL in application code
- Auth (when used): Clerk (new projects) or Supabase Auth (existing)
- Error handling: try/catch with typed errors, never silent failures

## Communication Style
- Be direct. No corporate fluff
- Suggest alternatives proactively
- Flag cost/complexity tradeoffs explicitly
- If unsure, ask — don't guess

## Available Agents, Skills, and Commands

### Agents
- `architect` — System design, Opus, effort: xhigh. Use for new features, tradeoff analysis, cross-cutting structural changes.
- `explorer` — Codebase exploration, Haiku, effort: low. Read-only investigation.
- `implement` — TDD workflow entry point, Sonnet, effort: high. Supports card/issue/free-form modes.
- `pr-prep` — PR description generation, Haiku, effort: low.

### Skills (auto-invoked)
- `code-review` — Review at workflow step 9
- `test-gen` — Failing tests at workflow step 4 (logic only)
- `db-migration` — Auto-fires on schema file changes
- `stripe-integration` — Auto-fires on Stripe-related files
- `secret-scan` — Pre-commit leak detection
- `deploy-checklist` — Pre-deployment verification
- `glacier-sync` — Hook-based board sync (optional, requires env vars)

### Slash commands
- `/implement` — Invokes the `implement` agent
- `/pr-prep` — PR description and changelist
- `/skip-tests` — Bypass test steps for ui/config issues
- `/cost-check` — Token usage, rate limits, optimization suggestions
- `/init` — Session initialisation
- `/glacier-sync` — Manual Glacier operations (hooks handle automatic sync)
