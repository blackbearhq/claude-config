---
name: deploy-checklist
description: Pre-deployment verification for Vercel-hosted Next.js projects. Use before deploying or when user mentions deploy, ship, or release.
effort: low
---
# Deploy Checklist

Before any deployment, verify:

## Code Quality
1. Run `npm run typecheck` — zero errors
2. Run `npm run lint` — zero warnings
3. Run `npm test` — all passing

## Environment
4. Check `.env.example` is up to date with any new variables
5. Verify Vercel environment variables are set for production
6. Confirm no hardcoded localhost URLs or dev API keys

## Database
7. Run `npx drizzle-kit status` (or `npx prisma migrate status`) — no pending migrations on production
8. Verify migration is safe (no destructive changes without confirmation) — see `db-migration` skill

## Secrets
9. Run the `secret-scan` skill on the diff — zero findings

## Stripe (if applicable)
10. Confirm webhook endpoint is registered in Stripe dashboard (live mode)
11. Verify webhook signing secret is in production env (not test mode)
12. Confirm idempotency keys are in use for all mutation endpoints

## Final
13. Create a git tag: `git tag -a v{version} -m "Release {version}"`
14. Push: `git push origin main --tags`

Report any failures. Do NOT proceed if any check fails.
