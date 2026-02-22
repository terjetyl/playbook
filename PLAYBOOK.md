# Production Deployment Playbook

Personal runbook for deploying apps on the shared Hetzner server.
All apps follow the same pattern: **pnpm monorepo → GHCR Docker images → SSH deploy → Docker Compose → Traefik (TLS) + Caddy (SPA+API proxy) + Postgres**.

## Live Apps

| App       | Domain       | Repo                                                        | Server path      |
| --------- | ------------ | ----------------------------------------------------------- | ---------------- |
| invoicia  | invoicia.eu  | [terjetyl/invoicia](https://github.com/terjetyl/invoicia)   | `/opt/invoicia`  |
| formvault | formvault.eu | [terjetyl/formvault](https://github.com/terjetyl/formvault) | `/opt/formvault` |

**Production server IP:** `46.225.16.76`

---

## Architecture

```
Internet
  │
  ▼
Traefik (Hetzner VPS, ports 80/443)
  │  TLS termination (Let's Encrypt)
  │  Routes by domain/label
  │
  ├──▶ Caddy container (per app)
  │       ├── Serves SPA static files from /srv
  │       └── Proxies /api/* → api:PORT (rewrites to /v1/*)
  │
  ├──▶ API container (Fastify/Node)
  │       └── Connects to Postgres via internal Docker network
  │
  └──▶ Postgres container
           └── Persistent volume
```

**Key points:**

- Traefik handles TLS — Caddy runs with `auto_https off`
- Each app has its own Docker Compose stack in `/opt/<appname>/`
- Docker images are built via GitHub Actions and pushed to GHCR
- Secrets & config are written to `.env.prod` on the server by the deploy workflow
- DB migrations run **as part of every deploy**, before the API starts — never manually on the server

---

## GitHub Dependabot

Dependabot automates dependency updates and security patches for all repositories. To enable:

1. Add a `.github/dependabot.yml` file to configure update frequency and package ecosystems.
2. Review and merge Dependabot PRs regularly to keep dependencies secure and up-to-date.

For more details, see [GitHub Dependabot documentation](https://docs.github.com/en/code-security/dependabot).

---

## Checklist: Deploying a New App

### 1. DNS

#### Register the domain (Gandi.net)

1. Go to [gandi.net](https://www.gandi.net) → search for `<app>.eu` → purchase.
2. The domain will appear in **Domain** → your account namespace.

#### Configure DNS records

Gandi.net uses its own DNS by default. Update records via **Domain → `<app>.eu` → DNS Records**:

| Type | Name  | Value         | TTL |
| ---- | ----- | ------------- | --- | ---------------------------------------- |
| A    | `@`   | `<server-ip>` | 3h  |
| A    | `api` | `<server-ip>` | 3h  | ← only if using a separate API subdomain |
| A    | `www` | `<server-ip>` | 3h  | ← optional redirect via Caddy            |

> TTL `3h` (10800s) is fine for production. Use `5m` (300s) during initial setup so you can correct mistakes quickly, then raise it.

#### Gandi DNS via LiveDNS API (optional automation)

If you prefer CLI over the UI:

```bash
# Set an A record using the Gandi LiveDNS API
curl -s -X PUT \
  -H "Authorization: Bearer <GANDI_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"rrset_values": ["<server-ip>"], "rrset_ttl": 10800}' \
  "https://api.gandi.net/v5/livedns/domains/<app>.eu/records/@/A"

# And for the api subdomain
curl -s -X PUT \
  -H "Authorization: Bearer <GANDI_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"rrset_values": ["<server-ip>"], "rrset_ttl": 10800}' \
  "https://api.gandi.net/v5/livedns/domains/<app>.eu/records/api/A"
```

Get your API key: **Gandi account → Security → Developer access → Personal Access Token**.

#### Verify propagation

```bash
dig +short <app>.eu A
dig +short api.<app>.eu A
# Both should return <server-ip>. Traefik will not issue a TLS cert until DNS resolves.
```

### 2. FjordID (if app needs auth)

Go to [auth.fjordid.eu](https://auth.fjordid.eu) → Realm `fjordid` → Clients:

1. Create **API client** (bearer-only):
   - Client ID: `fjid_xxxxxxxx` (copy this)
   - Access type: Bearer-only

2. Create **Web client** (public, OIDC):
   - Client ID: `fjid_yyyyyyyy` (copy this)
   - Valid redirect URIs: `https://<app>.eu/auth/callback`, `https://<app>.eu/`
   - Web Origins: `https://<app>.eu`

#### MCP Server (GitHub Copilot)

FjordID publishes an MCP server so Copilot can create and manage clients directly from the editor.

1. Create an API key at [fjordid.eu/console/api-keys](https://fjordid.eu/console/api-keys) — format `fjak_...`.
2. Add `.vscode/mcp.json` to the repo (add to `.gitignore` — the file contains a secret):

```json
{
  "servers": {
    "fjordid": {
      "command": "npx",
      "args": ["-y", "@fjordid/mcp"],
      "env": { "FJORDID_API_KEY": "fjak_your_key_here" }
    }
  }
}
```

3. Restart VS Code — Copilot can now call:
   - `fjordid_list_apps`, `fjordid_get_app`
   - `fjordid_create_app`, `fjordid_update_app`, `fjordid_delete_app`
   - `fjordid_add_redirect_uri`, `fjordid_remove_redirect_uri`
   - `fjordid_rotate_secret`
   - `fjordid_suggest_config` (AI-powered config recommendation)

> The API key gives full access to your FjordID account — never commit it. Add `.vscode/mcp.json` to `.gitignore`.

### 3. GitHub Environment

All environment variables and secrets are configured via the **GitHub CLI** (`gh`). The UI approach is error-prone — use the commands below instead.

#### Prerequisites

```bash
# Verify gh is installed
gh --version

# Verify you're authenticated
gh auth status

# If not logged in:
gh auth login
```

#### Create the environment

```bash
gh api repos/terjetyl/<appname>/environments/production --method PUT
```

#### Get the SSH private key from your dev machine

The deploy user's SSH key already exists on your dev machine (you use it for passwordless access to the server). Grab it directly:

```bash
# Find which key is configured for the server
ssh -G <server-ip> | grep identityfile

# Print the private key (use the path from the command above, typically id_ed25519 or id_rsa)
cat ~/.ssh/id_ed25519
```

Copy the full output (including `-----BEGIN...` and `-----END...` lines) for the next step.

#### Set secrets

```bash
REPO="terjetyl/<appname>"
ENV="production"

# SSH key: pipe directly so the multi-line value is set correctly
cat ~/.ssh/id_ed25519 | gh secret set SSH_PRIVATE_KEY --env "$ENV" --repo "$REPO"

gh secret set POSTGRES_PASSWORD --env "$ENV" --repo "$REPO" --body "changeme-strong-password"
```

#### Set variables

```bash
REPO="terjetyl/<appname>"
ENV="production"

gh variable set APP_HOSTNAME          --env "$ENV" --repo "$REPO" --body "<app>.eu"
gh variable set API_HOSTNAME          --env "$ENV" --repo "$REPO" --body "api.<app>.eu"
gh variable set ACME_EMAIL            --env "$ENV" --repo "$REPO" --body "terje@example.com"
gh variable set SERVER_HOST           --env "$ENV" --repo "$REPO" --body "<server-ip>"
gh variable set SERVER_USER           --env "$ENV" --repo "$REPO" --body "deploy"
gh variable set POSTGRES_USER         --env "$ENV" --repo "$REPO" --body "<appname>"
gh variable set POSTGRES_DB           --env "$ENV" --repo "$REPO" --body "<appname>_db"

# FjordID — skip if app has no auth
gh variable set FJORDID_URL           --env "$ENV" --repo "$REPO" --body "https://auth.fjordid.eu"
gh variable set FJORDID_REALM         --env "$ENV" --repo "$REPO" --body "fjordid"
gh variable set FJORDID_API_CLIENT_ID --env "$ENV" --repo "$REPO" --body "fjid_xxxxxxxx"
gh variable set FJORDID_WEB_CLIENT_ID --env "$ENV" --repo "$REPO" --body "fjid_yyyyyyyy"
```

#### Verify

```bash
gh variable list --env production --repo "$REPO"
gh secret list  --env production --repo "$REPO"
```

**Variable reference** (for documentation purposes):

| Variable                | Description            | Example                   |
| ----------------------- | ---------------------- | ------------------------- |
| `APP_HOSTNAME`          | Main domain            | `formvault.eu`            |
| `API_HOSTNAME`          | API domain             | `api.formvault.eu`        |
| `ACME_EMAIL`            | Let's Encrypt email    | `terje@example.com`       |
| `SERVER_HOST`           | Hetzner server IP      | `1.2.3.4`                 |
| `SERVER_USER`           | SSH user on server     | `deploy`                  |
| `POSTGRES_USER`         | DB user (optional)     | `formvault`               |
| `POSTGRES_DB`           | DB name (optional)     | `formvault_db`            |
| `FJORDID_URL`           | FjordID base URL       | `https://auth.fjordid.eu` |
| `FJORDID_REALM`         | FjordID realm          | `fjordid`                 |
| `FJORDID_API_CLIENT_ID` | API Keycloak client ID | `fjid_xxxxxxxx`           |
| `FJORDID_WEB_CLIENT_ID` | Web Keycloak client ID | `fjid_yyyyyyyy`           |

| Secret              | Description                       |
| ------------------- | --------------------------------- |
| `SSH_PRIVATE_KEY`   | Private key for server SSH access |
| `POSTGRES_PASSWORD` | Database password                 |

### 4. Infra Files in the Repo

Each app needs these files (copy from invoicia as templates):

```
infra/
  Caddyfile.prod         # Caddy config for production (auto_https off)
  compose.env.example    # Local dev .env template
  docker/
    api/
      Dockerfile         # Multi-stage Node build
    caddy/
      Dockerfile         # Caddy + SPA build
      entrypoint.sh      # Generates /config.js for SPA at runtime
docker-compose.prod.yml  # Production Compose stack
.vscode/
  mcp.json               # FjordID MCP server (gitignored — contains API key)
.github/
  copilot-instructions.md # Copilot context: links to shared playbook
  workflows/
    ci.yml               # Test + typecheck on every PR/push
    deploy-production.yml # Build GHCR images + SSH deploy
```

### 5. GitHub Copilot Instructions

Every repo gets a `.github/copilot-instructions.md` that points Copilot at this shared playbook so all AI assistance is consistent across projects.

```markdown
# Copilot Instructions

This project follows the shared production playbook.
Read and apply the conventions, patterns, and rules described in:

https://raw.githubusercontent.com/terjetyl/playbook/refs/heads/main/PLAYBOOK.md

Key things to be aware of:

- pnpm monorepo (apps/api, apps/web, packages/shared)
- Fastify API with Drizzle ORM + Postgres
- Docker Compose + Traefik + Caddy for production
- Migrations run automatically on every deploy via `dist/migrate.js` — never manually
- Auth via FjordID (self-hosted Keycloak) — verify JWTs with jose
- Tests: Vitest for unit tests, real Postgres for integration tests (never mock the DB)
- All secrets via GitHub Environment variables — never hardcoded
```

Update the bullet points with anything app-specific (e.g. domain, special env vars, notable business rules).

### 6. docker-compose.prod.yml Pattern

Services needed for every app:

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-<appname>}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB:-<appname>_db}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "pg_isready -U ${POSTGRES_USER:-<appname>} -d ${POSTGRES_DB:-<appname>_db}",
        ]
      interval: 5s
      timeout: 5s
      retries: 20
    restart: unless-stopped

  api:
    image: ${API_IMAGE}
    environment:
      PORT: 3002
      HOST: 0.0.0.0
      POSTGRES_HOST: db
      POSTGRES_PORT: 5432
      POSTGRES_USER: ${POSTGRES_USER:-<appname>}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB:-<appname>_db}
      # Add app-specific env vars here
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  caddy:
    image: ${CADDY_IMAGE}
    environment:
      APP_DOMAIN: ${APP_HOSTNAME}
      ACME_EMAIL: ${ACME_EMAIL}
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=fjordid_traefik"
      - "traefik.http.routers.<appname>-web.rule=Host(`${APP_HOSTNAME}`)"
      - "traefik.http.routers.<appname>-web.entrypoints=websecure"
      # ...additional Traefik labels
    networks:
      - default
      - fjordid_traefik
    restart: unless-stopped

volumes:
  postgres_data:

networks:
  fjordid_traefik:
    external: true
```

### 7. Caddyfile.prod Pattern

```caddy
{
  auto_https off   # Traefik handles TLS
}

:80 {
  log {
    output stdout
    format json
  }
  encode gzip

  route {
    handle_path /api/* {
      rewrite * /v1{path}
      reverse_proxy <appname>-api:3002
    }

    handle {
      root * /srv
      try_files {path} /index.html
      file_server
    }
  }
}
```

### 8. API Dockerfile Pattern

```dockerfile
FROM node:22-slim AS build
WORKDIR /app
RUN corepack enable

COPY package.json pnpm-lock.yaml pnpm-workspace.yaml tsconfig.base.json ./
COPY apps/<api-dir>/package.json apps/<api-dir>/package.json
COPY packages/shared/package.json packages/shared/package.json

RUN pnpm install --frozen-lockfile
COPY apps/<api-dir> apps/<api-dir>
COPY packages/shared packages/shared
RUN pnpm --filter @<scope>/shared build
RUN pnpm --filter @<scope>/api build

FROM node:22-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production
RUN corepack enable

COPY package.json pnpm-lock.yaml pnpm-workspace.yaml tsconfig.base.json ./
COPY apps/<api-dir>/package.json apps/<api-dir>/package.json
COPY packages/shared/package.json packages/shared/package.json
RUN pnpm install --prod --frozen-lockfile --filter @<scope>/api...

COPY --from=build /app/apps/<api-dir>/dist /app/apps/<api-dir>/dist
COPY --from=build /app/packages/shared/dist /app/packages/shared/dist
COPY --from=build /app/apps/<api-dir>/drizzle /app/apps/<api-dir>/drizzle

WORKDIR /app/apps/<api-dir>
EXPOSE 3002
CMD ["node", "dist/server.js"]
```

### 9. Deploy Workflow Pattern

Key steps from the deploy job in `.github/workflows/deploy-production.yml`:

```yaml
- name: Setup SSH key
  uses: webfactory/ssh-agent@v0.9.0
  with:
    ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

- name: Create deployment directory
  run: ssh ${{ vars.SERVER_USER }}@${{ vars.SERVER_HOST }} "mkdir -p /opt/<appname>"

- name: Copy deployment files
  run: scp docker-compose.prod.yml ${{ vars.SERVER_USER }}@${{ vars.SERVER_HOST }}:/opt/<appname>/

- name: Create production environment file
  run: |
    ssh ${{ vars.SERVER_USER }}@${{ vars.SERVER_HOST }} << 'SSH_EOF'
    cat > /opt/<appname>/.env.prod << 'END'
    APP_HOSTNAME=${{ vars.APP_HOSTNAME }}
    ...all vars and secrets...
    API_IMAGE=${{ needs.build-images.outputs.api-image }}
    CADDY_IMAGE=${{ needs.build-images.outputs.caddy-image }}
    END
    SSH_EOF

- name: Deploy application
  run: |
    ssh ${{ vars.SERVER_USER }}@${{ vars.SERVER_HOST }} "
      cd /opt/<appname>

      # 1. Pull new images
      docker compose -f docker-compose.prod.yml --env-file .env.prod pull

      # 2. Start DB first, wait until healthy
      docker compose -f docker-compose.prod.yml --env-file .env.prod up -d db
      until docker compose -f docker-compose.prod.yml --env-file .env.prod exec -T db sh -lc 'pg_isready -U \"\$POSTGRES_USER\" -d \"\$POSTGRES_DB\"' >/dev/null 2>&1; do
        sleep 2
      done

      # 3. Run migrations BEFORE starting the API (critical: never skip this)
      docker compose -f docker-compose.prod.yml --env-file .env.prod run --rm api node dist/migrate.js

      # 4. Start all services
      docker compose -f docker-compose.prod.yml --env-file .env.prod up -d --force-recreate

      docker image prune -f
    "
```

---

## Database Migrations

Migrations are managed with **Drizzle Kit** and run automatically on every production deploy.

### How it works

1. `drizzle-kit generate:pg` creates SQL migration files in `apps/<api-dir>/drizzle/`
2. The API Dockerfile copies the `drizzle/` folder into the image
3. `dist/migrate.js` applies any pending migrations using Drizzle's migrate helper
4. The deploy workflow runs `migrate.js` **before** starting the API container

### The migrate script (`apps/<api-dir>/src/migrate.ts`)

```typescript
import { drizzle } from "drizzle-orm/node-postgres";
import { migrate } from "drizzle-orm/node-postgres/migrator";
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const db = drizzle(pool);

await migrate(db, { migrationsFolder: "./drizzle" });
console.log("Migrations applied successfully");
await pool.end();
process.exit(0);
```

### Migration workflow

```bash
# 1. Make schema changes in src/db/schema.ts

# 2. Generate migration file
pnpm db:generate
# Creates: drizzle/XXXX_<description>.sql

# 3. Commit the migration file alongside the schema change
git add drizzle/ src/db/schema.ts
git commit -m "db: add <description>"

# 4. Push to main → deploy workflow runs migrations automatically
git push
```

### Rules

- **Never edit existing migration files** — always generate a new one
- **Always commit migration files** — they must be included in the Docker image
- **Never run migrations manually on the server** — always go through the deploy workflow so migrations are tracked in git history
- Migrations run in a one-off `docker compose run --rm api` container, not in the live API container, so the running API is unaffected until step 4
- If a migration fails, the deploy stops before `up -d` — the old API version keeps running

### Rolling back

Drizzle does not auto-generate rollback scripts. To roll back:

1. Write a new forward migration that reverses the change
2. Deploy it normally

---

## FjordID Auth Integration

All apps that need authentication use **FjordID** (self-hosted Keycloak at `auth.fjordid.eu`).

### API side (JWT verification)

```typescript
// Fastify plugin: verify Bearer token from FjordID
const JWKS_URL = `${FJORDID_URL}/realms/${FJORDID_REALM}/protocol/openid-connect/certs`;
// Use jose library to validate JWTs
```

Environment variables the API needs:

```
FJORDID_URL=https://auth.fjordid.eu
FJORDID_REALM=fjordid
FJORDID_CLIENT_ID=fjid_xxxxxxxx       # API client
FJORDID_ALLOWED_AZP=fjid_yyyyyyyy      # Web client (allowed azp claim)
```

### Web/SPA side (OIDC login)

Uses `oidc-client-ts` library:

```typescript
// Auth config read from /config.js at runtime (injected by Caddy entrypoint)
// This allows changing FjordID settings without rebuilding the image
```

The Caddy `entrypoint.sh` generates `/srv/config.js` at container start:

```bash
cat > /srv/config.js << EOF
window.__config__ = {
  FJORDID_URL: "${FJORDID_URL}",
  FJORDID_REALM: "${FJORDID_REALM}",
  FJORDID_CLIENT_ID: "${FJORDID_WEB_CLIENT_ID}"
};
EOF
```

### MCP Server

FjordID ships `@fjordid/mcp` — an MCP server for managing clients via Copilot. Configure it in `.vscode/mcp.json` (gitignored):

```json
{
  "servers": {
    "fjordid": {
      "command": "npx",
      "args": ["-y", "@fjordid/mcp"],
      "env": { "FJORDID_API_KEY": "fjak_your_key_here" }
    }
  }
}
```

Get your API key from [fjordid.eu/console/api-keys](https://fjordid.eu/console/api-keys). With this in place you can ask Copilot to create clients, rotate secrets, or add redirect URIs without leaving VS Code.

---

## Testing

All apps use **Vitest** for both unit and integration tests. Integration tests run against a real ephemeral Postgres — never mock the database.

### Setup

```bash
# Install in the API package
pnpm --filter @<scope>/api add -D vitest @vitest/coverage-v8
```

`apps/<api-dir>/vitest.config.ts`:

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    globals: true,
    environment: "node",
    coverage: {
      provider: "v8",
      reporter: ["text", "html"],
      include: ["src/**/*.ts"],
      exclude: ["src/**/*.test.ts", "src/test/**"],
    },
  },
});
```

`apps/<api-dir>/package.json` scripts:

```json
{
  "scripts": {
    "test": "vitest run",
    "test:unit": "vitest run --testPathPattern='\\.test\\.ts$' --exclude='**/integration/**'",
    "test:integration": "vitest run src/test/integration",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage"
  }
}
```

### File Structure

```
apps/api/
  src/
    routes/
      invoices.ts
      invoices.test.ts          ← integration test (uses real DB)
    services/
      invoice-service.ts
      invoice-service.test.ts   ← unit test (mocks repo)
    utils/
      format.ts
      format.test.ts
    test/
      helpers.ts                ← shared test app factory + teardown
      integration/
        health.test.ts
        migrations.test.ts
  docker-compose.test.yml       ← ephemeral Postgres for local integration tests
  vitest.config.ts
