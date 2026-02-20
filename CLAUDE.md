# BlackBear Studio — Global Claude Code Config

## Identity
- Builder: BlackBear Studio (blackbear.so)
- Core Stack: TypeScript, Next.js, React, Tailwind, Vercel
- Common Tools: PostgreSQL + pgvector, Drizzle ORM, Stripe, Supabase Auth

## Workflow Rules
- ALWAYS run `npm run check` before considering a task complete
- ALWAYS run tests after modifying business logic
- Use Prettier formatting (auto-configured via .prettierrc)
- Commit messages: conventional commits (feat:, fix:, chore:, docs:)
- Do NOT add "Co-Authored-By: Claude" to commit messages
- Never commit .env files. Use .env.example for documentation

## Workflow: Implementing Issues (TDD)
When asked to implement a feature, fix, or issue, follow this sequence:

0. **Classify**: Identify the issue type before doing anything else:
   - `logic` — business logic, API, DB, auth, data transformation → full TDD required
   - `ui` — components, layout, styling, copy → skip steps 4–7, use visual review instead
   - `config` — tooling, env, scripts, docs → skip steps 4–7 entirely
   State the classification out loud before proceeding.

1. **Branch**: Create a feature branch from main (never commit to main directly)
2. **Explore**: Delegate to the `explorer` agent to find relevant files and patterns
3. **Plan**: Present a brief implementation plan (max 7 steps). Wait for my approval
4. **Test first** _(logic only)_: Delegate to the `test-gen` agent to write failing tests that define the expected behavior
5. **Verify red** _(logic only)_: Run tests — confirm they FAIL. If they pass, the tests aren't testing anything new
6. **Implement**: Write the minimum code to make tests pass (main session, Sonnet)
7. **Verify green** _(logic only)_: Run `npm run test:run -- --testPathPattern=<changed-file>`. Confirm they PASS. Never run the full test suite during TDD cycles — full suite runs at PR prep stage only.
8. **Refactor**: Clean up while keeping tests green
9. **Review**: Delegate to the `code-reviewer` agent to review all changes
10. **Report**: Summarize what was done, flag any reviewer concerns, ask if I want to commit

Red → Green → Refactor. Steps 4–7 are gated on `logic` classification only.
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
- /code-review — Reviews code for quality, security, and patterns
- /test-gen — Generates tests following project conventions
- /pr-prep — Prepares PR description and changelist
- /explore-codebase — Read-only codebase analysis (Haiku)
