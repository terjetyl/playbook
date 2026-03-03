# Skill: Production Server Access

## Infrastructure overview

| Thing         | Value                                                     |
| ------------- | --------------------------------------------------------- |
| Provider      | Hetzner Cloud (Germany)                                   |
| Server IP     | `46.225.16.76`                                            |
| SSH user      | `deploy`                                                  |
| App directory | `/opt/<appname>/` (e.g. `/opt/formvault/`)                |
| Compose file  | `docker-compose.prod.yml`                                 |
| Env file      | `.env.prod` (never committed, written by deploy workflow) |

## Passwordless SSH setup

Do this once per machine. The deploy key already exists on your dev machine (the same key used by GitHub Actions).

```bash
# Check which key is configured for the server
ssh -G 46.225.16.76 | grep identityfile

# Test passwordless access
ssh deploy@46.225.16.76 echo ok

# If prompting for password, add the key to ssh-agent
ssh-add ~/.ssh/id_ed25519

# Or add a permanent entry to ~/.ssh/config:
Host hetzner
  HostName 46.225.16.76
  User deploy
  IdentityFile ~/.ssh/id_ed25519
# Then: ssh hetzner
```

## Navigating the server

```bash
# Connect
ssh deploy@46.225.16.76
# or: ssh hetzner (after config above)

# List running apps
ls /opt/

# Go to an app
cd /opt/formvault

# See what's running
docker compose -f docker-compose.prod.yml --env-file .env.prod ps
```

## Viewing logs

```bash
APP=formvault
SSH="ssh deploy@46.225.16.76"

# API logs (last 100 lines)
$SSH "cd /opt/$APP && docker compose -f docker-compose.prod.yml --env-file .env.prod logs --tail=100 api"

# Follow API logs in real time
$SSH "cd /opt/$APP && docker compose -f docker-compose.prod.yml --env-file .env.prod logs -f api"

# Caddy (HTTP access log, TLS errors, upstream failures)
$SSH "cd /opt/$APP && docker compose -f docker-compose.prod.yml --env-file .env.prod logs --tail=100 caddy"

# DB logs
$SSH "cd /opt/$APP && docker compose -f docker-compose.prod.yml --env-file .env.prod logs --tail=50 db"

# All services at once
$SSH "cd /opt/$APP && docker compose -f docker-compose.prod.yml --env-file .env.prod logs --tail=50"
```

## Exec into a running container

```bash
# Shell in the API container
$SSH "cd /opt/$APP && docker compose -f docker-compose.prod.yml --env-file .env.prod exec api sh"

# Run a one-off command in the API container (non-interactive)
$SSH "cd /opt/$APP && docker compose -f docker-compose.prod.yml --env-file .env.prod exec -T api node -e 'console.log(process.env.NODE_ENV)'"

# Postgres interactive shell
$SSH "cd /opt/$APP && docker compose -f docker-compose.prod.yml --env-file .env.prod exec db psql -U formvault formvault_db"
```

## Useful one-liners

```bash
# Check current image digests (verify a deploy landed)
$SSH "cd /opt/$APP && docker compose -f docker-compose.prod.yml --env-file .env.prod images"

# Restart just the API without full redeploy
$SSH "cd /opt/$APP && docker compose -f docker-compose.prod.yml --env-file .env.prod restart api"

# Tail only error lines from API
$SSH "cd /opt/$APP && docker compose -f docker-compose.prod.yml --env-file .env.prod logs -f api" | grep -i error

# Free up disk (prune old images)
$SSH "docker image prune -af"

# Show disk usage
$SSH "df -h"

# Manual DB backup
$SSH "docker exec \$(docker ps -qf name=formvault-db) pg_dump -U formvault formvault_db > /tmp/backup-\$(date +%Y%m%d).sql"

# Copy backup to local machine
scp deploy@46.225.16.76:/tmp/backup-*.sql ./
```

## Running migrations manually (emergency only)

Migrations normally run automatically via the deploy workflow. Only do this if a deploy failed mid-migration and you need to recover:

```bash
$SSH "cd /opt/$APP && docker compose -f docker-compose.prod.yml --env-file .env.prod run --rm api node dist/migrate.js"
```

## Checking env vars on the server

```bash
# Print .env.prod (confirm secrets are set correctly after a deploy)
$SSH "cat /opt/$APP/.env.prod"

# Check a specific var inside the running container
$SSH "cd /opt/$APP && docker compose -f docker-compose.prod.yml --env-file .env.prod exec -T api printenv ENCRYPTION_KEY"
```

## Common issues

| Symptom                    | Check                                                                             |
| -------------------------- | --------------------------------------------------------------------------------- |
| `502 Bad Gateway`          | API container crashed — `logs api`                                                |
| TLS cert missing           | Traefik labels wrong, or DNS not pointing to server — `docker logs traefik`       |
| Migration failed on deploy | `logs api` for the migrate step; DB may not have been ready                       |
| GHCR image pull fails      | Re-login: `echo $TOKEN \| docker login ghcr.io -u terjetyl --password-stdin`      |
| `401` on all API calls     | `FJORDID_CLIENT_ID` or `FJORDID_ALLOWED_AZP` env var mismatch — print `.env.prod` |
| Container keeps restarting | `docker inspect <container-id>` → check `State.Error`                             |

## Traefik (shared across all apps)

```bash
# Traefik runs in its own stack, not inside app stacks
$SSH "docker logs traefik --tail=50"

# List all Traefik-registered routers
$SSH "docker inspect traefik | grep -A5 rule"
```