```

### Unit Tests — Service Layer

Unit tests cover pure business logic without touching the DB or network. Mock any repository dependencies.

```typescript
// apps/api/src/services/invoice-service.test.ts
import { describe, it, expect, vi, beforeEach } from "vitest";
import { InvoiceService } from "./invoice-service";

const mockRepo = {
  findById: vi.fn(),
  create: vi.fn(),
  update: vi.fn(),
  delete: vi.fn(),
  findByUser: vi.fn(),
};

describe("InvoiceService", () => {
  let service: InvoiceService;

  beforeEach(() => {
    vi.clearAllMocks();
    service = new InvoiceService(mockRepo);
  });

  it("returns null when invoice not found", async () => {
    mockRepo.findById.mockResolvedValue(null);
    const result = await service.getById("non-existent-id");
    expect(result).toBeNull();
  });

  it("throws when creating invoice with zero amount", async () => {
    await expect(
      service.create({ title: "Bad invoice", amount: 0 }),
    ).rejects.toThrow("Amount must be greater than zero");
  });
});
```

For pure functions (calculations, formatters):

```typescript
// apps/api/src/utils/format.test.ts
import { describe, it, expect } from "vitest";
import { calculateTotal } from "./format";

describe("calculateTotal", () => {
  it("sums line items correctly", () => {
    const items = [
      { quantity: 2, unitPrice: 100 },
      { quantity: 1, unitPrice: 50 },
    ];
    expect(calculateTotal(items)).toBe(250);
  });

  it("returns 0 for empty items", () => {
    expect(calculateTotal([])).toBe(0);
  });
});
```

### Integration Tests — API Routes

Integration tests send real HTTP requests via Fastify's `inject` against the full app instance (plugins, auth, DB).

#### Test DB — Local

```yaml
# apps/api/docker-compose.test.yml
services:
  db-test:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: test
      POSTGRES_PASSWORD: test
      POSTGRES_DB: test_db
    ports:
      - "5433:5432"
    tmpfs:
      - /var/lib/postgresql/data # fast in-memory storage, no persistence needed
