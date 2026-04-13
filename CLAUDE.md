# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository tracks Docker Compose definitions, templates, and hand-edited configuration for a homelab deployment. It contains 12 services:

1. **Caddy** - Reverse proxy with Tailscale forward auth
2. **qBittorrent** - Torrent client with Gluetun VPN
3. **Stremio** - Media server with Gluetun VPN
4. **Immich** - Self-hosted photo backup (multi-service: server, redis, postgres)
5. **Dashy** - Dashboard homepage
6. **Home Assistant** - Home automation
7. **Spotify Bundle** - Slskd + Soulsync + Navidrome (multi-service)
8. **Mealie** - Recipe manager with postgres
9. **Actual Budget** - Budgeting app
10. **Kavita** - E-book/comic reader
11. **File Browser** - File manager
12. **AIO Streams** - Anime streaming aggregator
13. **Arcane** - Anime streaming service
14. **Warracker** - Web asset scanner with postgres

## Repository Structure

```
/opt/docker/
├── 0-templates/          # Template compose files for new services
│   ├── compose.template.yml  # Full template with examples
│   └── compose.quickstart.yml  # Minimal template
├── caddy/                # Caddy reverse proxy config
│   ├── compose.yml       # Caddy container definition
│   └── conf/Caddyfile    # Caddy configuration with all service configs
├── <service>/            # Per-service directories
│   ├── compose.yaml/yml  # Service docker compose file
│   ├── .env              # Service-specific secrets (ignored in git)
│   ├── .env.example      # Template for .env
│   └── <data-dir>/       # Service data (ignored except specific config)
```

## Environment Variables

All services load common environment variables from `/.env.global`:
- `PUID`, `PGID` - User/group IDs for file permissions
- `TZ` - Timezone (Europe/Lisbon)
- `DOMAIN` - Domain name (homelab.home.arpa)
- `CADD_CERT_PATH`, `CADD_CERT_KEY_PATH` - SSL cert paths
- `OPENVPN_USER`, `OPENVPN_PASSWORD`, `SERVER_CITIES` - VPN credentials
- `TS_AUTHKEY` - Tailscale auth key

Service-specific variables are loaded from `<service>/.env`.

## Git Ignore Conventions

The repository follows these ignore rules:
- **Ignored**: All `*.env` files (except `.env.example` templates)
- **Ignored**: All logs, backups, caches, tmp, state, and data directories
- **Ignored**: Database files (`.db`, `.sqlite*`)
- **Ignored**: Media libraries, uploads, downloads
- **Tracked**: Service compose files, config files, `.env.example` templates
- **Exception**: Home Assistant config files and Kavita config are tracked (see `.gitignore`)

## Common Development Tasks

### Running a service
```bash
cd <service>
docker compose up -d
```

### Checking service status
```bash
cd <service>
docker compose ps
```

### View logs
```bash
cd <service>
docker compose logs -f
```

### Stopping a service
```bash
cd <service>
docker compose down
```

### Adding a new service
1. Copy `0-templates/compose.quickstart.yml` to `<new-service>/compose.yaml`
2. Copy `0-templates/compose.template.yml` for reference
3. Create `0-templates/<new-service>.env.example` based on the service's requirements
4. Create `<new-service>/.env` by copying from `.env.example`
5. Add service config block to `caddy/conf/Caddyfile`
6. Add Caddy proxy config to `caddy/compose.yml`

## Caddy Configuration

The Caddyfile uses Caddyfile v2 syntax with import blocks:

```caddy
(homelab_tls) {
    tls /etc/caddy/certs/homelab.home.arpa+1.pem /etc/caddy/certs/homelab.home.arpa+1-key.pem
}

(tailscale_forward_auth) {
    forward_auth unix//run/tailscale.nginx-auth.sock {
        uri /auth
        # ... header mappings ...
    }
}

service.subdomain.homelab.home.arpa {
    import homelab_tls
    import tailscale_forward_auth
    reverse_proxy 127.0.0.1:PORT
}
```

## Multi-Service Patterns

Several services use multi-container patterns:

### VPN Pattern (qBittorrent, Stremio)
```yaml
services:
  gluetun:
    image: qmcgaw/gluetun
    # ... VPN config ...
  
  <service>:
    network_mode: service:gluetun  # Routes traffic through VPN
    depends_on:
      gluetun:
        condition: service_healthy
```

### Database Pattern (Immich, Mealie, Warracker)
```yaml
services:
  <service>:
    depends_on:
      postgres:
        condition: service_healthy
  postgres:
    image: postgres:XX
    volumes:
      - <path>:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready"]
      interval: 30s
      timeout: 20s
      retries: 3
```

## Caddyfile Import Syntax

Caddyfile v2 supports import blocks for reusable snippets. Each service gets its own block with:
- `import homelab_tls` - SSL certificate configuration
- `import tailscale_forward_auth` - Tailscale authentication (optional per service)
- `reverse_proxy 127.0.0.1:<PORT>` - Backend service address

## Template System

The templates in `0-templates/` show:
1. **Basic service** - Simple container with Caddy proxy
2. **VPN service** - Container with Gluetun VPN routing
3. **Multi-service** - Services with dependencies (database, cache, etc.)
4. **No Caddy** - Internal-only services

## Memory Management

Services use `deploy.resources` for memory limits:
```yaml
deploy:
  resources:
    limits:
      memory: 256M  # Gluetun is lightweight
    reservations:
      memory: 64M   # Memory guaranteed
```

## Health Checks

Recommended health checks for critical services:
```yaml
healthcheck:
  test: ["CMD", "pg_isready"]  # postgres
  test: redis-cli ping || exit 1  # redis/valkey
  test: wget -qO- http://localhost:PORT/api/health  # HTTP endpoints
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

## Important Notes

1. **Never modify tracked `.env` files** - They should not be committed to git. Use `.env.example` as template and create local `.env` files.

2. **Service compose files follow the template** - Most services use the pattern from `0-templates/compose.quickstart.yml` with service-specific adjustments.

3. **Caddyfile is the source of truth for service URLs** - When adding/editing services, update the Caddyfile first, then ensure the compose file ports match.

4. **Tailscale is used for access control** - Services use Tailscale forward auth for authentication. Disable with `import tailscale_forward_auth` removed from Caddyfile block.

5. **VPN is used for PIA/NordVPN routing** - Services like qBittorrent and Stremio use Gluetun to route through VPN providers.

6. **Docker Compose v2+** - Services use modern compose syntax with v3.8+ specs.
