# Agent Guide: Docker Compose Homelab Repo

## Core Workflow

- **Run a service**: `cd <service> && docker compose up -d`
- **Check status**: `docker compose ps`
- **View logs**: `docker compose logs -f`
- **Stop**: `docker compose down`

## Never Ignore

- Tracked `.env.example` files are templates; create local `.env` from them (never edit tracked `.env` in the repo)
- `caddy/conf/Caddyfile` is the source of truth for service URLs
- Tracked config in `/home-assistant/config/` and `/kavita/config/` (all other `.env` and `config/*` are ignored)

## Adding a Service

1. Copy `0-templates/compose.quickstart.yml` to `<new-service>/compose.yaml`
2. Copy `0-templates/<service>.env.example`; create local `.env` from the template
3. Add proxy config to `caddy/conf/Caddyfile`
4. Add service entry to `caddy/compose.yml`

## Multi-Service Patterns

- **VPN**: `gluetun` container + service with `network_mode: service:gluetun` (qBittorrent, Stremio)
- **Database**: Primary service + `postgres` container via `depends_on: postgres { condition: service_healthy }` (Immich, Mealie, Warracker)

## Caddyfile Syntax

All service blocks use:
- `import homelab_tls` for SSL
- `reverse_proxy 127.0.0.1:<PORT>`

## Common Gotchas

- Don't run `docker compose` outside a service directory (missing service-specific env vars)
- `gluetun` container uses memory limits (~64M reservation, ~256M limit)
- Health checks for database services use `pg_isready`; HTTP services use their API endpoints