```

```bash
# Start test DB, run integration tests, tear down
docker compose -f docker-compose.test.yml up -d
DATABASE_URL=postgres://test:test@localhost:5433/test_db pnpm test:integration
docker compose -f docker-compose.test.yml down -v
```

#### Test Helpers (`src/test/helpers.ts`)

```typescript
import { buildApp } from "../app";
import { drizzle } from "drizzle-orm/node-postgres";
import { migrate } from "drizzle-orm/node-postgres/migrator";
import { Pool } from "pg";
import * as schema from "../db/schema";

export async function createTestApp() {
  const pool = new Pool({
    connectionString:
      process.env.DATABASE_URL ?? "postgres://test:test@localhost:5433/test_db",
  });
  const db = drizzle(pool, { schema });

  // Apply all pending migrations to the test DB
  await migrate(db, { migrationsFolder: "./drizzle" });

  const app = buildApp({ db });
  await app.ready();

  return { app, db, pool };
}

export async function teardown(ctx: Awaited<ReturnType<typeof createTestApp>>) {
  await ctx.app.close();
  await ctx.pool.end();
}

/** Truncate all app tables between tests to keep them isolated */
export async function resetDb(ctx: Awaited<ReturnType<typeof createTestApp>>) {
  await ctx.db.execute(
    sql`TRUNCATE TABLE invoices, users RESTART IDENTITY CASCADE`,
  );
}
```

#### Critical: Health Endpoint (`src/test/integration/health.test.ts`)

```typescript
import { describe, it, expect, beforeAll, afterAll } from "vitest";
import { createTestApp, teardown } from "../helpers";

