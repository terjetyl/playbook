# Skill: Drizzle Migrations

## The golden rules (learned the hard way)

1. **Never edit an existing migration file** — Drizzle tracks migrations by filename hash. Editing a file that has already run will cause the next deploy to fail or silently skip the change.
2. **Always generate, never handwrite** — run `pnpm db:generate`, review the output SQL, then commit. Don't write SQL by hand in the drizzle/ folder.
3. **Commit the migration file in the same PR as the schema change** — schema.ts and the generated `.sql` file must always be in sync.
4. **Migrations run before the API container starts** — the deploy workflow runs `node dist/migrate.js` as a one-off container step. If the migration fails, `docker compose up -d` never runs and the old API version keeps serving.

## Full workflow

```bash
# 1. Edit the schema
# apps/forms-api/src/db/schema.ts

# 2. Generate the migration
pnpm db:generate
# → creates apps/forms-api/drizzle/XXXX_<slug>.sql
# → updates apps/forms-api/drizzle/meta/_journal.json

# 3. REVIEW the generated SQL before committing
cat apps/forms-api/drizzle/XXXX_<slug>.sql

# 4. Test locally against the test DB
docker compose -f apps/forms-api/docker-compose.test.yml up -d
DATABASE_URL=postgres://test:test@localhost:5433/test_db pnpm --filter @formvault/api test:integration
docker compose -f apps/forms-api/docker-compose.test.yml down -v

# 5. Commit both files together
git add apps/forms-api/drizzle/ apps/forms-api/src/db/schema.ts
git commit -m "db: add <description>"
```

## Common failure modes

### Migration fails: "relation already exists"

Cause: The migration was partially applied (container crashed mid-run) or you're applying to a DB that was manually modified.

Fix — make the migration idempotent with IF NOT EXISTS / IF EXISTS:

```sql
-- Instead of:
CREATE TABLE foo (...);

-- Use:
CREATE TABLE IF NOT EXISTS foo (...);
ALTER TABLE foo ADD COLUMN IF NOT EXISTS bar text;
```

If Drizzle generates non-idempotent SQL (it sometimes does for indexes), add the guard manually in the generated file — this is the one time it's acceptable to edit a generated migration, because you do it **before the first deploy**.

### Migration fails: "column of relation does not exist"

Cause: A previous migration that should have added the column never ran (e.g. it was skipped due to the hash mismatch issue), or schema.ts and the DB are out of sync.

Diagnosis:

```bash
# Check which migrations have run (on production)
ssh deploy@46.225.16.76 \
  "cd /opt/formvault && docker compose -f docker-compose.prod.yml --env-file .env.prod \
   exec -T db psql -U formvault formvault_db -c 'SELECT * FROM drizzle.__drizzle_migrations ORDER BY created_at;'"
```

### Migration hash mismatch ("migration file was tampered")

Cause: Someone edited an already-applied migration file.

Fix: Do NOT edit history. Write a new forward migration:

```sql
-- 0009_fix_foo.sql
ALTER TABLE foo ALTER COLUMN bar SET NOT NULL;
```

### `pnpm db:generate` produces empty migration

Cause: `schema.ts` change hasn't been rebuilt, or Drizzle's introspection is comparing against a different DB.

Fix:

```bash
# Ensure shared package is built first (schema types may come from there)
pnpm --filter @formvault/shared build
# Then re-generate
pnpm db:generate
```

### Deployment stuck: DB not ready when migrate.js runs

This shouldn't happen (the deploy workflow waits for `pg_isready`), but if it does, the safe fix is to re-trigger the deploy. The migration is idempotent — running it twice is safe.

## Adding a column to an existing table

```ts
// schema.ts
export const forms = pgTable("forms", {
  // existing columns...
  archivedAt: timestamp("archived_at"), // nullable = safe default, no migration default needed
});
```

```bash
pnpm db:generate
# Generates: ALTER TABLE "forms" ADD COLUMN "archived_at" timestamp;
```

**Nullable columns need no DEFAULT** — Postgres adds them instantly without a table rewrite.

**NOT NULL columns require a DEFAULT** or must be added in two migrations:

```sql
-- Step 1: add nullable
ALTER TABLE forms ADD COLUMN status text;

-- Step 2 (separate migration after backfilling): add constraint
UPDATE forms SET status = 'active' WHERE status IS NULL;
ALTER TABLE forms ALTER COLUMN status SET NOT NULL;
```

Drizzle doesn't automate this two-step process — do it manually when the column is NOT NULL without a default.

## jsonb columns (fields and settings)

The `fields` and `settings` columns are `jsonb`. Migrations only change the column definition (type, nullability, default) — never the shape of the data inside. Shape validation lives in Zod (`packages/shared/src/index.ts`).

If you need to add a required key to every existing row's jsonb:

```sql
-- Safe: set a default for any existing rows missing the key
UPDATE forms
SET settings = settings || '{"retentionDays": null}'::jsonb
WHERE settings -> 'retentionDays' IS NULL;
```

## Rollback strategy

Drizzle does not auto-generate rollback scripts. To roll back a migration:

1. Write a new forward migration that reverses the change
2. Deploy it normally

```sql
-- 0010_rollback_archived_at.sql
ALTER TABLE forms DROP COLUMN IF EXISTS archived_at;
```

## Migration idempotency test

The integration test suite includes a migration idempotency test (`src/test/integration/migrations.test.ts`). It runs migrations twice and asserts no error. Always ensure this test passes before deploying.

```bash
DATABASE_URL=postgres://test:test@localhost:5433/test_db \
  pnpm --filter @formvault/api test:integration -- --testPathPattern=migrations
```

## Checking migration state

```bash
# Local test DB: see which migrations have been applied
psql postgres://test:test@localhost:5433/test_db \
  -c "SELECT * FROM drizzle.__drizzle_migrations ORDER BY created_at;"

# Production (via SSH)
ssh deploy@46.225.16.76 \
  "docker exec \$(docker ps -qf name=formvault-db) \
   psql -U formvault formvault_db \
   -c 'SELECT id, hash, created_at FROM drizzle.__drizzle_migrations ORDER BY created_at;'"
```
