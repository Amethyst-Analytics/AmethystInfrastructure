# AmethystInfrastructure — Agent Rules

> Repo-specific rules for Claude Code and Gemini CLI. Read `../CLAUDE.md` and `../GEMINI.md` first — this file extends those, it does not replace them.

---

## Purpose

AmethystInfrastructure manages **deployment** of the Amethyst stack onto the Raspberry Pi. It contains Docker Compose files, setup scripts, config templates, and the client generation script. It has no Python application code.

---

## Stack

| Item               | Detail                                                   |
| ------------------ | -------------------------------------------------------- |
| Language           | Bash (shell scripts)                                     |
| Orchestration      | Docker Compose v2                                        |
| Target runtime     | Raspberry Pi 4, ARM64, Raspberry Pi OS 64-bit (Bookworm) |
| Container registry | GHCR (`ghcr.io/amethyst-analytics/`)                     |
| Config format      | `.env` files (never committed with real secrets)         |

---

## File Structure

```
AmethystInfrastructure/
├── setup.sh                            # Entry point: clone repos, configure, launch
├── scripts/
│   ├── install.sh                      # Install Docker, Poetry, tools on Pi
│   ├── configure.sh                    # Generate .env from .env.example + user input
│   ├── deploy.sh                       # docker compose up with health verification
│   ├── update.sh                       # Pull latest images, migrate, restart
│   ├── teardown.sh                     # docker compose down, optionally remove volumes
│   ├── backup.sh                       # pg_dump to dated file on SSD
│   ├── health_check.sh                 # Verify all services healthy post-deploy
│   ├── generate_secrets.sh             # Generate random API keys and JWT secret
│   └── generate_client.sh             # Regenerate amethyst_client from openapi.json
├── docker-compose/
│   ├── docker-compose.yml              # Production (used on Pi)
│   ├── docker-compose.dev.yml          # Dev overrides (volume mounts, exposed ports)
│   └── docker-compose.test.yml        # Test environment
├── config/
│   ├── .env.example                    # All environment variables documented (no real values)
│   ├── postgres/postgres.conf          # Pi-tuned PostgreSQL 16 configuration
│   └── redis/redis.conf               # Redis 7.2 configuration
└── README.md
```

---

## Critical Rules

- **SSD-first**: All Docker volumes for PostgreSQL data, Redis AOF/RDB, and application data MUST bind-mount to `/mnt/ssd/`. Never bind to the SD card path (`/home/`, `/var/`, etc.).
- **No builds on Pi**: Docker Compose files must use `image:` only — never `build:`. Images come from GHCR.
- **Multi-arch images**: CI builds both `linux/amd64` and `linux/arm64`. The Pi pulls the `linux/arm64` variant automatically.
- **Secrets never committed**: `.env` files with real credentials are never committed. `.env.example` documents all variables with placeholder values.
- **PostgreSQL version**: 16 (not 15). Update `postgres.conf` accordingly.

---

## Docker Compose Rules

- Memory limits and reservations must match the values in `codeplan.md` §4.
- CPU limits must match the values in `codeplan.md` §4.
- All services must have a `healthcheck` defined.
- Services must start in dependency order: `timescaledb` → `redis` → `market_monitor` → `amethyst_server` → `amethyst_analytics`.
- `restart: unless-stopped` on all production services.

---

## Shell Script Rules

- All scripts: `#!/bin/bash` + `set -euo pipefail`
- Destructive operations (`teardown.sh`, volume deletion) require explicit `--yes` flag or interactive `y/N` prompt
- Scripts must print a human-readable status line for each major step
- Long-running operations must show progress
- Exit codes: `0` = success, `1` = general error, `2` = usage error

---

## Naming Conventions

- Script files: `kebab-case.sh`
- Environment variables in `.env.example`: `SCREAMING_SNAKE_CASE`
- Docker service names in Compose: `snake_case`

---

## Forbidden Patterns

- No Python code (this repo is Bash and YAML/Docker only)
- No hardcoded IP addresses — use Docker service names for inter-container networking
- No `docker build` in any script run on the Pi
- No bind mounts to SD card paths
- No secrets in `.env.example` — placeholder values only (e.g., `POSTGRES_PASSWORD=changeme`)

---

## Service Boundaries

AmethystInfrastructure has no runtime. It is the deployment layer only. It orchestrates the other services via Docker Compose but does not contain application logic.