describe("GET /api/health", () => {
  let ctx: Awaited<ReturnType<typeof createTestApp>>;

  beforeAll(async () => {
    ctx = await createTestApp();
  });
  afterAll(async () => {
    await teardown(ctx);
  });

  it("returns 200 with status ok", async () => {
    const res = await ctx.app.inject({ method: "GET", url: "/api/health" });
    expect(res.statusCode).toBe(200);
    expect(res.json()).toMatchObject({ status: "ok" });
  });

  it("health check confirms DB is reachable", async () => {
    const res = await ctx.app.inject({ method: "GET", url: "/api/health" });
    // If DB is unreachable the health route should return 503
    expect(res.statusCode).not.toBe(503);
  });
});
```

#### Critical: Migration Idempotency (`src/test/integration/migrations.test.ts`)

```typescript
import { describe, it, expect } from "vitest";
import { drizzle } from "drizzle-orm/node-postgres";
import { migrate } from "drizzle-orm/node-postgres/migrator";
import { Pool } from "pg";

describe("DB migrations", () => {
  it("can apply migrations twice without error (idempotency)", async () => {
    const pool = new Pool({ connectionString: process.env.DATABASE_URL });
    const db = drizzle(pool);

    // First run
    await migrate(db, { migrationsFolder: "./drizzle" });
    // Second run — must be a no-op, not throw
    await expect(
      migrate(db, { migrationsFolder: "./drizzle" }),
    ).resolves.not.toThrow();

    await pool.end();
  });
});
```

#### Authenticated Routes

Mock the FjordID JWT verification at the plugin level so integration tests don't depend on a live Keycloak:

```typescript
// apps/api/src/routes/invoices.test.ts
import {
  describe,
  it,
  expect,
  beforeAll,
  afterAll,
  beforeEach,
  vi,
} from "vitest";
import { createTestApp, teardown, resetDb } from "../test/helpers";

