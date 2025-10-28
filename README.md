# HomeLab-
portainer-wsl/
├─ README.md
├─ docker-compose.yml
├─ .env.example
├─ scripts/
│  ├─ install.sh
│  ├─ update.sh
│  └─ uninstall.sh
├─ wsl-notes.md
├─ SECURITY.md
├─ LICENSE
└─ .gitignore
version: "3.9"

services:
  portainer:
    image: portainer/portainer-ce:lts
    container_name: ${PORTAINER_NAME:-portainer}
    restart: unless-stopped
    ports:
      # 9443 = HTTPS UI; 8000 = Edge/Agent (optional but exposed by default)
      - "${PORTAINER_HTTPS_PORT:-9443}:9443"
      - "${PORTAINER_EDGE_PORT:-8000}:8000"
    volumes:
      # Required to manage the local Docker engine
      - /var/run/docker.sock:/var/run/docker.sock
      # Persistent Portainer data (users, stacks, endpoints, etc.)
      - portainer_data:/data

volumes:
  portainer_data:
    name: ${PORTAINER_DATA_VOLUME:-portainer_data}
# ===== Portainer WSL defaults =====
# Container name visible in `docker ps`
PORTAINER_NAME=portainer

# Ports exposed on the WSL host (change if conflicts)
PORTAINER_HTTPS_PORT=9443
PORTAINER_EDGE_PORT=8000

# Named volume for persistent data
PORTAINER_DATA_VOLUME=portainer_data
#!/usr/bin/env bash
set -euo pipefail

# --- helpers ---
log() { printf "\033[1;32m[+] %s\033[0m\n" "$*"; }
warn() { printf "\033[1;33m[!] %s\033[0m\n" "$*"; }
err() { printf "\033[1;31m[x] %s\033[0m\n" "$*"; }

require() {
  command -v "$1" >/dev/null 2>&1 || { err "Missing $1. Install it and retry."; exit 1; }
}

# --- checks ---
require docker
if ! docker info >/dev/null 2>&1; then
  err "Docker daemon not reachable. If using Docker Desktop, enable WSL integration. Otherwise start dockerd."
  exit 1
fi

