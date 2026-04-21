# Docker Compose Backup

This repository tracks compose definitions, templates, and hand-edited configuration.

It intentionally does not track:
- secrets in `.env` files
- databases and runtime state
- logs, caches, and backups
- large media libraries and downloads

## Conventions

- Keep live secrets in ignored `.env` files and update the matching `.env.example` files when variables change.
- Run `docker compose` from each service directory after copying `.env.example` to `.env` where needed.
- VPN-routed apps can share one tunnel in `gluetun/`; start `gluetun` first, then dependent apps using `network_mode: container:app-gluetun`.
- Use a separate backup tool such as Restic or Borg for the small state you want to preserve outside git.