// Mock JWT verification — replace with vi.mock pointing at your actual auth plugin path
vi.mock("../plugins/auth", () => ({
  verifyToken: vi.fn().mockResolvedValue({
    sub: "user-123",
    email: "test@example.com",
  }),
}));

describe("Invoices API", () => {
  let ctx: Awaited<ReturnType<typeof createTestApp>>;

  beforeAll(async () => {
    ctx = await createTestApp();
  });
  afterAll(async () => {
    await teardown(ctx);
  });
  beforeEach(async () => {
    await resetDb(ctx);
  });

  it("GET /api/invoices returns empty list for new user", async () => {
    const res = await ctx.app.inject({
      method: "GET",
      url: "/api/invoices",
      headers: { authorization: "Bearer test-token" },
    });
    expect(res.statusCode).toBe(200);
    expect(res.json()).toEqual([]);
  });

  it("POST /api/invoices creates an invoice", async () => {
    const res = await ctx.app.inject({
      method: "POST",
      url: "/api/invoices",
      headers: { authorization: "Bearer test-token" },
      payload: { title: "Test Invoice", amount: 1000 },
    });
    expect(res.statusCode).toBe(201);
    expect(res.json()).toMatchObject({ title: "Test Invoice", amount: 1000 });
  });

  it("GET /api/invoices/:id returns 404 for unknown id", async () => {
    const res = await ctx.app.inject({
      method: "GET",
      url: "/api/invoices/00000000-0000-0000-0000-000000000000",
      headers: { authorization: "Bearer test-token" },
    });
    expect(res.statusCode).toBe(404);
  });

  it("rejects request without auth token", async () => {
    const res = await ctx.app.inject({
      method: "GET",
      url: "/api/invoices",
    });
    expect(res.statusCode).toBe(401);
  });
});
```

#### Critical: Auth Middleware

```typescript
// apps/api/src/plugins/auth.test.ts
import { describe, it, expect, vi } from "vitest";
import { verifyFjordToken } from "./auth";
import * as jose from "jose";

