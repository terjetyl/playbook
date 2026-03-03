# Skill: App Testing

## Testing architecture

Two test layers — both use Vitest:

| Layer       | Command            | Speed         | Needs DB?           | What it tests                                   |
| ----------- | ------------------ | ------------- | ------------------- | ----------------------------------------------- |
| Unit        | `test:unit`        | Fast (<5s)    | No                  | Pure functions, service logic, auth middleware  |
| Integration | `test:integration` | Slower (~30s) | Yes (real Postgres) | Full HTTP request/response through Fastify + DB |

E2E tests (Playwright) are a third layer — see the `playwright-testing` skill.

## Running tests

```bash
# Unit only (fast, no Docker needed)
pnpm --filter @formvault/api test:unit

# Integration (start test DB first)
docker compose -f apps/forms-api/docker-compose.test.yml up -d
DATABASE_URL=postgres://test:test@localhost:5433/test_db pnpm --filter @formvault/api test:integration
docker compose -f apps/forms-api/docker-compose.test.yml down -v

# All tests (CI-equivalent, needs Docker)
pnpm --filter @formvault/api test

# Watch mode (during development)
pnpm --filter @formvault/api test:watch

# Coverage
pnpm --filter @formvault/api test:coverage
```

## Test helpers (`src/test/helpers.ts`)

Always use the shared helpers — don't build your own app instance in tests:

```ts
import { createTestApp, teardown, resetDb } from "../test/helpers";

describe("My route", () => {
  let ctx: Awaited<ReturnType<typeof createTestApp>>;

  beforeAll(async () => {
    ctx = await createTestApp(); // boots Fastify + runs migrations
  });
  afterAll(async () => {
    await teardown(ctx); // closes app + pool
  });
  beforeEach(async () => {
    await resetDb(ctx); // truncates tables between tests
  });
});
```

## Writing integration tests

### Unauthenticated route (public form player)

```ts
it("GET /public/forms/:id returns the form", async () => {
  const form = await ctx.db
    .insert(forms)
    .values({
      tenantId: TEST_ORG_ID,
      name: "Test form",
      fields: [],
      settings: {},
    })
    .returning();

  const res = await ctx.app.inject({
    method: "GET",
    url: `/public/forms/${form[0].id}`,
  });

  expect(res.statusCode).toBe(200);
  expect(res.json().name).toBe("Test form");
});
```

### Authenticated route — mock the JWT, test membership

```ts
// At the top of the test file — mock auth before imports
vi.mock("../plugins/auth", () => ({
  verifyFjordToken: vi.fn().mockResolvedValue({
    sub: "user-123",
    email: "test@example.com",
  }),
}));

it("GET /forms/:tenantId lists forms for org member", async () => {
  // Seed: create org + membership + a form
  const [org] = await ctx.db
    .insert(organisations)
    .values({
      name: "Test Org",
      ownerSub: "user-123",
    })
    .returning();
  await ctx.db.insert(organisationMembers).values({
    orgId: org.id,
    userSub: "user-123",
    role: "owner",
  });
  await ctx.db.insert(forms).values({
    tenantId: org.id,
    name: "My Form",
    fields: [],
    settings: {},
  });

  const res = await ctx.app.inject({
    method: "GET",
    url: `/forms/${org.id}`,
    headers: { authorization: "Bearer fake-token" },
  });

  expect(res.statusCode).toBe(200);
  expect(res.json()).toHaveLength(1);
});

it("returns 403 for non-member", async () => {
  const [org] = await ctx.db
    .insert(organisations)
    .values({
      name: "Someone elses Org",
      ownerSub: "other-user",
    })
    .returning();

  const res = await ctx.app.inject({
    method: "GET",
    url: `/forms/${org.id}`,
    headers: { authorization: "Bearer fake-token" },
    // user-123 is NOT a member of this org
  });

  expect(res.statusCode).toBe(403);
});
```

### Testing encrypted submissions

Submissions are encrypted at rest. In tests, decrypt to assert the content:

```ts
import { decryptData } from "../utils/encryption";

it("POST /public/submissions/:formId stores encrypted data", async () => {
  const res = await ctx.app.inject({
    method: "POST",
    url: `/public/submissions/${form.id}`,
    payload: { answers: { "field-1": "hello" } },
  });

  expect(res.statusCode).toBe(201);
  const { respondentToken } = res.json();
  expect(respondentToken).toBeTruthy();

  // Verify it's encrypted in DB
  const [stored] = await ctx.db
    .select()
    .from(formSubmissions)
    .where(eq(formSubmissions.formId, form.id));

  expect(stored.encryptedData).not.toContain("hello"); // never stored in plaintext

  // Decrypt and verify
  const decrypted = decryptData(
    stored.encryptedData,
    process.env.ENCRYPTION_KEY!,
  );
  expect(JSON.parse(decrypted).answers["field-1"]).toBe("hello");
});
```

## Critical paths — always test these

### 1. Health endpoint

```ts
it("GET /health returns 200", async () => {
  const res = await ctx.app.inject({ method: "GET", url: "/health" });
  expect(res.statusCode).toBe(200);
  expect(res.json().status).toBe("ok");
});
```

### 2. 401 for unauthenticated requests

```ts
it("returns 401 without auth header", async () => {
  const res = await ctx.app.inject({ method: "GET", url: "/forms/some-uuid" });
  expect(res.statusCode).toBe(401);
});
```

### 3. Auth middleware rejects bad tokens

```ts
// src/plugins/auth.test.ts (unit test)
import { verifyFjordToken } from "./auth";
import * as jose from "jose";
vi.mock("jose");

it("throws 401 on expired token", async () => {
  vi.mocked(jose.jwtVerify).mockRejectedValue(
    new jose.errors.JWTExpired("expired"),
  );
  await expect(verifyFjordToken("expired.token")).rejects.toMatchObject({
    statusCode: 401,
  });
});
```

### 4. Migration idempotency

```ts
// src/test/integration/migrations.test.ts
it("can apply migrations twice without error", async () => {
  const pool = new Pool({ connectionString: process.env.DATABASE_URL });
  const db = drizzle(pool);
  await migrate(db, { migrationsFolder: "./drizzle" });
  await expect(
    migrate(db, { migrationsFolder: "./drizzle" }),
  ).resolves.not.toThrow();
  await pool.end();
});
```

## Testing new features — checklist

When implementing any new route or feature:

- [ ] Happy path: the feature works as expected
- [ ] Auth: returns `401` without a token
- [ ] Membership: returns `403` for a user who isn't in the org
- [ ] Not found: returns `404` for a non-existent resource
- [ ] Validation: returns `400` for a malformed request body
- [ ] If submissions are involved: assert on the decrypted content

## Unit tests for utility functions

```ts
// src/utils/encryption.test.ts
import { encryptData, decryptData } from "./encryption";

it("round-trips data correctly", () => {
  const original = JSON.stringify({
    name: "Alice",
    email: "alice@example.com",
  });
  const key = "test-key-32-chars-padded-00000000";
  const encrypted = encryptData(original, key);
  expect(encrypted).not.toContain("Alice");
  const decrypted = decryptData(encrypted, key);
  expect(decrypted).toBe(original);
});
```

## CI integration

Tests are gated in CI before any Docker build or deploy. The `ci.yml` runs:

1. `test:unit` (no services needed — fast feedback)
2. `test:integration` (with a Postgres service container)
3. Playwright E2E (against the built Vite app)

A failing unit test blocks the integration tests. A failing integration test blocks the Playwright tests and the deploy job.
