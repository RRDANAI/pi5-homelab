# Pi5 Homelab

n8n workflow automation on Raspberry Pi 5 with secure Tailscale access and public webhook endpoints.

## Hardware

- Raspberry Pi 5 (8GB RAM)
- Pironman 5 MAX case
- WD Black SN750SE 1TB NVMe

## Architecture

```
Internet
    │
    ├─► nerdrack.maroonoven.work (Public - Webhooks only)
    │
    └─► racknerd-b406ebf.tailf84e24.ts.net (Private - Full UI)
                    │
                    └─► Caddy ─► n8n ─┬─► PostgreSQL
                                      └─► Redis
```

## Services

| Service | Purpose |
|---------|---------|
| n8n | Workflow automation UI and triggers |
| n8n-worker | Queue-based workflow execution |
| PostgreSQL 16 | Database storage |
| Redis | Job queue |

Caddy and Tailscale run on the host (not in Docker).

## Setup

1. Clone the repo and create `.env` from the example:

```bash
cp .env.example .env
```

2. Generate an encryption key and fill in `.env`:

```bash
openssl rand -hex 32
```

3. Copy config files to system locations:

```bash
sudo cp config/caddy/Caddyfile /etc/caddy/Caddyfile
sudo cp config/tailscale/tailscaled /etc/default/tailscaled
```

4. Start services:

```bash
docker compose up -d
```

## Access

- **Private UI**: `https://racknerd-b406ebf.tailf84e24.ts.net` (requires Tailscale)
- **Public webhooks**: `https://nerdrack.maroonoven.work/webhook/*`

## File Structure

```
.
├── config/
│   ├── caddy/Caddyfile        # Reverse proxy config
│   └── tailscale/tailscaled   # Tailscale daemon config
├── docker-compose.yml          # n8n stack
├── init-data.sh               # PostgreSQL user setup
├── .env.example               # Environment template
└── README.md
```

## Common Commands

```bash
docker compose logs -f n8n      # View n8n logs
docker compose restart n8n      # Restart n8n
docker compose down             # Stop everything
docker compose pull && docker compose up -d  # Update
```