vi.mock("jose");

describe("verifyFjordToken", () => {
  it("throws 401 when token is expired", async () => {
    vi.mocked(jose.jwtVerify).mockRejectedValue(
      new jose.errors.JWTExpired("JWT expired"),
    );
    await expect(verifyFjordToken("expired.token.here")).rejects.toMatchObject({
      statusCode: 401,
    });
  });

  it("throws 401 when token is malformed", async () => {
    vi.mocked(jose.jwtVerify).mockRejectedValue(new jose.errors.JWTInvalid());
    await expect(verifyFjordToken("bad-token")).rejects.toMatchObject({
      statusCode: 401,
    });
  });
});
```

### Critical Paths — Always Test

| Path                                         | Type        | Why                                                                    |
| -------------------------------------------- | ----------- | ---------------------------------------------------------------------- |
| `GET /api/health`                            | Integration | Confirms app boots and DB is reachable                                 |
| JWT auth middleware                          | Unit        | Must reject expired/malformed tokens                                   |
| DB migration idempotency                     | Integration | Migrations run on every deploy — must never fail on re-run             |
| Core CRUD routes (list, create, get, delete) | Integration | The primary user-facing paths                                          |
| Business logic / calculations                | Unit        | e.g. totals, tax — correctness is critical                             |
| `401` for unauthenticated requests           | Integration | Security — never let protected routes return 200 without a valid token |

### CI: Tests in GitHub Actions

Add to `.github/workflows/ci.yml`:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: "pnpm"

      - run: pnpm install --frozen-lockfile

      - name: Typecheck
        run: pnpm --filter @<scope>/api typecheck

      - name: Unit tests
        run: pnpm --filter @<scope>/api test:unit

      - name: Integration tests
        run: pnpm --filter @<scope>/api test:integration
        env:
          DATABASE_URL: postgres://test:test@localhost:5432/test_db
```

