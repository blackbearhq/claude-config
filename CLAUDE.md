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

### Schema Change Workflow
1. Edit the schema file (e.g., `db/schema.ts`, `prisma/schema.prisma`)
2. Generate a migration file (never push directly)
3. Commit the migration file to the branch
4. Migration is reviewed in the PR
5. Applied to production only after merge and explicit approval
6. **Breaking changes** (enum changes, column renames, type changes) must be communicated to all developers before applying

### Multi-Developer Projects
When a database is shared between developers:
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
When asked to implement a feature, fix, or issue, follow this sequence:

0. **Classify**: Identify the issue type before doing anything else:
   - `logic` — business logic, API, DB, auth, data transformation → full TDD required
   - `ui` — components, layout, styling, copy → skip steps 4–7, use visual review instead
   - `config` — tooling, env, scripts, docs → skip steps 4–7 entirely
   State the classification out loud before proceeding.

1. **Branch**: Create a feature branch from main (never commit to main directly)
1b. **Board sync** _(optional)_: If `glacier-sync` skill is enabled and `GLACIER_ENABLED=true` + `GLACIER_WORKSPACE_ID` + `GLACIER_PROJECT_ID` are set in the environment, move the linked Glacier card to **In Progress** (marks start of cycle time). Always pass `workspace_id` to Glacier MCP calls. If glacier-sync is not available or env vars are missing, skip silently and continue — never fail the workflow.
2. **Explore**: Delegate to the `explorer` agent to find relevant files and patterns
3. **Plan**: Present a brief implementation plan (max 7 steps). Wait for my approval
4. **Test first** _(logic only)_: Invoke the `test-gen` skill to write failing tests that define the expected behavior
5. **Verify red** _(logic only)_: Run tests — confirm they FAIL. If they pass, the tests aren't testing anything new
6. **Implement**: Write the minimum code to make tests pass (main session, Sonnet)
7. **Verify green** _(logic only)_: Run `npm run test:run -- --testPathPattern=<changed-file>`. Confirm they PASS. Never run the full test suite during TDD cycles — full suite runs at PR prep stage only.
8. **Refactor**: Clean up while keeping tests green
9. **Review**: Invoke the `code-review` skill to review all changes
9b. **Board sync** _(optional)_: If glacier-sync is active and a PR was opened via `gh pr create`, move the linked card to **In Review**. Skip silently if not available.
10. **Report**: Summarize what was done, flag any reviewer concerns, ask if I want to commit

Red → Green → Refactor. Steps 4–7 are gated on `logic` classification only.
Steps 1b and 9b are gated on glacier-sync availability — they are always optional.
If the reviewer flags 🔴 Critical issues, fix them before reporting.

### Skip Protocol
If the prompt includes `[skip-tests]`, bypass steps 4–7 without asking.
- Only valid for `ui` and `config` issue types. Refuse if classification is `logic`.
- Note the skip reason in the PR description under a `## Skipped Tests` heading.
- Use `/skip-tests` slash command to invoke this flow explicitly.

## Branching Rules
- NEVER commit directly to main or master
- When starting work on a new feature, bug fix, or issue:
  1. Create a branch: `git checkout -b <type>/<short-description>`
  2. Branch naming: feat/, fix/, chore/, docs/ prefix
  3. Examples: feat/stripe-webhooks, fix/auth-redirect, chore/update-deps
- When asked to "commit changes" on main, STOP and ask if I want a branch first
- Only push to main via PR merge

# Git Guidelines
- NEVER run `git diff` without flags — it hangs on large diffs
- Always use `git diff --stat` for overview of changes
- Use `git diff --stat --cached` for staged changes
- If you need actual diff content, use `git diff -- <specific-file>` for targeted files only

## Cost Discipline
- Default to Sonnet for implementation work
- Use Haiku sub-agents for exploration, search, and summarization
- Use Opus only for planning, architecture, and complex debugging
- Keep sub-agent prompts short and focused (3-5 word descriptions)
- Use /clear between unrelated tasks to reset context
- Use /cost regularly to monitor token spend

## Code Standards
- TypeScript strict mode, no `any` types
- React: functional components + hooks only
- API responses: `{ success: boolean, data: T, error?: string }`
- Database: Drizzle ORM, never raw SQL in application code
- Auth (when used): Supabase Auth with SSR
- Error handling: try/catch with typed errors, never silent failures

## Communication Style
- Be direct. No corporate fluff
- Suggest alternatives proactively
- Flag cost/complexity tradeoffs explicitly
- If unsure, ask — don't guess

## Available Skills
- /implement — Main workflow entry point (manual)
- /pr-prep — PR description and changelist (manual)
- /skip-tests — Bypass test steps for ui/config issues (manual)
- /cost-check — Token usage and cost analysis (manual)
- /init — Session initialisation (manual)
- /glacier-sync — Sync repo activity with Glacier board (manual)
- code-review — Skill, auto-invoked at workflow step 9
- test-gen — Skill, auto-invoked at workflow step 4
- bbs-brand — Skill, auto-invoked for copy and client-facing content
- deploy-checklist — Skill, auto-invoked before deployments
- glacier-sync — **Optional** skill, auto-invoked at step 1b (branch creation → In Progress), step 9b (PR open → In Review), after PR merge (→ Done), or on manual board sync requests. Requires `GLACIER_ENABLED=true`, `GLACIER_WORKSPACE_ID`, and `GLACIER_PROJECT_ID` in `.env.local`. If not enabled or env vars absent, workflow continues without it.
