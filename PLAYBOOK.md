# Production Deployment Playbook

Personal runbook for deploying apps on the shared Hetzner server.
All apps follow the same pattern: **pnpm monorepo → GHCR Docker images → SSH deploy → Docker Compose → Traefik (TLS) + Caddy (SPA+API proxy) + Postgres**.

## Live Apps

| App       | Domain       | Repo                                                        | Server path      |
| --------- | ------------ | ----------------------------------------------------------- | ---------------- |
| invoicia  | invoicia.eu  | [terjetyl/invoicia](https://github.com/terjetyl/invoicia)   | `/opt/invoicia`  |
| formvault | formvault.eu | [terjetyl/formvault](https://github.com/terjetyl/formvault) | `/opt/formvault` |

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

## Checklist: Deploying a New App

### 1. DNS

Add DNS A records:

```
<app>.eu       → <server-ip>
api.<app>.eu   → <server-ip>   # if using a separate API domain
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

### 3. GitHub Environment

In the GitHub repo → Settings → Environments → New environment → name it `production`.

**Variables** (adapt per app):

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

**Secrets**:

| Secret              | Description                       |
| ------------------- | --------------------------------- |
| `SSH_PRIVATE_KEY`   | Private key for server SSH access |
| `POSTGRES_PASSWORD` | Database password                 |

> Skip FjordID variables if the app has no auth.

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
.github/
  workflows/
    ci.yml               # Test + typecheck on every PR/push
    deploy-production.yml # Build GHCR images + SSH deploy
```

### 5. docker-compose.prod.yml Pattern

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

### 6. Caddyfile.prod Pattern

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

### 7. API Dockerfile Pattern

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

### 8. Deploy Workflow Pattern

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

---

## Production Debugging

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
[ ] Domain registered (e.g. formvault.eu)
[ ] DNS A record pointing to server IP
[ ] GitHub repo created
[ ] pnpm monorepo structure set up (apps/api, apps/web, packages/shared)
[ ] infra/ files created (Caddyfile.prod, Dockerfiles, docker-compose.prod.yml)
[ ] GitHub Actions workflows added (ci.yml, deploy-production.yml)
[ ] GitHub Environment "production" created with all vars + secrets
[ ] FjordID clients created (if auth needed)
[ ] SSH_PRIVATE_KEY secret added (reuse same key as invoicia)
[ ] POSTGRES_PASSWORD secret set
[ ] First deploy triggered (push to main)
[ ] Health check endpoint returns 200 (GET /api/health)
[ ] FjordID login works end-to-end
[ ] Traefik TLS cert issued (check https:// in browser)
```