### Rules

- Unit tests live as `.test.ts` siblings to the file they test
- Integration tests that need a DB live in `src/test/integration/` or alongside the route as `routes/<name>.test.ts`
- **Never mock the DB in integration tests** — always use a real Postgres
- **Always mock external services** (FjordID JWT verification, email, etc.) in integration tests — no live network calls in CI
- Each integration test file is fully isolated: reset tables in `beforeEach`, not just `beforeAll`
- CI runs unit tests before integration tests — a unit failure fails fast before spinning up Postgres
- `test:unit` and `test:integration` are separate scripts so they can run independently

---

## Production Debugging

> **Note:** For easier debugging of production issues, use passwordless SSH to get logs from the server. Set up SSH key authentication once, then all the commands below run without password prompts.

### Check container status

```bash
export SERVER_HOST="<server-ip>"
export SERVER_USER="<user>"

ssh $SERVER_USER@$SERVER_HOST "cd /opt/<appname> && docker compose -f docker-compose.prod.yml --env-file .env.prod ps"
```

### Tail logs

```bash
# Caddy (TLS errors, upstream failures, HTTP access)
ssh $SERVER_USER@$SERVER_HOST "cd /opt/<appname> && docker compose -f docker-compose.prod.yml --env-file .env.prod logs --tail=100 caddy"

# API
ssh $SERVER_USER@$SERVER_HOST "cd /opt/<appname> && docker compose -f docker-compose.prod.yml --env-file .env.prod logs --tail=100 api"

# DB
ssh $SERVER_USER@$SERVER_HOST "cd /opt/<appname> && docker compose -f docker-compose.prod.yml --env-file .env.prod logs --tail=100 db"
```

Or use the helper scripts from invoicia (copy to each app):

```bash
./infra/scripts/prod-logs.sh caddy
./infra/scripts/prod-logs.sh api
./infra/scripts/prod-logs.sh all
```

Scripts need env vars:

```bash
export INVOICIA_SSH_HOST="<server-ip>"
export INVOICIA_SSH_USER="<user>"
```

### Common issues

| Symptom             | Likely cause                              | Fix                                                                        |
| ------------------- | ----------------------------------------- | -------------------------------------------------------------------------- |
| `502 Bad Gateway`   | API container down or DB not ready        | Check API logs, verify DB healthy                                          |
| `Certificate error` | Traefik label wrong or DNS not propagated | Check Traefik labels in compose, wait for DNS                              |
| Migration fails     | DB not ready when migration runs          | Check `depends_on` healthcheck config                                      |
| GHCR pull fails     | GitHub token expired on server            | Re-login: `echo $TOKEN \| docker login ghcr.io -u <user> --password-stdin` |
| `401 Unauthorized`  | FjordID client ID mismatch                | Check `FJORDID_CLIENT_ID` and `FJORDID_ALLOWED_AZP` env vars               |

---

## Server Maintenance

```bash
# Free up disk space (prune unused images)
ssh $SERVER_USER@$SERVER_HOST "docker image prune -af"

# View all running containers
ssh $SERVER_USER@$SERVER_HOST "docker ps"

# Restart a specific service without full redeploy
ssh $SERVER_USER@$SERVER_HOST "cd /opt/<appname> && docker compose -f docker-compose.prod.yml --env-file .env.prod restart api"

# Manual DB backup
ssh $SERVER_USER@$SERVER_HOST "docker exec \$(docker ps -qf name=<appname>-db) pg_dump -U <dbuser> <dbname> > /tmp/backup.sql"
```

---

## New App Launch Checklist

