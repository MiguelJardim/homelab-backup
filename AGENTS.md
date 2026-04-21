# AGENTS.md

## Repository Purpose

Docker Compose definitions and configuration for a homelab with 9 services. Config-only repo: no code, no builds.

## Quick Commands

```bash
cd <service> && docker compose up -d    # Start service
cd <service> && docker compose logs -f   # View logs
cd <service> && docker compose down      # Stop service
```

## Structure

```
/opt/docker/
├── 0-templates/          # compose.quickstart.yml for new services
├── caddy/conf/Caddyfile  # Source of truth for all service URLs/ports
└── <service>/            # Each service: compose.yaml + .env + .env.example
```

## Environment Variables

- **Global**: `/.env.global` — shared across all services (PUID, PGID, TZ, DOMAIN, TS_AUTHKEY)
- **VPN shared**: `/.env.vpn` — shared VPN credentials/settings for VPN-routed services
- **Per-service**: `<service>/.env` — service-specific secrets (never commit, use `.env.example` as template)

## Key Patterns

### Adding a Service
1. Copy `0-templates/compose.quickstart.yml` to `<service>/compose.yaml`
2. Create `<service>/.env` from `.env.example`
3. Add block to `caddy/conf/Caddyfile` with `reverse_proxy 127.0.0.1:<PORT>`
4. Ensure Caddyfile port matches compose file

### VPN Pattern (qBittorrent, Stremio)
```yaml
services:
  gluetun:
    image: qmcgaw/gluetun
  <service>:
    network_mode: service:gluetun  # Routes traffic through VPN
    depends_on:
      gluetun:
        condition: service_healthy
```

### Database Pattern (Immich)
- Services depend on postgres with `condition: service_healthy`
- Healthcheck: `["CMD", "pg_isready"]`

## Git Conventions

- **Tracked**: compose files, `.env.example` templates, Caddyfile, selected service configs
- **Ignored**: all `*.env`, logs, caches, databases (`*.db`, `*.sqlite*`), media libraries, downloads

## Important Notes

- Caddyfile uses v2 import syntax: `(homelab_tls)` and `(tailscale_forward_auth)` blocks
- Tailscale forward auth provides authentication — remove import to disable per-service
- Network `caddy-network` is external for Caddy proxy communication
- Domain: `homelab.home.arpa` (local), timezone: `Europe/Lisbon`
