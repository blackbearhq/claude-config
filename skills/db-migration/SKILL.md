---
name: db-migration
description: Enforces Black Bear Studio's database safety rules. Auto-invoke on any schema file change or when user mentions migration, schema change, ALTER TABLE, drizzle-kit, or prisma migrate.
paths:
  - "**/db/schema.ts"
  - "**/db/schema/*.ts"
  - "**/prisma/schema.prisma"
  - "**/migrations/**"
  - "**/drizzle/**"
model: sonnet
effort: high
tools: Read, Bash, Glob, Grep
color: red
---
# Database Migration Safety

This skill enforces the DB safety rules from CLAUDE.md. Any schema-altering work must flow through this skill.

## Core principle
NEVER alter a shared or production database schema without explicit approval. When in doubt, generate a migration file — don't push.

## Workflow

1. **Identify the target database**
   - Read `.env.local` (or equivalent) to find the active `DATABASE_URL`
   - Classify: local / shared dev / production
   - State the classification out loud before any action

2. **If target is shared or production → STOP**
   - Do not run any schema-altering command
   - Generate a migration file instead (see step 3)
   - Present the migration for review before it's applied

3. **Generate, never push**
   - Drizzle: `npx drizzle-kit generate` (creates SQL migration file, does not apply)
   - Prisma: `npx prisma migrate dev --create-only` (creates migration, does not apply)
   - Commit the migration file to the feature branch — it will be reviewed in the PR

4. **If target is local → direct push is acceptable**
   - Drizzle: `npm run db:push` is OK on a developer's own local DB
   - Prisma: `npx prisma db push` is OK on a developer's own local DB

## Commands requiring explicit approval

NEVER run these against shared or production DBs without asking first:
- `npx drizzle-kit push` / `npm run db:push`
- `npx prisma db push` / `npx prisma migrate deploy`
- Raw SQL: `ALTER TABLE`, `DROP TABLE`, `CREATE TYPE`, `ALTER TYPE`, `DROP COLUMN`, `RENAME`
- Any ORM CLI that syncs schema to a live database

## Safe commands

These are always fine to run:
- `npx drizzle-kit generate` — generates migration SQL files only
- `npx prisma migrate dev --create-only` — generates migration without applying
- `npx drizzle-kit studio` / `npx prisma studio` — read-only DB viewers
- `npm run check` — type checking only

## Breaking changes

Enum changes, column renames, and type changes break instantly for anyone running old code. Before applying any of these:
1. Announce the change in the team channel
2. Deploy the schema change during a maintenance window or with a backward-compatible migration (add new, deprecate old, remove later)
3. Never ship a breaking change in the same PR as the code that depends on it — split into two PRs

## Multi-developer projects

When a database is shared between developers:
- Prefer local DB instances (Docker, `supabase start`, local Postgres) over shared cloud DBs
- If using a shared cloud DB, coordinate schema changes in Slack — never push unannounced
- Enum and type changes are especially dangerous: they break instantly for anyone on old code

## Output format

When this skill activates, always report:
1. Target database classification (local / shared / production)
2. What the change is (added column, new enum, renamed field, etc.)
3. Whether it's breaking (yes / no, with reasoning)
4. What action was taken (generated migration / direct push to local / STOPPED and awaiting approval)

Never silently apply schema changes.