```
[ ] Domain registered on Gandi.net (e.g. formvault.eu)
[ ] DNS A records added in Gandi LiveDNS (@ and api subdomain → server IP)
[ ] DNS propagation verified (dig +short <app>.eu returns server IP)
[ ] GitHub repo created
[ ] pnpm monorepo structure set up (apps/api, apps/web, packages/shared)
[ ] .github/copilot-instructions.md created (links to shared playbook)
[ ] infra/ files created (Caddyfile.prod, Dockerfiles, docker-compose.prod.yml)
[ ] GitHub Actions workflows added (ci.yml, deploy-production.yml)
[ ] gh auth status confirms GitHub CLI is authenticated
[ ] GitHub Environment "production" created (gh api ... --method PUT)
[ ] SSH_PRIVATE_KEY secret set via gh (piped from ~/.ssh/id_ed25519)
[ ] POSTGRES_PASSWORD secret set via gh
[ ] All variables set via gh variable set && verified with gh variable list
[ ] FjordID clients created (if auth needed)
[ ] FjordID API key created (fjordid.eu/console/api-keys) and added to .vscode/mcp.json
[ ] .vscode/mcp.json added to .gitignore
[ ] vitest installed and vitest.config.ts added to API package
[ ] docker-compose.test.yml added for local integration test DB
[ ] src/test/helpers.ts created (createTestApp, teardown, resetDb)
[ ] Health endpoint integration test passing (GET /api/health → 200)
[ ] Migration idempotency test passing
[ ] Auth middleware unit tests passing (reject expired/malformed tokens)
[ ] Core CRUD route integration tests passing
[ ] CI test job configured in ci.yml with Postgres service
[ ] CI passes (unit + integration) before first deploy
[ ] First deploy triggered (push to main)
[ ] Health check endpoint returns 200 (GET /api/health)
[ ] FjordID login works end-to-end
[ ] Traefik TLS cert issued (check https:// in browser)
```

---

## Google Analytics & Google Search Console

### Google Analytics

1. Go to [Google Analytics](https://analytics.google.com/) and create a new property for the app domain.
2. Add the provided GA4 Measurement ID to your frontend (e.g., via environment variable or directly in the analytics integration).
3. Verify that page views and events are tracked in the Analytics dashboard.

### Google Search Console

1. Go to [Google Search Console](https://search.google.com/search-console/about) and add the app domain as a new property.
2. Verify domain ownership (recommended: DNS TXT record method).
3. Submit the sitemap (e.g., `/sitemap.xml`) if available.
4. Monitor indexing status and resolve any coverage or enhancement issues.

---

## SEO

For projects that need search engine visibility, implement these SEO best practices:

### Meta Tags

Add essential meta tags to your HTML `<head>`:

```html
<title>Page Title - Brand Name</title>
<meta
  name="description"
  content="Concise description (150-160 characters) that appears in search results"
/>
<meta name="keywords" content="relevant, keywords, separated, by, commas" />

<!-- Open Graph (Facebook, LinkedIn) -->
<meta property="og:title" content="Page Title" />
<meta property="og:description" content="Description for social shares" />
<meta property="og:image" content="https://yourdomain.com/og-image.jpg" />
<meta property="og:url" content="https://yourdomain.com/page" />
<meta property="og:type" content="website" />

<!-- Twitter Card -->
<meta name="twitter:card" content="summary_large_image" />
<meta name="twitter:title" content="Page Title" />
<meta name="twitter:description" content="Description for Twitter shares" />
<meta name="twitter:image" content="https://yourdomain.com/twitter-image.jpg" />
```

### Sitemap

Generate and serve a sitemap at `/sitemap.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://yourdomain.com/</loc>
    <lastmod>2026-02-22</lastmod>
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
  </url>
  <url>
    <loc>https://yourdomain.com/page</loc>
    <lastmod>2026-02-22</lastmod>
    <changefreq>monthly</changefreq>
    <priority>0.8</priority>
  </url>
</urlset>
```

Submit the sitemap to Google Search Console after deployment.

### Robots.txt

Serve a `/robots.txt` file to control crawler access:

```
User-agent: *
Allow: /
Sitemap: https://yourdomain.com/sitemap.xml
```

### Performance & Technical SEO

- **Fast load times**: Optimize images, lazy-load assets, minimize JavaScript.
- **Mobile responsive**: Ensure the site works well on all device sizes.
- **HTTPS**: Always use TLS/SSL (handled by Traefik in our setup).
- **Semantic HTML**: Use proper heading hierarchy (h1, h2, h3) and semantic tags.
- **Clean URLs**: Use readable paths like `/blog/post-title` instead of `/blog?id=123`.

### Content Best Practices

- **Unique titles and descriptions** for each page.
- **H1 tag** on every page with the main topic.
- **Alt text** for all images.
- **Internal linking** between related pages.
- **Regular content updates** signal freshness to search engines.

---

## 3rd Party Integrations

### Email - Brevo

For apps requiring email functionality, we use [Brevo](https://www.brevo.com/) (formerly Sendinblue).

**Setup:**

1. Create a Brevo account or use existing account.
2. Generate an API key from **Settings → SMTP & API → API Keys**.
3. Add the API key to environment secrets (e.g., `BREVO_API_KEY`).
4. Use Brevo's REST API or SMTP relay for sending transactional emails.

### Payments - Stripe

For payment processing, we use [Stripe](https://stripe.com/).

**Setup:**

1. Create a Stripe account or use existing account.
2. Get API keys from **Developers → API keys** (use test keys for development, live keys for production).
3. Add keys to environment secrets (e.g., `STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY`).
4. Configure webhook endpoints for payment events (e.g., `checkout.session.completed`).
5. Add webhook signing secret to environment (e.g., `STRIPE_WEBHOOK_SECRET`).
6. Monitor indexing status and resolve any coverage or enhancement issues.