# Warn about /mnt/c bind-mount performance if user intends to clone here
case "$PWD" in
  /mnt/*) warn "Repo is under /mnt/. For best performance, clone under your Linux home (e.g., /home/<you>/portainer-wsl)";;
esac

# env
if [ ! -f ".env" ]; then
  log "Creating .env from template"
  cp -n .env.example .env
fi

# Validate compose
log "Validating compose file"
docker compose config >/dev/null

# Pull & start
log "Pulling images"
docker compose pull

log "Starting Portainer"
docker compose up -d

# Show status
log "Containers:"
docker compose ps

# Basic reachability hint
HTTPS_PORT=$(grep -E '^PORTAINER_HTTPS_PORT=' .env | cut -d= -f2 || echo 9443)
log "Open Portainer at: https://localhost:${HTTPS_PORT}  (or https://<WSL-host-IP>:${HTTPS_PORT})"
#!/usr/bin/env bash
set -euo pipefail

# --- helpers ---
log() { printf "\033[1;32m[+] %s\033[0m\n" "$*"; }
warn() { printf "\033[1;33m[!] %s\033[0m\n" "$*"; }
err() { printf "\033[1;31m[x] %s\033[0m\n" "$*"; }

require() {
  command -v "$1" >/dev/null 2>&1 || { err "Missing $1. Install it and retry."; exit 1; }
}

# --- checks ---
require docker
if ! docker info >/dev/null 2>&1; then
  err "Docker daemon not reachable. If using Docker Desktop, enable WSL integration. Otherwise start dockerd."
  exit 1
fi

# Warn about /mnt/c bind-mount performance if user intends to clone here
case "$PWD" in
  /mnt/*) warn "Repo is under /mnt/. For best performance, clone under your Linux home (e.g., /home/<you>/portainer-wsl)";;
esac

# env
if [ ! -f ".env" ]; then
  log "Creating .env from template"
  cp -n .env.example .env
fi

# Validate compose
log "Validating compose file"
docker compose config >/dev/null

# Pull & start
log "Pulling images"
docker compose pull

log "Starting Portainer"
docker compose up -d

# Show status
log "Containers:"
docker compose ps

# Basic reachability hint
HTTPS_PORT=$(grep -E '^PORTAINER_HTTPS_PORT=' .env | cut -d= -f2 || echo 9443)
log "Open Portainer at: https://localhost:${HTTPS_PORT}  (or https://<WSL-host-IP>:${HTTPS_PORT})"
#!/usr/bin/env bash
set -euo pipefail
log() { printf "\033[1;32m[+] %s\033[0m\n" "$*"; }

log "Pulling latest images"
docker compose pull

log "Recreating containers (no data loss)"
docker compose up -d

log "Done. Current status:"
docker compose ps
#!/usr/bin/env bash
set -euo pipefail
log() { printf "\033[1;33m[!] %s\033[0m\n" "$*"; }

log "Stopping and removing stack"
docker compose down

log "NOTE: Persistent data volume was NOT removed."
log "If you want a full reset, run:  docker volume rm $(docker volume ls -q | grep -E '^portainer_data$' || true)"
# Portainer on WSL (Docker Compose)

One-command install of **Portainer Community Edition** on **WSL2** with Docker.
Secure by default (HTTPS on 9443), persistent data, and copy-paste scripts.

## Quickstart

```bash
git clone https://github.com/<you>/portainer-wsl.git
cd portainer-wsl
cp .env.example .env     # optional: edit ports/names
./scripts/install.sh
Open: https://localhost:9443

Create the admin user, select Local environment, and you’re in.

If 9443 is in use, set a new port in .env (e.g., PORTAINER_HTTPS_PORT=9444) and re-run ./scripts/install.sh.

Files

docker-compose.yml — minimal, secure defaults (9443 HTTPS only)

.env.example — change container name, host ports, volume name

scripts/install.sh — validates, pulls, starts

scripts/update.sh — pulls updates and recreates

scripts/uninstall.sh — stops/removes stack (keeps data volume)

wsl-notes.md — WSL networking & performance tips

SECURITY.md — basic hardening guidance

Common tasks

Update Portainer

./scripts/update.sh


Uninstall (keep data)

./scripts/uninstall.sh


Uninstall AND delete data (irreversible)

docker compose down
docker volume rm portainer_data

Troubleshooting

Docker not reachable in WSL

If using Docker Desktop: Settings → Resources → WSL integration → enable for your distro.

If using native Docker Engine: make sure dockerd is running.

Can’t open https://localhost:9443

Port collision? Edit .env → PORTAINER_HTTPS_PORT=9444 → ./scripts/install.sh

Check status: docker compose ps

Logs: docker logs portainer

Slow performance with bind mounts

Avoid placing the repo under /mnt/c/.... Use a Linux path like /home/<you>/portainer-wsl.

Security

Keep the UI private to your LAN or behind a VPN/reverse proxy.

Use a strong admin password; enable 2FA inside Portainer settings.

Expose only the ports you need. This compose maps 9443 and 8000 by default.


## 7) `wsl-notes.md`
```markdown
# WSL notes

## Docker options
- **Docker Desktop (recommended)**: Turn on WSL integration for your distro. No extra daemon work.
- **Native Docker Engine**: Possible, but you must run `dockerd` manually or set up systemd in WSL.

## Paths & performance
- Prefer Linux home (e.g., `/home/you/portainer-wsl`) instead of `/mnt/c/...`.
- If you must use Windows files, expect slower I/O with bind mounts.

## Networking
- `https://localhost:9443` should work from Windows.
- To access from another device on your LAN, find your Windows host IP (e.g., `ipconfig` in Windows PowerShell), then browse to `https://<HOST_IP>:9443`.
- Firewalls might block incoming connections—allow the chosen port.

## Systemd (optional)
On recent WSL, you can enable systemd:


/etc/wsl.conf
[boot]
systemd=true

Then in Windows PowerShell:


wsl --shutdown
wsl

This can help if you run native services in WSL.

8) SECURITY.md
# Security guide

- **Use HTTPS**: This stack publishes Portainer on `9443` (TLS). Keep it.
- **Restrict exposure**: Prefer LAN/VPN access. If you must expose to the Internet, place it behind a reverse proxy with auth and IP allowlists.
- **Admin hygiene**: Strong admin password, rotate periodically, enable 2FA.
- **Least privilege**: Only the volumes and ports in `docker-compose.yml` are required.
- **Updates**: Run `./scripts/update.sh` regularly to keep images patched.

9) LICENSE (MIT)
MIT License

Copyright (c) 2025 …

Permission is hereby granted, free of charge, to any person obtaining a copy…
[standard MIT text]

10) .gitignore
.env
*.log
.DS_Store
Thumbs.db

🧪 WSL-specific validation (do these after cloning)
# In WSL
cd ~/portainer-wsl
cp .env.example .env
docker compose config        # should print the fully-rendered, valid compose
./scripts/install.sh         # pulls and starts
docker ps | grep portainer   # confirm it's running

# Open in browser (Windows side):
https://localhost:9443


If you want me to init a repo with these files and tweak the branding/readme badges for GitHub, tell me the repo name and I’ll output a ready-to-git init bundle too.
