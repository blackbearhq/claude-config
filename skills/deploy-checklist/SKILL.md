---
name: deploy-checklist
description: "Runs build checks, validates environment variables, verifies database migration status, and checks Stripe webhook configuration for Vercel-hosted Next.js projects. Use before deploying, shipping, releasing, or when the user mentions going to production."
---
# Deploy Checklist

Run each section in order before any deployment. If any check fails, stop and fix before proceeding.

## Code Quality
1. Run `npm run typecheck` — zero errors
2. Run `npm run lint` — zero warnings
3. Run `npm test` — all passing

> **If failing:** Fix type/lint errors first. For test failures, check if they are flaky (`npm test -- --retry 1`) or genuine regressions.

## Environment
4. Check `.env.example` is up to date with any new variables
5. Verify Vercel environment variables are set for production (`vercel env ls`)
6. Confirm no hardcoded localhost URLs or dev API keys — `grep -r "localhost" src/`

> **If missing vars:** Add to Vercel via `vercel env add <NAME> production` before proceeding.

## Database
7. Run `npx prisma migrate status` — no pending migrations
8. Verify migration is safe (no destructive changes without confirmation)

> **If pending migrations:** Review the migration SQL, then run `npx prisma migrate deploy` after explicit approval. Never auto-apply destructive changes.

## Stripe (if SkillsFrame)
9. Confirm webhook endpoint is registered in Stripe dashboard
10. Verify webhook signing secret is in production env

## Final
11. Create a git tag: `git tag -a v{version} -m "Release {version}"`
12. Push: `git push origin main --tags`
13. After deploy completes, verify the production URL loads and key routes respond
